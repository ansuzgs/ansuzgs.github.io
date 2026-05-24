+++
title = "Lo que malloc() no quiere que sepas"
author = ["pablo"]
date = 2026-05-24
draft = false
translationKey = "post-01-no-malloc"
series = ["Memory Allocation y GC desde cero"]
subtitle = "Un Bump Allocator en 80 líneas de C"
seriesPost = 1
+++

_Post 1 de 13 — Serie: Memory Allocation y Garbage Collection desde cero_

Cada vez que escribes `malloc(128)`, ocurre algo aparentemente mágico: el runtime te devuelve un puntero a 128 bytes de memoria que nadie más está usando. No pediste permiso al sistema operativo. No especificaste _dónde_ querían vivir esos bytes. Simplemente aparecieron. Y cuando llamas `free()`, desaparecen de vuelta al vacío.

La magia es mentira.

Debajo de `malloc()` no hay nada sofisticado. Hay una syscall que mueve un número hacia arriba. Hay un puntero que avanza. Hay una estructura de datos que lleva la cuenta. Tu programa, cualquier programa en C, está corriendo código muy parecido al que vamos a escribir hoy.

En este post, construiremos un **allocador de memoria funcional** en aproximadamente 80 líneas de C. Es pequeño. Es legible. No hace todo lo que hace `malloc()`, le faltan piezas importantes que añadiremos en posts posteriores. Pero lo que sí hace es _real_: pide memoria al sistema operativo, la reparte entre llamantes, y la gestiona con una struct y tres funciones.

La promesa de esta serie es simple: al final de los 13 posts, habrás construido, con tus manos, un allocador completo con free lists, coalescing, arenas basadas en `mmap()`, y un garbage collector funcional. Todo compilable, todo ejecutable, todo explicado.

Empezamos por lo más básico. Empezamos por el _bump pointer_.


## Sección 1: Cómo funciona malloc() de verdad {#sección-1-cómo-funciona-malloc-de-verdad}


### El espacio de direcciones de un proceso {#el-espacio-de-direcciones-de-un-proceso}

Cuando el kernel de Linux carga tu programa, le asigna un **espacio de direcciones virtual**. En x86-64, ese espacio es enorme (48 bits de direcciones útiles), pero la estructura lógica es la misma que en cualquier arquitectura: unas cuantas regiones bien definidas con roles fijos.

Lo que nos importa son dos:

<pre style="width: fit-content; margin: 0 auto;">
 Direcciones altas (0x7fff...)
┌─────────────────────────┐
│         STACK           │  ← crece hacia abajo (cada llamada a función)
│           │             │
│           ▼             │
│                         │
│     (espacio libre)     │
│                         │
│           ▲             │
│           │             │
│         HEAP            │  ← crece hacia arriba (cada malloc)
├─────────────────────────┤
│   program break (brk)   │  ← frontera: debajo es tuyo, arriba no
├─────────────────────────┤
│      BSS / Data         │
│      Text (código)      │
└─────────────────────────┘
 Direcciones bajas (0x0000...)
 </pre>

El **stack** crece hacia abajo cada vez que llamas a una función. El **heap** crece hacia arriba cada vez que necesitas memoria dinámica. Entre ambos hay un océano de espacio virtual sin mapear.

El punto crítico es el **program break** (`brk`). Es una variable que el kernel mantiene por proceso. Todo lo que está _debajo_ del break es memoria que tu proceso puede leer y escribir. Todo lo que está _arriba_ es territorio del kernel, si intentas tocarlo, recibes un `SIGSEGV`.


### La syscall sbrk(2) {#la-syscall-sbrk--2}

`sbrk()` es la herramienta que mueve ese break. Su firma conceptual es trivial:

```c
void *sbrk(intptr_t increment);
```

Lo que hace: mueve el program break `increment` bytes hacia arriba (o hacia abajo si `increment` es negativo). Retorna la posición _anterior_ del break, es decir, la dirección base de la nueva región.

Llamar `sbrk(0)` sin mover nada te dice dónde está el break ahora mismo. Es la manera de "preguntar" sin "pedir".

