+++
title = "Coalescing y Splitting - Defragmentando en el sitio"
author = ["pablo"]
date = 2026-05-24
tags = ["memory", "c"]
draft = true
translationKey = "post-04-coalescing-and-splitting"
series = ["Memory Allocation y GC desde cero"]
subtitle = "Cómo convertir fragmentos dispersos en bloques útiles"
seriesPost = 1
+++

_Post 4 de 13 — Serie: Memory Allocation y Garbage Collection desde cero_

En el [Post 1]({{< relref "index" >}}) construimos un bump allocator que solo avanza. En el Post 2 le dimos block headers para que cada bloque sepa cuánto mide y si está en uso. En el Post 3 añadimos `heap_free()` y una free list: por primera vez, el allocador puede reclamar bloques y reutilizarlos con first-fit. Funciona. Pero tiene un problema que dejamos deliberadamente abierto, visible en el Ejemplo 2 del Post 3:

El allocador fragmenta la memoria y no puede hacer nada al respecto.

Si allocas tres bloques de 100 bytes, liberas el del medio, y luego pides 200 bytes, la petición falla. Hay 100 bytes libres en el hueco del medio, pero no son suficientes. Y aunque el bloque siguiente también esté libre, el allocador no lo sabe — trata cada bloque como una isla independiente. Los 200 bytes que necesitas podrían estar ahí, contiguos, esperando a ser fusionados. Pero nadie los fusiona.

Hoy arreglamos eso. Implementamos **coalescing** (fusionar bloques libres adyacentes cuando se llama a `free()`) y **splitting** (dividir bloques demasiado grandes cuando se llama a `alloc()`). Son dos caras de la misma moneda: coalescing crea bloques grandes a partir de fragmentos; splitting crea fragmentos controlados a partir de bloques grandes. Juntos, transforman un allocador que degrada con el tiempo en uno que recicla memoria de verdad.

Para hacer coalescing eficiente, necesitamos una pieza nueva de infraestructura: **boundary tags** (footers). Un pequeño espejo del header al final de cada bloque que permite caminar hacia atrás en O(1), sin recorrer toda la free list. El código pasa de ~300 a ~500 líneas. La complejidad añadida es significativa pero contenida. Lo que ganamos — un allocador que se desfragmenta a sí mismo — lo justifica con creces.


## Sección 1: La Tragedia de la Fragmentación {#sección-1-la-tragedia-de-la-fragmentación}


### El problema concreto {#el-problema-concreto}

Revisemos el allocador del Post 3 con una secuencia específica:

```c
a = heap_alloc(h, 100)
b = heap_alloc(h, 100)
c = heap_alloc(h, 100)
heap_free(h, b)           // b queda libre, entre a y c
heap_free(h, c)           // c queda libre, adyacente a b
```

Después de estas operaciones, el heap se ve así:

<pre style="width: fit-content; margin: 0 auto;">
  ┌────────┐┌────────┐┌────────┐
  │ A      ││ B      ││ C      │
  │ IN_USE ││ FREE   ││ FREE   │
  │ 100B   ││ 100B   ││ 100B   │
  └────────┘└────────┘└────────┘

  Free list: C -> B -> NULL
</pre>

Hay 200 bytes libres. Pero si alguien pide `heap_alloc(h, 200)`, el first-fit recorre la free list, encuentra C (100 bytes — no cabe), encuentra B (100 bytes — no cabe), y declara derrota. Pide más memoria al OS con `sbrk()`, desperdiciando los 200 bytes que ya tenía.

Esto es **fragmentación externa**: memoria libre total suficiente, pero dispersa en bloques que individualmente son demasiado pequeños. Es el enemigo número uno de todo allocador, y el motivo por el que `malloc()` de producción gasta tanta ingeniería en combatirlo.


### La solución intuitiva {#la-solución-intuitiva}

La observación clave es que B y C son **adyacentes en memoria**. No hay nada entre ellos — el final de B es literalmente el principio de C. Si borramos la frontera entre ambos, obtenemos un bloque de 200+ bytes (más el espacio que ocupaba el header de C, que ya no necesitamos):

<pre style="width: fit-content; margin: 0 auto;">
  ┌────────┐┌─────────────────────────┐
  │ A      ││ B+C                     │
  │ IN_USE ││ FREE                    │
  │ 100B   ││ 200+ bytes (fusionados) │
  └────────┘└─────────────────────────┘

  Free list: B+C -> NULL
</pre>

Ahora `heap_alloc(h, 200)` encuentra el bloque fusionado y lo sirve directamente. Esto es **coalescing**: fusionar bloques libres adyacentes en un bloque más grande.

La pregunta es _cuándo_ y _cómo_ hacerlo.


### Cuándo coalescer {#cuándo-coalescer}

Hay dos estrategias:

**Immediate coalescing:** fusionar en el momento del `free()`. Cada vez que liberas un bloque, verificas si sus vecinos están libres y los fusionas inmediatamente. Es lo que implementaremos hoy — es la estrategia más simple y la que usa la mayoría de allocadores educativos.