<pre style="width: fit-content; margin: 0 auto;">
Antes de sbrk(256):
        ┌───────────────────┐
        │   espacio libre   │
 brk ──►├───────────────────┤
        │  heap existente   │
        └───────────────────┘
Después de sbrk(256):
        ┌───────────────────┐
        │   espacio libre   │
 brk ──►├───────────────────┤
        │  256 bytes nuevos │  ← sbrk retorna esta dirección
        ├───────────────────┤
        │  heap existente   │
        └───────────────────┘
 </pre>

Una vez que llamas `sbrk(n)`, esos `n` bytes son _tuyos_. El kernel ha mapeado las páginas virtuales correspondientes a memoria física (o al menos ha prometido hacerlo cuando las toques, _demand paging_). Puedes leerlos y escribirlos hasta que el proceso termine.

Hay un invariante importante aquí: `sbrk()` no te da bloques aislados. Te da una **extensión contigua** del heap. Todo lo que has pedido hasta ahora forma un array continuo de bytes. Esta propiedad es simultáneamente su mayor fortaleza (simplicidad) y su mayor debilidad (rigidez). En el Post 7, cuando migremos a `mmap()`, veremos cómo superar esta limitación.


### El modelo mental: el heap lineal {#el-modelo-mental-el-heap-lineal}

Con lo que sabemos, la vida de `malloc()` se reduce a esto:

1.  Al arrancar, el heap es un array vacío. `sbrk(0)` nos dice dónde empieza.
2.  Alguien pide `n` bytes. Movemos el break `n` bytes hacia arriba con `sbrk(n)`. Retornamos la dirección base.
3.  Alguien pide `m` bytes más. Volvemos a mover el break. Retornamos la nueva base.
4.  Repetimos.

En pseudocódigo:

```text
allocate(size):
    old_break = sbrk(size)
    if old_break == error:
        return NULL
    return old_break
```

Eso es un **bump allocator**, un allocador que solo avanza un puntero. Nunca retrocede. No necesita recordar qué bloques están libres porque _ningún bloque se libera jamás_. Es la estrategia de allocation más simple posible, y es exactamente lo que vamos a implementar.

¿Por qué empezar con algo tan limitado? Porque hace visible la mecánica fundamental sin ruido. El bump allocator te muestra qué está haciendo `sbrk()`, cómo se organiza la memoria, y dónde viven tus datos. Esas intuiciones sobreviven intactas cuando, en los posts siguientes, añadamos complejidad encima.


## Sección 2: El Bump Allocator — Código {#sección-2-el-bump-allocator-código}


### La struct heap_t {#la-struct-heap-t}

Toda la información de nuestro allocador vive en una struct:

```c
typedef struct {
    void  *start;       /* dirección base de la región sbrk'd          */
    void  *brk;         /* program break actual (start + capacity)     */
    size_t capacity;    /* bytes totales pedidos al OS                 */
    size_t used;        /* bytes asignados (offset del bump pointer)   */
} heap_t;
```

Cuatro campos. Cada uno está ahí por una razón concreta, y cada uno pagará dividendos en posts futuros:

`start` es la dirección base que `sbrk()` nos dio al inicializar. No cambia nunca. Es el "cero" del heap, todas las posiciones internas se calculan como offsets desde aquí. En el Post 7, cuando tengamos múltiples arenas con `mmap()`, cada arena tendrá su propio `start`.

`brk` es el program break actual. Siempre igual a `start + capacity`. Lo mantenemos explícito en lugar de calcularlo cada vez porque, en posts futuros, `brk` se desacoplará de `start + capacity` cuando tengamos regiones no contiguas.

`capacity` es cuántos bytes hemos pedido al OS _en total_. No cuántos hemos repartido, cuántos tenemos disponibles. Cuando un `heap_alloc()` no cabe, pedimos más con `sbrk()` y actualizamos `capacity`.

`used` es el bump pointer en sí: cuántos bytes de `capacity` ya hemos repartido. Siempre que alguien pide memoria, `used` avanza. Nunca retrocede. La posición libre actual es siempre `start + used`.