**Deferred coalescing:** no fusionar en el `free()`, sino esperar hasta que `alloc()` no encuentre un bloque adecuado, y entonces recorrer el heap buscando bloques adyacentes que fusionar. `jemalloc` y `tcmalloc` usan variantes de esta estrategia porque el coalescing inmediato puede destruir patrones de uso que benefician al cache. Pero para nuestro allocador, la complejidad no compensa.

La elección tiene consecuencias de rendimiento que veremos en la Sección 9, pero por ahora: coalescing inmediato en `free()`.


## Sección 2: El Problema del Caminante Hacia Atrás {#sección-2-el-problema-del-caminante-hacia-atrás}


### Coalescing hacia adelante es fácil {#coalescing-hacia-adelante-es-fácil}

Para verificar si el bloque _siguiente_ está libre, solo necesitamos aritmética de punteros. Conocemos el header del bloque actual y su tamaño, así que el siguiente header está en:

```c
block_header_t *next = (block_header_t *)((char *)hdr + block_total(hdr));
```

Verificamos que el puntero esté dentro del heap, que tenga magic válido, y que no esté en uso. Si se cumplen las tres condiciones, fusionamos. Trivial.


### Coalescing hacia atrás es el problema {#coalescing-hacia-atrás-es-el-problema}

Para verificar si el bloque _anterior_ está libre, necesitamos saber dónde empieza. Y ahí está el dilema: desde un header, no hay manera directa de calcular dónde está el header anterior. Los bloques tienen tamaños variables — no puedes simplemente restar una constante.

Tenemos dos opciones:

**Opción 1: Recorrer desde el inicio del heap.** Empezar en `h->start` y caminar bloque a bloque con `offset +` HEADER_SIZE + hdr-&gt;size= hasta llegar al bloque justo antes del nuestro. Funciona, pero es O(n) donde n es el número de bloques. En un heap con miles de bloques, esto es inaceptable — convierte cada `free()` en una operación lineal.

**Opción 2: Boundary tags.** Almacenar información al _final_ de cada bloque que nos permita calcular dónde empieza el bloque anterior en O(1). Esto es lo que implementaremos.


### Boundary tags: la idea de Knuth {#boundary-tags-la-idea-de-knuth}

La técnica fue descrita por Donald Knuth en _The Art of Computer Programming_ (1973). La idea es simple: al final de cada bloque, colocamos un **footer** — una pequeña struct que duplica el tamaño del header. Con el footer del bloque anterior, podemos calcular dónde empieza su header.

El razonamiento es aritmético. Si estamos parados en el header de un bloque y queremos encontrar el bloque anterior:

1.  Retrocedemos `FOOTER_SIZE` bytes. Eso nos pone en el footer del bloque anterior.
2.  Leemos `footer->size` — el tamaño del payload del bloque anterior.
3.  Retrocedemos `HEADER_SIZE + footer->size` bytes más. Eso nos pone en el header del bloque anterior.

Todo en O(1). Sin recorrer listas, sin buscar desde el inicio. Aritmética pura.

<pre style="width: fit-content; margin: 0 auto;">
  Bloque anterior              Bloque actual
  ┌────────┬─────────┬──────┐┌────────┬─────────┬──────┐
  │ HEADER │ payload │FOOTER││ HEADER │ payload │FOOTER│
  │ size=X │  X bytes│size=X││ size=Y │  Y bytes│size=Y│
  └────────┴─────────┴──────┘└────────┴─────────┴──────┘
  ^                          ^
  │                          hdr (aqui estamos)
  │
  Para llegar aqui:
    1. (char*)hdr - FOOTER_SIZE  → footer del anterior
    2. leer footer->size = X
    3. retroceder HEADER_SIZE + X → header del anterior
</pre>

El costo es claro: cada bloque ahora tiene un footer además del header. El "impuesto de la metadata" del Post 2 acaba de subir. En nuestro caso, el footer es un `size_t` de 8 bytes, alineado a 16 bytes por nuestro esquema de alineación. Cada bloque paga 16 bytes de header + 16 bytes de footer = 32 bytes de metadata, frente a los 16 bytes del Post 3.

Es un precio que vale la pena pagar. La alternativa — O(n) por cada `free()` — es peor.


## Sección 3: La Struct del Footer y el Nuevo Layout {#sección-3-la-struct-del-footer-y-el-nuevo-layout}


### block_footer_t {#block-footer-t}

```c
typedef struct {
    size_t size;    /* espejo de header->size */
} block_footer_t;
```

Un solo campo: el tamaño del payload. Nada más. Algunas implementaciones también almacenan flags o un magic number en el footer, pero para nuestro propósito es redundante — el header ya tiene esa información, y cuando caminamos hacia atrás, lo primero que hacemos después de encontrar el header anterior es verificar su magic.


### El nuevo layout de un bloque {#el-nuevo-layout-de-un-bloque}

Cada bloque ahora tiene tres partes:

<pre style="width: fit-content; margin: 0 auto;">
  ┌────────────────────┬────────────────────┬────────────────┐
  │   block_header_t   │     payload        │ block_footer_t │
  │   (HEADER_SIZE)    │   (aligned_size)   │  (FOOTER_SIZE) │
  ├────────────────────┼────────────────────┼────────────────┤
  │ size, flags, magic │ datos del usuario  │ size (espejo)  │
  └────────────────────┴────────────────────┴────────────────┘
  │                                                          │
  ├──────────── block_total = H + payload + F ───────────────┤
  </pre>

Y en nuestro caso concreto, con `ALIGNMENT = 16`:

```c
sizeof(block_header_t) = 16   →  HEADER_SIZE = 16
sizeof(block_footer_t) =  8   →  FOOTER_SIZE = 16  (alineado)
```

El footer "real" solo ocupa 8 bytes, pero lo alineamos a 16 para mantener la invariante de alineación. Sí, estamos desperdiciando 8 bytes por bloque en padding del footer. Un allocador de producción empaquetaría el footer más agresivamente — por ejemplo, robando bits del tamaño para almacenar el flag de in_use — pero la claridad pedagógica compensa el desperdicio.


### Tamaños y constantes {#tamaños-y-constantes}

```c
#define HEADER_SIZE ((size_t)align_up(sizeof(block_header_t), ALIGNMENT))
#define FOOTER_SIZE ((size_t)align_up(sizeof(block_footer_t), ALIGNMENT))
#define MIN_BLOCK_SIZE (HEADER_SIZE + ALIGNMENT + FOOTER_SIZE)
```

`MIN_BLOCK_SIZE` es nuevo: es el tamaño mínimo que debe tener un bloque para que valga la pena crearlo. Un bloque necesita espacio para header (16) + payload mínimo (16, por alineación) + footer (16) = 48 bytes. Si durante un split quedan menos de 48 bytes, no creamos un nuevo bloque — dejamos ese espacio como fragmentación interna dentro del bloque original.


### La invariante del footer {#la-invariante-del-footer}

La invariante más importante de todo este post:

> Para todo bloque en el heap: `footer->size =` header-&gt;size=.

Cada vez que modificamos `hdr->size` — durante un split, durante un coalesce — debemos actualizar el footer correspondiente. Si esta invariante se rompe, `header_before()` calcula mal la posición del bloque anterior, y el próximo coalesce corrompe el heap. Es el tipo de bug que no se manifiesta inmediatamente sino tres operaciones después, cuando ya has perdido toda referencia al estado que lo causó.


## Sección 4: Funciones de Navegación {#sección-4-funciones-de-navegación}


### footer_of(): acceder al footer de un bloque {#footer-of-acceder-al-footer-de-un-bloque}

```c
static inline block_footer_t *footer_of(block_header_t *hdr)
{
    return (block_footer_t *)((char *)hdr + HEADER_SIZE + hdr->size);
}
```

El footer vive justo después del payload: sumamos `HEADER_SIZE` (para saltar el header) y `hdr->size` (para saltar el payload). Es la posición natural — al final del bloque, antes del inicio del siguiente.


### block_total(): tamaño total de un bloque {#block-total-tamaño-total-de-un-bloque}

```c
static inline size_t block_total(block_header_t *hdr)
{
    return HEADER_SIZE + hdr->size + FOOTER_SIZE;
}
```

Función utilitaria que encapsula el cálculo. Antes era `HEADER_SIZE + hdr->size`. Ahora es `HEADER_SIZE + hdr->size + FOOTER_SIZE`. Cada vez que recorremos el heap o calculamos posiciones, usamos `block_total()` en lugar de hacer la aritmética a mano. Un único punto donde el cálculo puede equivocarse.


### header_after(): el bloque siguiente en O(1) {#header-after-el-bloque-siguiente-en-o--1}

```c
static inline block_header_t *header_after(block_header_t *hdr,
                                           const heap_t *h)
{
    char *next = (char *)hdr + block_total(hdr);
    char *heap_end = (char *)h->start + h->used;
    if (next + HEADER_SIZE > heap_end)
        return NULL;
    return (block_header_t *)next;
}
```

Idéntico en espíritu al Post 3, pero ahora `block_total()` incluye el footer. La verificación `next + HEADER_SIZE > heap_end` asegura que no leamos fuera de la región usada del heap. Si retornamos NULL, el bloque actual es el último — no hay siguiente.


### header_before(): el bloque anterior en O(1) — la novedad {#header-before-el-bloque-anterior-en-o--1--la-novedad}

```c
static inline block_header_t *header_before(block_header_t *hdr,
                                            const heap_t *h)
{
    if ((char *)hdr <= (char *)h->start)
        return NULL;

    block_footer_t *prev_ftr =
        (block_footer_t *)((char *)hdr - FOOTER_SIZE);

    char *prev_start = (char *)hdr - FOOTER_SIZE
                       - prev_ftr->size - HEADER_SIZE;

    if (prev_start < (char *)h->start)
        return NULL;

    return (block_header_t *)prev_start;
}
```

Esta es la función que justifica todo el mecanismo de boundary tags. El flujo:

1.  Si `hdr` es el primer bloque del heap (está en `h->start`), no hay anterior — retornamos NULL.
2.  Retrocedemos `FOOTER_SIZE` bytes desde `hdr`. Eso nos pone en el footer del bloque anterior.
3.  Leemos `prev_ftr->size` — el tamaño del payload del bloque anterior.
4.  Retrocedemos `prev_ftr->size + HEADER_SIZE` bytes más para llegar al header del bloque anterior.
5.  Verificamos que no nos hayamos salido del heap por la izquierda.