¿Por qué una struct y no cuatro variables globales? Es una decisión de diseño que parece innecesaria ahora, pero que evita un rewrite total más adelante. En el Post 7, `heap_t` evolucionará hacia una `allocator_t` que puede manejar múltiples arenas, cada una con su propio `start` y `capacity`. Si empezáramos con globales, la transición sería dolorosa. Con una struct, el refactor será mecánico.


### heap_init() {#heap-init}

```c
heap_t heap_init(size_t initial_size)
{
    heap_t h;
    h.start = sbrk(0);               /* ¿dónde está el break ahora?   */
    if (h.start == (void *)-1) {
        h.start = NULL;  h.brk = NULL;
        h.capacity = 0;  h.used = 0;
        return h;
    }
    void *result = sbrk((intptr_t)initial_size);  /* mueve el break    */
    if (result == (void *)-1) {
        h.start = NULL;  h.brk = NULL;
        h.capacity = 0;  h.used = 0;
        return h;
    }
    h.brk      = (char *)h.start + initial_size;
    h.capacity = initial_size;
    h.used     = 0;
    return h;
}
```

El flujo es directo. Primero preguntamos dónde está el break con `sbrk(0)`. Si eso falla (retorna `(void *)-1`), la situación es tan catastrófica que retornamos un heap nulo, el llamante debe comprobarlo. Luego movemos el break `initial_size` bytes con `sbrk(initial_size)`. Si _eso_ falla, mismo tratamiento.

¿Por qué dos llamadas en lugar de una? Porque `sbrk(n)` retorna el break _anterior_, no el nuevo. Necesitamos saber dónde _empezó_ nuestra región, no solo que creció. La primera llamada con `sbrk(0)` nos da esa base; la segunda hace el trabajo real.

El manejo de errores aquí es deliberadamente simple. Un allocador de producción haría cosas más sofisticadas, `jemalloc`, por ejemplo, tiene múltiples fallbacks. Nosotros retornamos NULL y confiamos en que el llamante comprueba. Es un hack pedagógico, y lo reconocemos. En posts posteriores, el manejo de errores mejorará.


### heap_alloc() {#heap-alloc}

```c
void *heap_alloc(heap_t *h, size_t size)
{
    if (h->start == NULL) return NULL;
    if (size == 0) return NULL;
    /* ¿Cabe en la capacidad actual? */
    if (h->used + size > h->capacity) {
        size_t grow = size;
        if (grow < h->capacity)
            grow = h->capacity;       /* duplicar como heurística */
        void *result = sbrk((intptr_t)grow);
        if (result == (void *)-1)
            return NULL;
        h->capacity += grow;
        h->brk = (char *)h->start + h->capacity;
    }
    void *ptr = (char *)h->start + h->used;
    h->used += size;
    return ptr;
}
```

Esta es la función central, y cabe en 15 líneas significativas.

Los dos primeros guards son defensivos: si el heap no se inicializó correctamente o si alguien pide cero bytes, retornamos NULL sin hacer nada. El caso de cero bytes merece una nota, `malloc(0)` tiene comportamiento _implementation-defined_ según el estándar C. Puede retornar NULL o un puntero único que no debes dereferenciar. Nosotros elegimos NULL por simplicidad.

El bloque de crecimiento es interesante. Si lo que piden no cabe en la capacidad actual, necesitamos más memoria. La heurística es: pedir al menos lo que necesitamos, pero si la capacidad actual es mayor, duplicarla. Duplicar la capacidad es una estrategia amortizada clásica, la misma que usa `std::vector` en C++, y por la misma razón: evitar `O(n)` llamadas a `sbrk()` para `n` allocaciones.

Las dos líneas finales son el _bump_: calculamos la dirección actual (`start + used`), avanzamos `used`, y retornamos la dirección. Eso es todo. No hay metadata, no hay listas, no hay contabilidad de bloques individuales. Esa simplicidad es el punto, y también es la limitación que los posts futuros resolverán.

Nota la ausencia total de **alineación**. Si pides 3 bytes seguidos de un `int*`, el `int*` empezará en una dirección no alineada. En x86-64 esto _funciona_ (los accesos desalineados son legales pero lentos), pero en ARM o RISC-V puede causar un trap. El Post 2 introducirá block headers que fuerzan alineación a 8 o 16 bytes.