Todo son sumas y restas. No hay bucles, no hay búsquedas. O(1) garantizado, independientemente de cuántos bloques haya en el heap. Esta es la propiedad que nos permite coalescer hacia atrás sin penalización de rendimiento.


## Sección 5: Splitting — La Otra Cara de la Moneda {#sección-5-splitting-la-otra-cara-de-la-moneda}


### El problema de los bloques sobrantes {#el-problema-de-los-bloques-sobrantes}

En el Post 3, cuando first-fit encuentra un bloque libre de 256 bytes para una petición de 32, retorna los 256 bytes completos. El usuario solo necesitaba 32, pero recibe 256. Los 224 bytes restantes están desperdiciados: no están en la free list (el bloque entero se marcó como IN_USE) y no se pueden reutilizar hasta que el bloque se libere entero. Esto es **fragmentación interna** — espacio desperdiciado _dentro_ de un bloque asignado.

Splitting resuelve esto. Si un bloque libre es significativamente más grande que lo necesitado, lo dividimos en dos: la primera parte se asigna al usuario, la segunda se mantiene libre en la free list.


### Cuándo dividir {#cuándo-dividir}

No siempre tiene sentido dividir. Un bloque libre de 64 bytes que recibe una petición de 48 tiene 16 bytes sobrantes. ¿Vale la pena crear un nuevo bloque con esos 16 bytes? Ese nuevo bloque necesitaría header (16) + payload (al menos 16) + footer (16) = 48 bytes mínimo. 16 bytes sobrantes no alcanzan ni para el header. Intentar dividir aquí sería un error — estaríamos escribiendo un header fuera de los límites del bloque.

La regla es simple: dividimos solo si los bytes sobrantes son `>` MIN_BLOCK_SIZE= (48 bytes en nuestro caso). Si sobra menos, aceptamos la fragmentación interna y dejamos el bloque entero como está.

<pre style="width: fit-content; margin: 0 auto;">
  Petición: 50 bytes  →  aligned: 64 bytes
  Total necesario:  HEADER_SIZE(16) + 64 + FOOTER_SIZE(16) = 96 bytes

  Bloque libre disponible: 256 bytes totales
  Sobrante: 256 - 96 = 160 bytes  →  >= MIN_BLOCK_SIZE(48)  →  SPLIT

  Resultado:
  ┌──────────────────────┐┌───────────────────────────────┐
  │ Bloque A (asignado)  ││ Bloque B (libre, remanente)   │
  │ HDR + 64B + FTR      ││ HDR + payload_rem + FTR       │
  │ 96 bytes totales     ││ 160 bytes totales             │
  └──────────────────────┘└───────────────────────────────┘
</pre>


### El pseudocódigo de splitting {#el-pseudocódigo-de-splitting}

```text
remaining = block_total(curr) - total_needed

if (remaining >= MIN_BLOCK_SIZE) {
    // Crear un nuevo header en curr + total_needed
    new_hdr = curr + total_needed
    new_hdr->size = remaining - HEADER_SIZE - FOOTER_SIZE
    new_hdr->flags = 0  // FREE
    new_hdr->magic = BLOCK_MAGIC

    // Escribir su footer
    new_ftr = footer_of(new_hdr)
    new_ftr->size = new_hdr->size

    // Insertar en la free list
    insert_to_free_list(h, new_hdr)

    // Ajustar el tamaño del bloque original
    curr->size = aligned_size
}
```

El punto sutil: después del split, debemos actualizar `curr->size` al tamaño real solicitado (no al tamaño original del bloque libre). Y debemos escribir el footer del bloque original con el nuevo tamaño. Si olvidamos cualquiera de los dos, la invariante `footer->size =` header-&gt;size= se rompe, y el próximo coalesce falla.


## Sección 6: heap_free() con Coalescing {#sección-6-heap-free-con-coalescing}


### El nuevo flujo {#el-nuevo-flujo}

```c
void heap_free(heap_t *h, void *ptr)
{
    if (ptr == NULL) return;

    block_header_t *hdr = header_from_payload(ptr);
    if (hdr == NULL) return;

    if (!(hdr->flags & FLAG_IN_USE)) {
        fprintf(stderr,
                "  [DOUBLE-FREE] bloque en %p ya estaba libre\n",
                (void *)hdr);
        return;
    }

    /* Marcar como libre */
    hdr->flags &= ~FLAG_IN_USE;

    /* Escribir footer */
    block_footer_t *ftr = footer_of(hdr);
    ftr->size = hdr->size;

    /* --- Coalescing con el bloque SIGUIENTE --- */
    block_header_t *next_hdr = header_after(hdr, h);
    if (next_hdr != NULL &&
        block_is_valid(next_hdr) &&
        !(next_hdr->flags & FLAG_IN_USE))
    {
        size_t merged = hdr->size + FOOTER_SIZE
                      + HEADER_SIZE + next_hdr->size;
        hdr->size = merged;
        remove_from_free_list(h, next_hdr);
        ftr = footer_of(hdr);
        ftr->size = hdr->size;
    }

    /* --- Coalescing con el bloque ANTERIOR --- */
    block_header_t *prev_hdr = header_before(hdr, h);
    if (prev_hdr != NULL &&
        block_is_valid(prev_hdr) &&
        !(prev_hdr->flags & FLAG_IN_USE))
    {
        size_t merged = prev_hdr->size + FOOTER_SIZE
                      + HEADER_SIZE + hdr->size;
        prev_hdr->size = merged;
        remove_from_free_list(h, prev_hdr);
        ftr = footer_of(prev_hdr);
        ftr->size = prev_hdr->size;
        hdr = prev_hdr;
    }

    /* Insertar el bloque (fusionado o no) en la free list */
    insert_to_free_list(h, hdr);
}
```

Analicemos paso a paso.

**Las tres primeras comprobaciones** son idénticas al Post 3: NULL check, magic check (vía `header_from_payload()`), y double-free check. Sin cambios.

**Marcar como libre y escribir el footer.** Aquí está la primera novedad: inmediatamente después de limpiar el flag `FLAG_IN_USE`, escribimos el footer del bloque. Esto es necesario porque los bloques IN_USE del bump allocator ya tenían footer escrito (lo hacemos en `heap_alloc()`), pero queremos asegurarnos de que el footer esté actualizado antes de intentar coalescer.

**Coalescing con el siguiente.** Calculamos `header_after(hdr, h)` — el header del bloque que viene después. Si existe, tiene magic válido, y no está en uso, fusionamos. El nuevo tamaño es la suma de ambos payloads más los bytes de metadata intermedios que ahora se "absorben":

```text
merged = hdr->size       (payload del bloque actual)
       + FOOTER_SIZE     (footer del bloque actual, ahora interior)
       + HEADER_SIZE     (header del siguiente, ahora interior)
       + next_hdr->size  (payload del siguiente)
```

Actualizamos `hdr->size` con el nuevo tamaño, sacamos el bloque siguiente de la free list (con `remove_from_free_list()`), y escribimos el footer del bloque fusionado.

**Coalescing con el anterior.** Mismo proceso pero en dirección contraria. Usamos `header_before(hdr, h)` — la función que justifica los boundary tags. Si el bloque anterior existe, es válido, y está libre, fusionamos. La diferencia crítica es que aquí el header resultante es el del bloque _anterior_, no el del bloque actual:

```c
hdr = prev_hdr;  // El cabezal del bloque unificado es el anterior
```

Esta reasignación es la razón por la que coalescemos primero hacia adelante y luego hacia atrás. Si hiciéramos el orden inverso, después de fusionar con el anterior, `hdr` cambiaría, y el cálculo del bloque siguiente sería incorrecto (estaríamos calculándolo desde el header anterior, no desde el original).

**Insertar en la free list.** Finalmente, insertamos el bloque resultante — que puede ser el original, el original fusionado con el siguiente, el original fusionado con el anterior, o los tres fusionados — en la free list.


### La aritmética del merge {#la-aritmética-del-merge}

Visualicemos exactamente qué ocurre cuando fusionamos dos bloques adyacentes:

<pre style="width: fit-content; margin: 0 auto;">
  ANTES (dos bloques libres adyacentes):
  ┌──────┬───────────┬──────┐┌──────┬───────────┬──────┐
  │ HDR  │ payload A │ FTR  ││ HDR  │ payload B │ FTR  │
  │ 16B  │  A bytes  │ 16B  ││ 16B  │  B bytes  │ 16B  │
  └──────┴───────────┴──────┘└──────┴───────────┴──────┘

  DESPUÉS (un solo bloque libre):
  ┌──────┬─────────────────────────────────────────┬──────┐
  │ HDR  │     payload fusionado                   │ FTR  │
  │ 16B  │  A + 16 + 16 + B bytes                  │ 16B  │
  └──────┴─────────────────────────────────────────┴──────┘

  El nuevo payload = A + FOOTER_SIZE + HEADER_SIZE + B
</pre>

El footer del primer bloque y el header del segundo "desaparecen" — sus bytes se convierten en payload del bloque fusionado. El header del primer bloque y el footer del segundo son los que sobreviven como metadata del bloque unificado.


## Sección 7: heap_alloc() con Splitting {#sección-7-heap-alloc-con-splitting}


### El nuevo flujo {#el-nuevo-flujo}