### heap_dump() {#heap-dump}

```c
void heap_dump(const heap_t *h)
{
    if (h->start == NULL) {
        printf("  [heap no inicializado]\n");
        return;
    }
    double pct_used = (double)h->used / (double)h->capacity * 100.0;
    size_t free_bytes = h->capacity - h->used;
    const int bar_width = 50;
    int used_bar = (int)((double)h->used / (double)h->capacity * bar_width);
    int free_bar = bar_width - used_bar;
    printf("╔══════════════════════════════════════════════════════╗\n");
    printf("║       HEAP DUMP — Post 1 (Bump Allocator)          ║\n");
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║ Heap base:     %p                    ║\n", h->start);
    printf("║ Current break: %p                    ║\n", h->brk);
    printf("║ Capacity:      %-6zu bytes                         ║\n",
           h->capacity);
    printf("║ Used:          %-6zu bytes (%5.1f%%)                ║\n",
           h->used, pct_used);
    printf("║ Free:          %-6zu bytes (%5.1f%%)                ║\n",
           free_bytes, 100.0 - pct_used);
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║ [");
    for (int i = 0; i < used_bar; i++) printf("=");
    printf(" USED ][");
    for (int i = 0; i < free_bar; i++) printf("-");
    printf(" FREE ]║\n");
    printf("╚══════════════════════════════════════════════════════╝\n");
}
```

La función de dump es más larga que las otras dos juntas, y eso es intencional. La visualización es pedagogía, es lo que transforma números abstractos en comprensión espacial.


## Sección 3: Visualización — Entendiendo tu heap {#sección-3-visualización-entendiendo-tu-heap}


### Ejemplo 1: Allocaciones básicas {#ejemplo-1-allocaciones-básicas}

```c
heap_t h = heap_init(1024);
int  *nums = heap_alloc(&h, sizeof(int) * 10);   /* 40 bytes */
char *msg  = heap_alloc(&h, 128);
for (int i = 0; i < 10; i++)
    nums[i] = i * i;
strcpy(msg, "Hola desde nuestro heap custom!");
heap_dump(&h);
printf("nums[5] = %d\n", nums[5]);
printf("msg     = \"%s\"\n", msg);
```

Salida (direcciones variarán):

<pre style="width: fit-content; margin: 0 auto;">
╔═════════════════════════════════════════════════════╗
║       HEAP DUMP — Post 1 (Bump Allocator)           ║
╠═════════════════════════════════════════════════════╣
║ Heap base:     0x555555576000                       ║
║ Current break: 0x555555576400                       ║
║ Capacity:      1024   bytes                         ║
║ Used:          168    bytes ( 16.4%)                ║
║ Free:          856    bytes ( 83.6%)                ║
╠═════════════════════════════════════════════════════╣
║ [======== USED ][--------------------- FREE ]       ║
╚═════════════════════════════════════════════════════╝
nums[5] = 25
msg     = "Hola desde nuestro heap custom!"
</pre>

168 bytes usados de 1024 disponibles. `nums` y `msg` viven uno junto al otro, sin separación: `nums` empieza en offset 0 y ocupa 40 bytes, `msg` empieza en offset 40 y ocupa 128 bytes. No hay padding, no hay headers, no hay huecos. Es memoria cruda y contigua.


### Ejemplo 2: Múltiples allocaciones variadas {#ejemplo-2-múltiples-allocaciones-variadas}

```c
heap_t h = heap_init(1024);
void *a = heap_alloc(&h, 16);     /* offset 0   */
void *b = heap_alloc(&h, 32);     /* offset 16  */
void *c = heap_alloc(&h, 400);    /* offset 48  */
void *d = heap_alloc(&h, 50);     /* offset 448 */
heap_dump(&h);
```

Los offsets son predecibles: 0, 16, 48, 448. Cada bloque empieza exactamente donde termina el anterior. Y aquí está el problema que el dump revela implícitamente: **si hiciéramos `free(b)`** (los 32 bytes del segundo bloque), ¿cómo recuperaríamos ese espacio? El bump allocator no tiene manera de saberlo. El Post 2 añade block headers para resolver exactamente esto.