```c
void *heap_alloc(heap_t *h, size_t size)
{
    if (h->start == NULL) return NULL;
    if (size == 0)        return NULL;

    size_t aligned_size = align_up(size, ALIGNMENT);
    size_t total_needed = HEADER_SIZE + aligned_size + FOOTER_SIZE;

    /* --- Fase 1: free list (first-fit) --- */
    block_header_t *prev = NULL;
    block_header_t *curr = h->free_list_head;

    while (curr != NULL) {
        if (curr->magic != BLOCK_MAGIC) {
            fprintf(stderr,
                    "  [CORRUPCION] free list corrupta en %p\n",
                    (void *)curr);
            break;
        }

        size_t curr_total = block_total(curr);
        if (curr_total >= total_needed) {
            /* Sacar de la free list */
            if (prev == NULL)
                h->free_list_head = free_node_of(curr)->next;
            else
                free_node_of(prev)->next = free_node_of(curr)->next;

            /* --- SPLITTING --- */
            size_t remaining = curr_total - total_needed;

            if (remaining >= MIN_BLOCK_SIZE) {
                block_header_t *new_hdr =
                    (block_header_t *)((char *)curr + total_needed);
                new_hdr->size  = remaining - HEADER_SIZE - FOOTER_SIZE;
                new_hdr->flags = 0;
                new_hdr->magic = BLOCK_MAGIC;

                block_footer_t *new_ftr = footer_of(new_hdr);
                new_ftr->size = new_hdr->size;

                insert_to_free_list(h, new_hdr);

                curr->size = aligned_size;
            }

            curr->flags = FLAG_IN_USE;
            block_footer_t *ftr = footer_of(curr);
            ftr->size = curr->size;

            return payload_of(curr);
        }

        prev = curr;
        curr = free_node_of(curr)->next;
    }

    /* --- Fase 2: bump allocator --- */
    if (h->used + total_needed > h->capacity) {
        size_t grow = total_needed;
        if (grow < h->capacity) grow = h->capacity;

        void *result = sbrk((intptr_t)grow);
        if (result == (void *)-1) return NULL;

        h->capacity += grow;
        h->brk = (char *)h->start + h->capacity;
    }

    block_header_t *hdr =
        (block_header_t *)((char *)h->start + h->used);
    hdr->size  = aligned_size;
    hdr->flags = FLAG_IN_USE;
    hdr->magic = BLOCK_MAGIC;

    block_footer_t *ftr = footer_of(hdr);
    ftr->size = aligned_size;

    h->used += total_needed;
    return payload_of(hdr);
}
```

Los cambios respecto al Post 3 son tres:

**1. `total_needed` incluye el footer.** Antes era `HEADER_SIZE + aligned_size`. Ahora es `HEADER_SIZE + aligned_size + FOOTER_SIZE`. Este cambio afecta tanto a la búsqueda en la free list (necesitamos un bloque más grande) como al bump allocator (avanzamos `h->used` más).

**2. Splitting después de sacar de la free list.** Antes, cuando encontrábamos un bloque en la free list, lo marcábamos como IN_USE y lo retornábamos. Ahora, antes de retornarlo, calculamos `remaining = curr_total - total_needed`. Si `remaining >` MIN_BLOCK_SIZE=, creamos un nuevo bloque con el remanente. Si no, aceptamos la fragmentación interna.

**3. Footer en el bump allocator.** Cada bloque nuevo creado por el bump allocator ahora también tiene footer. Esto es necesario porque esos bloques eventualmente serán liberados, y cuando se liberen, sus vecinos necesitarán leer su footer para coalescer.


### El flujo del splitting visualizado {#el-flujo-del-splitting-visualizado}

<pre style="width: fit-content; margin: 0 auto;">
  ANTES del split:
  ┌──────┬──────────────────────────────────┬──────┐
  │ HDR  │        payload libre (256B)      │ FTR  │
  │ 16B  │                                  │ 16B  │
  └──────┴──────────────────────────────────┴──────┘
  total = 288B

  Petición: 64 bytes alineados
  total_needed = 16 + 64 + 16 = 96B
  remaining = 288 - 96 = 192B  >= MIN_BLOCK_SIZE(48)  →  SPLIT

  DESPUÉS del split:
  ┌──────┬──────────┬──────┐┌──────┬───────────────┬──────┐
  │ HDR  │ pay 64B  │ FTR  ││ HDR  │ pay 160B      │ FTR  │
  │ 16B  │ IN_USE   │ 16B  ││ 16B  │ FREE          │ 16B  │
  └──────┴──────────┴──────┘└──────┴───────────────┴──────┘
  96B (retornado)            192B (en free list)
  </pre>


## Sección 8: Operaciones sobre la Free List {#sección-8-operaciones-sobre-la-free-list}


### insert_to_free_list() {#insert-to-free-list}

```c
static void insert_to_free_list(heap_t *h, block_header_t *hdr)
{
    free_block_t *node = free_node_of(hdr);
    node->next = h->free_list_head;
    h->free_list_head = hdr;
}
```

Sin cambios respecto al Post 3. Inserción al inicio, O(1). El bloque más recientemente liberado queda al principio de la lista.


### remove_from_free_list() {#remove-from-free-list}

```c
static void remove_from_free_list(heap_t *h, block_header_t *target)
{
    block_header_t *prev = NULL;
    block_header_t *curr = h->free_list_head;

    while (curr != NULL) {
        if (curr == target) {
            if (prev == NULL)
                h->free_list_head = free_node_of(curr)->next;
            else
                free_node_of(prev)->next = free_node_of(curr)->next;
            return;
        }
        prev = curr;
        curr = free_node_of(curr)->next;
    }
}
```

Esta función es nueva. En el Post 3, solo sacábamos el bloque de la free list dentro de `heap_alloc()`, donde ya teníamos el puntero `prev` del recorrido first-fit. Ahora necesitamos sacar bloques _durante el coalescing_, cuando no estamos recorriendo la free list — estamos en medio de un `free()` y sabemos qué bloque queremos eliminar, pero no dónde está en la lista.

La implementación es un recorrido lineal O(n). Es la parte más cara del coalescing. Una free list doblemente enlazada la haría O(1), pero añadiría otro puntero al payload de cada bloque libre. Para nuestro allocador educativo, O(n) es aceptable. Para uno de producción, no — `glibc malloc` usa listas doblemente enlazadas exactamente por esta razón.


## Sección 9: Visualización: Antes y Después {#sección-9-visualización-antes-y-después}


### Paso a paso con heap_dump() {#paso-a-paso-con-heap-dump}

Ejecutemos el programa y veamos qué ocurre en cada paso.

**Paso 1: Tres allocaciones.**

<pre style="width: fit-content; margin: 0 auto;">
  a = heap_alloc(h, 40)    →  alineado a 48 bytes
  b = heap_alloc(h, 128)   →  alineado a 128 bytes
  c = heap_alloc(h, 64)    →  alineado a 64 bytes
  ┌──────┬──────┬──────┐┌──────┬───────┬──────┐┌──────┬──────┬──────┐
  │ HDR  │ 48B  │ FTR  ││ HDR  │ 128B  │ FTR  ││ HDR  │ 64B  │ FTR  │
  │      │IN_USE│      ││      │IN_USE │      ││      │IN_USE│      │
  └──────┴──────┴──────┘└──────┴───────┴──────┘└──────┴──────┴──────┘
    A (80B total)          B (160B total)         C (96B total)

  Free list: (vacia)
  </pre>

Cada bloque paga 32 bytes de metadata (16 header + 16 footer). A ocupa 80 bytes totales (16+48+16), B ocupa 160 (16+128+16), C ocupa 96 (16+64+16). Total: 336 bytes de 4096.

**Paso 2: Liberar B.**

<pre style="width: fit-content; margin: 0 auto;">
  heap_free(h, b)

  ┌──────┬──────┬──────┐┌──────┬───────┬──────┐┌──────┬──────┬──────┐
  │ HDR  │ 48B  │ FTR  ││ HDR  │ 128B  │ FTR  ││ HDR  │ 64B  │ FTR  │
  │      │IN_USE│      ││      │ FREE  │      ││      │IN_USE│      │
  └──────┴──────┴──────┘└──────┴───────┴──────┘└──────┴──────┴──────┘
    A (en uso)             B (libre)              C (en uso)

  Free list: B -> NULL
  </pre>

B queda libre, pero sus vecinos (A y C) están en uso. No hay coalescencia posible. B se inserta en la free list tal cual.

**Paso 3: Liberar C → COALESCENCIA con B.**

<pre style="width: fit-content; margin: 0 auto;">
  heap_free(h, c)

  1. C se marca como libre.
  2. header_after(C) → NULL (C es el ultimo bloque). Sin coalescencia.
  3. header_before(C) → B. B esta libre. FUSIONAR.
     merged_size = B.size(128) + FTR(16) + HDR(16) + C.size(64) = 224
     B se saca de la free list. hdr = B.
  4. B (ahora B+C) se inserta en la free list.

  ┌──────┬──────┬──────┐┌──────┬────────────────────────────┬──────┐
  │ HDR  │ 48B  │ FTR  ││ HDR  │       224B                 │ FTR  │
  │      │IN_USE│      ││      │       FREE (B+C)           │      │
  └──────┴──────┴──────┘└──────┴────────────────────────────┴──────┘
    A (en uso)             B+C (libre, fusionados)

  Free list: B+C -> NULL
  </pre>

El bloque fusionado tiene 224 bytes de payload — suficiente para servir peticiones de hasta 224 bytes. Los 32 bytes de metadata que C pagaba (su header y footer) ahora son parte del payload de B+C. La metadata se redujo de 96 bytes (3 bloques × 32) a 64 bytes (2 bloques × 32).

**Paso 4: Alloc(50) → SPLITTING.**

<pre style="width: fit-content; margin: 0 auto;">
  d = heap_alloc(h, 50)    →  alineado a 64 bytes

  total_needed = 16 + 64 + 16 = 96B
  Bloque B+C tiene 256B totales (16 + 224 + 16).
  remaining = 256 - 96 = 160B  >= MIN_BLOCK_SIZE(48)  →  SPLIT

  ┌──────┬──────┬──────┐┌──────┬──────┬──────┐┌──────┬───────┬──────┐
  │ HDR  │ 48B  │ FTR  ││ HDR  │ 64B  │ FTR  ││ HDR  │ 128B  │ FTR  │
  │      │IN_USE│      ││      │IN_USE│      ││      │ FREE  │      │
  └──────┴──────┴──────┘└──────┴──────┴──────┘└──────┴───────┴──────┘
    A (en uso)             D (nuevo)             remanente (libre)

  Free list: remanente -> NULL
  </pre>

El bloque fusionado se divide: D ocupa los primeros 96 bytes, y el remanente (160 bytes totales, 128 de payload) queda libre en la free list. El usuario pidió 50, recibió 64 (alineación), y no desperdició los 160 bytes restantes.

**Pasos 5-8: Coalescencia en cadena.**

Los pasos siguientes del programa demuestran coalescencia progresiva. Cuando liberas D (Paso 7), se fusiona con A (que fue liberado en el Paso 6) porque son adyacentes. Cuando liberas E (Paso 8), el bloque se fusiona con el bloque A+D a su izquierda. El resultado final: un único bloque libre de 304 bytes de payload.