### Ejemplo 3: Crecimiento automático {#ejemplo-3-crecimiento-automático}

```c
heap_t h = heap_init(100);     /* heap pequeño */
printf("Capacidad inicial: %zu\n", h.capacity);
void *big = heap_alloc(&h, 500);  /* no cabe → crece */
heap_dump(&h);
```

La capacidad salta de 100 a 600. Esto demuestra la heurística de crecimiento: como 500 &gt; 100 (la capacidad actual), pedimos exactamente 500 en lugar de duplicar. Si hubiéramos pedido 80 bytes con un heap de 100, la capacidad habría saltado a 200 (duplicación).


## Sección 4: Pruebas y casos límite {#sección-4-pruebas-y-casos-límite}


### ¿Qué pasa si allocas más de la capacidad? {#qué-pasa-si-allocas-más-de-la-capacidad}

El heap crece automáticamente. `heap_alloc()` llama a `sbrk()` para pedir más bytes. Esto funciona porque `sbrk()` extiende el heap de manera contigua. ¿Tiene límite? Sí. El kernel impone un límite por proceso (configurable con `ulimit -v`), y eventualmente las páginas virtuales se agotan. Cuando `sbrk()` falla, retorna `(void *)-1` y nuestro `heap_alloc()` retorna NULL.


### ¿Qué pasa si allocas 0 bytes? {#qué-pasa-si-allocas-0-bytes}

```c
void *zero = heap_alloc(&h, 0);
// zero == NULL
```

Nuestra implementación retorna NULL. El estándar C dice que `malloc(0)` puede retornar NULL o un puntero único no-dereferenciable. Ambos son legales. Nosotros elegimos NULL porque es más simple y más seguro.


### ¿Qué pasa cuando el programa termina sin "free"? {#qué-pasa-cuando-el-programa-termina-sin-free}

Nada malo. Cuando un proceso termina, el kernel reclama _toda_ su memoria virtual, heap, stack, todo. No importa si llamaste `free()` o no. Para programas cortos (utilidades de línea de comandos, scripts, herramientas), no liberar memoria explícitamente es una estrategia legítima. El problema aparece en programas de larga duración (servidores, daemons) donde la memoria no liberada se acumula hasta agotar los recursos.


### Reserva contigua: una invariante que importa {#reserva-contigua-una-invariante-que-importa}

```c
heap_t h = heap_init(1024);
char *a = heap_alloc(&h, 100);
char *b = heap_alloc(&h, 200);
char *c = heap_alloc(&h, 300);
printf("a-base: %zu\n", (size_t)(a - (char *)h.start));  /* 0   */
printf("b-base: %zu\n", (size_t)(b - (char *)h.start));  /* 100 */
printf("c-base: %zu\n", (size_t)(c - (char *)h.start));  /* 300 */
printf("b-a:    %zu\n", (size_t)(b - a));                 /* 100 */
printf("c-b:    %zu\n", (size_t)(c - b));                 /* 200 */
```

Las diferencias entre punteros son _exactamente_ los tamaños de los bloques. Este comportamiento cambiará en el Post 2, donde cada bloque llevará un header de 8–16 bytes.


## Sección 5: Simplificaciones explícitas {#sección-5-simplificaciones-explícitas}

Este allocador tiene limitaciones serias. Las enumeramos explícitamente, porque cada una es un _feature del Post 1_ que se convierte en un _bugfix en un post posterior_:

**No hay freeing.** El bump pointer solo avanza. En el Post 3, añadiremos una **free list**: una lista enlazada de bloques liberados que `heap_alloc()` consulta antes de pedir más memoria al OS.

**No hay alineación.** Un alloc de 3 bytes seguido de un alloc de `double*` produce un puntero desalineado. En el Post 2, cada allocation se alineará a 8 o 16 bytes automáticamente.

**No hay metadata de bloques.** El allocador no sabe dónde empieza o termina cada bloque individual. En el Post 2, cada bloque llevará un **block header**: una pequeña struct al inicio que almacena el tamaño, el estado (libre/ocupado), y un puntero al siguiente bloque.