<pre style="width: fit-content; margin: 0 auto;">
  Paso 8 (estado final):
  ┌──────┬──────────────────────────────────────────┬──────┐
  │ HDR  │          304B payload (FREE)              │ FTR  │
  └──────┴──────────────────────────────────────────┴──────┘

  Free list: bloque_unico -> NULL
  </pre>

Todos los bloques se fusionaron. De 3 bloques × 32B de metadata = 96B, pasamos a 1 bloque × 32B = 32B. El allocador recuperó 64 bytes de metadata que ya no necesita. El heap se desfragmentó por completo.


## Sección 10: El Balance entre Coalescing y Cache Locality {#sección-10-el-balance-entre-coalescing-y-cache-locality}


### Coalescing no es gratis {#coalescing-no-es-gratis}

Hay una tensión que vale la pena reconocer: coalescing mejora la fragmentación pero puede degradar la cache locality.

Consideremos un patrón de uso donde un programa alloca muchos bloques pequeños del mismo tamaño (digamos, 64 bytes cada uno). Algunos los libera, otros los mantiene. Sin coalescing, los bloques libres mantienen su tamaño original. Cuando el programa pide otro bloque de 64 bytes, first-fit lo encuentra inmediatamente — probablemente cerca de los bloques que el programa está usando activamente. Buena localidad de cache.

Con coalescing inmediato, esos bloques libres de 64 bytes se fusionan en bloques más grandes. Cuando el programa pide 64 bytes, el allocador hace un split del bloque grande, que puede estar lejos en memoria de los bloques activos. La localidad de cache se degrada.

<pre style="width: fit-content; margin: 0 auto;">
  Sin coalescing:
  [USED 64][FREE 64][USED 64][FREE 64][USED 64]
  → alloc(64) reusa un hueco cercano. Buena localidad.

  Con coalescing:
  [USED 64][====FREE 128====][USED 64]
  → alloc(64) divide el bloque grande. Potencialmente lejos.
  </pre>

Este trade-off es real. `tcmalloc` y `jemalloc` usan **thread-local caches** (tcache/slab allocators) que mantienen pools de bloques pequeños sin coalescer, precisamente para preservar la localidad. Solo coalescen cuando la presión de memoria lo justifica. Es un tema que exploraremos en posts posteriores cuando implementemos arenas por tamaño.

Por ahora, nuestro allocador coalesce siempre. Es la decisión correcta para un allocador educativo donde la prioridad es entender la mecánica, no optimizar para cache L1.


## Sección 11: Garantías e Invariantes {#sección-11-garantías-e-invariantes}

Listamos explícitamente las invariantes que el allocador mantiene después de este post:

**Invariante 1 — Magic.** Todo header tiene `magic =` 0xDEADBEEF= y `size > 0`. Un magic corrupto indica escritura fuera de límites o use-after-free.

**Invariante 2 — Footer.** Para todo bloque: `footer_of(hdr)->size =` hdr-&gt;size=. Si esta invariante se rompe, `header_before()` calcula posiciones incorrectas y el coalescing corrompe el heap.

**Invariante 3 — Alineación.** Todo payload comienza en una dirección alineada a `ALIGNMENT` (16 bytes). Esto se mantiene porque `HEADER_SIZE` y `FOOTER_SIZE` están ambos alineados, y los tamaños de payload se alinean hacia arriba.

**Invariante 4 — Free list.** Solo bloques con `FLAG_IN_USE =` 0= están en la free list. No hay bloques duplicados en la free list. Todo bloque en la free list tiene magic válido.

**Invariante 5 — Min block size.** Todo bloque en el heap tiene un tamaño total `>` MIN_BLOCK_SIZE= (48 bytes). Splitting nunca crea bloques menores.

**Invariante 6 — No adyacencia libre.** Después de cada `heap_free()`, no existen dos bloques libres adyacentes en memoria. El coalescing inmediato garantiza que siempre se fusionan.

La invariante 6 es nueva y es la más poderosa. Significa que la fragmentación externa del heap está minimizada: todo espacio libre que puede fusionarse, ya está fusionado. La única fragmentación que queda es la interna (padding de alineación y splits fallidos) y la temporal (bloques en uso que separan bloques libres).


## Sección 12: Posibles Extensiones {#sección-12-posibles-extensiones}

Tres extensiones naturales que exploraremos en posts futuros:

**Best-fit allocation.** En lugar de retornar el primer bloque que cabe (first-fit), recorrer toda la free list y retornar el bloque más ajustado al tamaño pedido. Reduce la fragmentación interna pero convierte cada alloc en O(n). El Post 5 comparará first-fit, best-fit, y next-fit con métricas concretas.

**Buddy allocator.** Dividir bloques en potencias de 2 para simplificar el coalescing: un bloque de 64 solo se fusiona con su "buddy" de 64 para formar un bloque de 128. No necesita boundary tags porque las posiciones de los buddies se calculan con XOR. Lo veremos cuando implementemos arenas.

**Address-ordered free list.** Ordenar la free list por dirección de memoria en lugar de por tiempo de inserción. Mejora la localidad espacial de las allocaciones y hace que coalescing sea más predecible. El costo es que la inserción pasa de O(1) a O(n).