**El heap es contiguo.** `sbrk()` solo puede extender el heap hacia arriba. En el Post 7, migraremos a `mmap()`, que puede pedir regiones de memoria arbitrarias en cualquier parte del espacio de direcciones.

**Single-threaded.** No hay locks, no hay thread-safety. En el Post 7, reconoceremos este problema; la solución completa (arenas per-thread) queda fuera del alcance de esta serie, pero señalaremos cómo `jemalloc` y `mimalloc` lo resuelven.

| Limitación      | Estado en Post 1 | Se resuelve en         |
|-----------------|------------------|------------------------|
| No freeing      | Solo avanza      | Post 3 (free list)     |
| No alineación   | Offsets crudos   | Post 2 (block headers) |
| No metadata     | Bloques anónimos | Post 2 (block headers) |
| Heap contiguo   | Solo sbrk()      | Post 7 (mmap + arenas) |
| Single-threaded | Sin locks        | Post 7+ (reconocido)   |


## Sección 6: Hacia dónde vamos {#sección-6-hacia-dónde-vamos}

En el **Post 2**, añadiremos **block headers**. Cada allocation llevará una pequeña struct al inicio con el tamaño del bloque y flags de estado. Esto transforma el heap de un blob opaco en una secuencia de bloques autodescriptivos que podemos recorrer, inspeccionar, y eventualmente liberar.

En el **Post 3**, implementaremos `heap_free()` y una **free list**. Cuando un bloque se libera, se marca como libre y se inserta en una lista enlazada. Futuras allocaciones buscan primero en la free list antes de pedir más memoria al OS.

En el **Post 4**, abordaremos **coalescing y splitting**: cuando dos bloques libres son adyacentes, los fusionamos en uno grande; cuando un bloque libre es mucho mayor que lo pedido, lo dividimos.

El **Post 5** explora **políticas de allocation**, first-fit, best-fit, next-fit, y cómo cada una afecta la fragmentación y el rendimiento.

En el **Post 7** ocurre la transición grande: reemplazamos `sbrk()` por `mmap()`, refactorizamos `heap_t` en una `allocator_t` con múltiples arenas, y preparamos el terreno para thread-safety.

Los **Posts 8–12** construyen un **garbage collector** progresivamente: reference counting, mark-and-sweep, conservative GC, e incremental GC.

El **Post 13** es el capstone: copying collector, generational GC, y referencias a implementaciones de producción (Go GC, jemalloc, mimalloc).


## Conclusión {#conclusión}

Hoy construimos un allocador de memoria funcional en ~80 líneas de C. Pide memoria al sistema operativo con `sbrk()`, la reparte con un bump pointer, y la visualiza con un dump ASCII. Es incompleto, no puede liberar, no alinea, no lleva metadata, pero esas limitaciones son features pedagógicos, no bugs.

Lo importante no es el código en sí, sino lo que revela: que `malloc()` no es magia. Es una struct, un par de syscalls, y algo de contabilidad. El runtime de C que usas todos los días hace exactamente esto, más sofisticado, más optimizado, más robusto, pero la mecánica fundamental es la misma.

En el Post 2, damos el siguiente paso: **block headers y alineación**. El heap deja de ser un blob opaco y se convierte en una secuencia de bloques autodescriptivos. Es el primer paso hacia `free()`.


## Key Takeaways {#key-takeaways}

-   `sbrk()` es la syscall que subyace a `malloc()`: mueve el program break para pedir memoria contigua al OS.
-   Un bump allocator es la estrategia más simple: un puntero que solo avanza. No hay freeing, no hay fragmentación, no hay complejidad.
-   `heap_t` es una decisión arquitectónica: usar una struct en lugar de globales prepara el refactor a arenas en el Post 7.
-   Las limitaciones son deliberadas: cada simplificación (no freeing, no alineación, no metadata) se resuelve en un post posterior específico.
-   La memoria no liberada la reclama el OS al terminar: para programas cortos, no liberar es una estrategia legítima; para servidores, es un memory leak.
