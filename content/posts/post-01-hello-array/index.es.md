+++
title = "Hello, Array: malloc, free y la Gestión Manual de Metadatos"
author = ["pablo"]
date = 2026-05-24
draft = false
translationKey = "post-01-hello-array"
series = ["Arrays Dinámicos en C"]
+++

## El Problema del que Nadie Habla al Empezar {#el-problema-del-que-nadie-habla-al-empezar}

Tienes cinco enteros. Los metes en un array:

```c
int numeros[5] = { 10, 20, 30, 40, 50 };
```

Listo. C te da un bloque contiguo de 20 bytes en la pila indexado del 0 al 4, y todo va bien.

Ahora tu usuario quiere añadir un sexto entero. ¿Qué haces?

No puedes redimensionar un array en la pila. Su tamaño quedó grabado en el binario en tiempo de compilación: el compilador vio `5`, calculó 20 bytes, y ese es el espacio que tiene el stack frame de tu función. No hay negociación posible. Podrías declarar `int numeros[1000]` y confiar en que sea suficiente, pero la esperanza no es una estrategia de gestión de memoria.

Podrías usar un array de longitud variable (`int numeros[n]`), pero eso solo desplaza el problema: `n` sigue siendo fijo en cuanto entras en la función. Peor aún, los VLAs viven en la pila, que está limitada a unos pocos megabytes. Almacena un millón de enteros y volarás la pila sin ninguna recuperación elegante.

La solución real vive en el heap. `malloc` te permite pedirle al sistema operativo un bloque de memoria en tiempo de ejecución, del tamaño que quieras, limitado solo por la RAM disponible. Pero malloc te da bytes en bruto y un puntero. Sin seguimiento del tamaño. Sin comprobación de límites. Sin "¿cuánto espacio tengo ocupado?". Obtienes la memoria y la responsabilidad.

Aquí es donde empieza todo array dinámico: no con una estructura de datos ingeniosa, sino con una pregunta básica de contabilidad. ¿Quién controla cuántos elementos has almacenado? ¿Quién controla cuántos podrías almacenar? ¿Quién se asegura de que la memoria se libera cuando terminas?

En C, la respuesta siempre es la misma: tú.

Este artículo construye el array dinámico más sencillo posible, uno que almacena enteros, tiene capacidad fija, y hace exactamente tres cosas: crear, insertar y destruir. Sin crecimiento automático (eso es el Post 2), sin genéricos (Post 4), sin recuperación de errores (Post 6). Solo el esqueleto básico sobre el que todo lo demás se construye.

Al final, entenderás dos cosas que la mayoría de tutoriales de C se saltan: por qué existe el struct de metadatos, y por qué importa el orden en que llamas a `free`.

Vamos a reservar memoria.


## El Struct: Lo que un Array Sabe de Sí Mismo {#el-struct-lo-que-un-array-sabe-de-sí-mismo}

Una llamada desnuda a `malloc` devuelve `void *`, un puntero a bytes sin ningún significado asociado. Si reservas espacio para 10 enteros, nadie recuerda ese número excepto tú. En el momento en que pierdes la pista de la capacidad, estás introduciendo bugs.

Así que lo primero que necesita un array dinámico no son los datos. Son los _metadatos_: un pequeño struct que vive junto a los datos y recuerda los detalles de contabilidad.

```c
typedef struct {
    int    *data;       /* Puntero al buffer en el heap que contiene los elementos */
    size_t  size;       /* Cuántos elementos se han almacenado                     */
    size_t  capacity;   /* Cuántos elementos puede albergar el buffer              */
} IntArray;
```

Tres campos. Esta es la contabilidad mínima viable.

`data` es un puntero a la reserva real en el heap donde viven los elementos. Es el resultado de una llamada a `malloc(capacity * sizeof(int))`. Cuando accedes a `arr->data[3]`, estás leyendo el cuarto entero de esa reserva.

`size` controla cuántos elementos ha insertado el usuario. Empieza en 0 y se incrementa con cada `array_push`. No es lo mismo que capacity: esta distinción es el concepto más importante en el diseño de arrays dinámicos.

`capacity` controla cuántos elementos puede _albergar_ la reserva. Si hiciste malloc para 10 enteros, capacity es 10, aunque size sea solo 3. La diferencia entre size y capacity es memoria desperdiciada: la reservamos pero aún no la usamos. Gestionar esa diferencia es el arte de los arrays dinámicos.

Piénsalo como un aparcamiento. `capacity` es el número de plazas. `size` es el número de coches aparcados en este momento. El aparcamiento existe en una dirección concreta (`data`). Puedes tener un aparcamiento vacío (size=0, capacity=100) o uno lleno (size=100, capacity=100), pero nunca puedes aparcar más coches que plazas, a menos que construyas uno más grande.


## El Código Completo {#el-código-completo}

Aquí está el fichero fuente completo y autocontenido. Compila sin advertencias bajo `gcc -Wall -Wextra -Wpedantic -std=c11`, se ejecuta sin fugas, y genera tanto una visualización ASCII en stdout como un fichero Graphviz DOT.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
/* --- El struct ----------------------------------------------------------- */
typedef struct {
    int    *data;
    size_t  size;
    size_t  capacity;
} IntArray;
/* --- Ciclo de vida ------------------------------------------------------- */
IntArray *array_create(size_t capacity)
{
    if (capacity == 0) {
        fprintf(stderr, "array_create: capacity debe ser > 0\n");
        return NULL;
    }
    IntArray *arr = malloc(sizeof(IntArray));
    if (!arr) {
        fprintf(stderr, "array_create: fallo al reservar el struct\n");
        return NULL;
    }
    arr->data = malloc(capacity * sizeof(int));
    if (!arr->data) {
        fprintf(stderr, "array_create: fallo al reservar el buffer\n");
        free(arr);
        return NULL;
    }
    arr->size     = 0;
    arr->capacity = capacity;
    return arr;
}
void array_destroy(IntArray *arr)
{
    if (!arr) return;
    free(arr->data);
    arr->data = NULL;
    free(arr);
}
/* --- Operaciones --------------------------------------------------------- */
int array_push(IntArray *arr, int value)
{
    if (!arr) return -1;
    if (arr->size >= arr->capacity) {
        fprintf(stderr,
                "array_push: lleno (size=%zu, capacity=%zu)\n",
                arr->size, arr->capacity);
        return -1;
    }
    arr->data[arr->size] = value;
    arr->size++;
    return 0;
}
int array_get(const IntArray *arr, size_t index, int *out)
{
    if (!arr || index >= arr->size) return -1;
    *out = arr->data[index];
    return 0;
}
size_t array_size(const IntArray *arr)     { return arr ? arr->size     : 0; }
size_t array_capacity(const IntArray *arr) { return arr ? arr->capacity : 0; }
```


### Las Funciones de Visualización {#las-funciones-de-visualización}

El visualizador ASCII imprime una instantánea completa del estado del array: la disposición de la memoria de los metadatos con dibujo de cajas, etiquetas de índice y estadísticas de utilización:

```c
void array_visualize_ascii(const IntArray *arr, const char *label)
{
    if (!arr) { printf("(array NULL)\n"); return; }
    size_t cap = arr->capacity;
    size_t sz  = arr->size;
    printf("\n╔══════════════════════════════════════════════════════╗\n");
    printf("║  %-50s  ║\n", label ? label : "ESTADO DEL ARRAY");
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║  size = %-5zu  capacity = %-5zu  elem = %zu bytes    ║\n",
           sz, cap, sizeof(int));
    printf("║  data = %-14p  (heap)                   ║\n",
           (void *)arr->data);
    printf("╠══════════════════════════════════════════════════════╣\n");
    size_t show = cap <= 16 ? cap : 16;
    /* Borde superior */
    printf("║  ");
    for (size_t i = 0; i < show; i++) printf("┌──────");
    printf("┐  ║\n");
    /* Valores */
    printf("║  ");
    for (size_t i = 0; i < show; i++) {
        if (i < sz) printf("│%5d ", arr->data[i]);
        else        printf("│  ·   ");
    }
    printf("│  ║\n");
    /* Borde inferior */
    printf("║  ");
    for (size_t i = 0; i < show; i++) printf("└──────");
    printf("┘  ║\n");
    /* Etiquetas de índice */
    printf("║  ");
    for (size_t i = 0; i < show; i++) printf(" %3zu   ", i);
    printf("   ║\n");
    /* Estadísticas */
    size_t used_bytes  = sz  * sizeof(int);
    size_t alloc_bytes = cap * sizeof(int);
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║  %zuB usados / %zuB reservados = %.1f%% utilización\n",
           used_bytes, alloc_bytes,
           cap > 0 ? 100.0 * (double)sz / (double)cap : 0.0);
    printf("╚══════════════════════════════════════════════════════╝\n\n");
}
```

El generador DOT escribe un fichero Graphviz que muestra la relación estructural entre el struct de metadatos y el buffer en el heap:

```c
void array_generate_dot(const IntArray *arr, const char *filename)
{
    FILE *f = fopen(filename, "w");
    if (!f) return;
    fprintf(f, "digraph IntArray {\n");
    fprintf(f, "  rankdir=LR;\n");
    /* Nodo de metadatos */
    fprintf(f, "  metadata [shape=record, style=filled, "
               "fillcolor=\"#FFF3CD\",\n");
    fprintf(f, "    label=\"{IntArray|data: %p|size: %zu|"
               "capacity: %zu}\"];\n",
            (void *)arr->data, arr->size, arr->capacity);
    /* Nodo del buffer */
    fprintf(f, "  buffer [shape=record, style=filled, "
               "fillcolor=\"#D1ECF1\",\n");
    fprintf(f, "    label=\"{Buffer en Heap (%zu bytes)|{",
            arr->capacity * sizeof(int));
    for (size_t i = 0; i < arr->capacity; i++) {
        if (i > 0) fprintf(f, "|");
        if (i < arr->size)
            fprintf(f, "[%zu]=%d", i, arr->data[i]);
        else
            fprintf(f, "[%zu]=·", i);
    }
    fprintf(f, "}}\"];\n");
    fprintf(f, "  metadata -> buffer [label=\"posee (heap)\"];\n");
    fprintf(f, "}\n");
    fclose(f);
}
```


### La Función Main {#la-función-main}

`main()` dirige la demostración: crear un array, insertar elementos uno a uno (visualizando tras cada inserción), llenarlo hasta la capacidad, intentar superarla, generar el fichero DOT y destruirlo:

```c
int main(void)
{
    IntArray *arr = array_create(5);
    /* Insertar valores de uno en uno, visualizar tras cada uno */
    int valores[] = {10, 20, 30, 40, 50};
    for (int i = 0; i < 5; i++) {
        array_push(arr, valores[i]);
        array_visualize_ascii(arr, "Tras push");
    }
    /* Esto fallará — el array está lleno */
    int rc = array_push(arr, 60);  /* devuelve -1 */
    /* Generar Graphviz y limpiar */
    array_generate_dot(arr, "outputs/post_01_estado.dot");
    array_destroy(arr);
    return 0;
}
```


## Análisis del Código {#análisis-del-código}


### Dos Reservas, Dos Liberaciones {#dos-reservas-dos-liberaciones}

El patrón más importante de todo este fichero es la simetría entre `array_create` y `array_destroy`.

`array_create` realiza dos reservas:

```text
malloc(sizeof(IntArray))        -> el struct de metadatos
malloc(capacity * sizeof(int))  -> el buffer de elementos
```

`array_destroy` realiza dos liberaciones, en orden inverso:

```text
free(arr->data)   -> primero el buffer de elementos
free(arr)         -> después el struct de metadatos
```

El orden no es negociable. Si lo inviertes, haciendo `free(arr)` primero y luego `free(arr->data)`, estarás desreferenciando `arr` después de haberlo liberado. La memoria en `arr` ha sido devuelta al asignador. Leer `arr->data` en ese punto es comportamiento indefinido: puede que funcione, puede que se cuelgue, puede que corrompa silenciosamente el heap. Valgrind lo marcaría como una lectura inválida.

El `arr->data = NULL` defensivo tras el primer free es opcional pero barato. Garantiza que si algo toca accidentalmente el struct entre las dos liberaciones (en código más complejo con callbacks o gestión de errores), el acceso inválido produzca una desreferencia de NULL, un crash ruidoso y depurable, en lugar de un use-after-free silencioso.


### Por Qué Push Comprueba `size >= capacity` {#por-qué-push-comprueba-size-capacity}

La cláusula de guarda de la función push es sencilla:

```c
if (arr->size >= arr->capacity) return -1;
```

Aquí es donde los metadatos se pagan solos. Sin el campo `capacity`, no habría forma de saber si `arr->data[arr->size]` es una escritura válida o un desbordamiento de buffer. La comprobación cuesta una comparación por inserción, unos pocos nanosegundos, y previene la clase más común de bugs de corrupción de heap.

Fíjate en que push no hace crecer el array cuando está lleno. Simplemente rechaza la operación y devuelve -1. Esta es una decisión de diseño deliberada para el Post 1: primero construimos la comprensión del caso estático. El Post 2 introduce `realloc` y el crecimiento automático. Si te mueres de ganas por hacer crecer el array, esa frustración es pedagógicamente intencionada: estás sintiendo el mismo dolor que motiva el crecimiento dinámico.


### La Visualización como Herramienta de Depuración {#la-visualización-como-herramienta-de-depuración}

`array_visualize_ascii` no es solo un impresor bonito para el blog. Es una herramienta de depuración que puedes incrustar en cualquier proyecto. Cada vez que el estado del array cambia, puedes imprimirlo:

<pre style="width: fit-content; margin: 0 auto;">
╔════════════════════════════════════════════════════╗
║  Tras push(30)                                     ║
╠════════════════════════════════════════════════════╣
║  size = 3      capacity = 5      elem = 4 bytes    ║
╠════════════════════════════════════════════════════╣
║  ┌──────┌──────┌──────┌──────┌──────┐              ║
║  │   10 │   20 │   30 │  ·   │  ·   │              ║
║  └──────└──────└──────└──────└──────┘              ║
║     0      1      2      3      4                  ║
╚════════════════════════════════════════════════════╝
</pre>

Tres enteros almacenados, dos huecos aún vacíos (mostrados como `·`). Los metadatos en la parte superior te dicen todo: size, capacity, dirección del puntero, tamaño del elemento. Las estadísticas al final cuantifican el desperdicio. Puedes redirigir la salida a un fichero (`./post_01 > traza.txt`) y recorrer todo el ciclo de vida de tu array.

Esto es algo que la mayoría de tutoriales no te dan: la capacidad de ver la memoria. Construiremos sobre esta visualización en cada artículo, añadiendo diagramas Graphviz para las relaciones entre punteros, y eventualmente un dashboard HTML interactivo para el análisis de rendimiento.


## Conceptos Clave y Compromisos {#conceptos-clave-y-compromisos}


### Pila vs Heap para el Struct de Metadatos {#pila-vs-heap-para-el-struct-de-metadatos}

Nuestro `array_create` reserva el struct IntArray en el heap:

```c
IntArray *arr = malloc(sizeof(IntArray));
```

¿Por qué no ponerlo en la pila? Podrías escribir:

```c
IntArray array_create_stack(size_t capacity) {
    IntArray arr;
    arr.data = malloc(capacity * sizeof(int));
    arr.size = 0;
    arr.capacity = capacity;
    return arr;  /* devuelve una copia */
}
```

Funciona, pero cambia el contrato de la API de formas sutiles. El llamador recibe una copia del struct. Si la pasa a una función que la modifica (por ejemplo, `push`), necesita pasar un puntero a su copia local. Y no puede devolverla desde una función que la crea condicionalmente, porque el stack frame desaparece. La reserva en el heap te da un puntero estable que sobrevive a los límites de función y puede almacenarse en otras estructuras de datos. El compromiso es que ahora necesitas liberarlo explícitamente: la memoria en el heap nunca se limpia sola.

Para una API de biblioteca, la reserva en el heap es la elección estándar. Le da a los llamadores un único puntero a gestionar, y la función destroy se encarga de la limpieza interna. Verás este patrón en prácticamente todas las bibliotecas C: `cosa_create()` devuelve un puntero, `cosa_destroy()` lo libera.


### Memoria Desperdiciada: El Problema de la Capacidad {#memoria-desperdiciada-el-problema-de-la-capacidad}

Cuando creas un array con capacidad 10 y almacenas 3 elementos, estás desperdiciando 7 huecos × 4 bytes = 28 bytes. Eso es un 70% de desperdicio. ¿Es malo?

Depende. Para arrays pequeños, el desperdicio es insignificante: 28 bytes no son nada en una máquina moderna con gigabytes de RAM. Para millones de arrays pequeños, se acumula. Para un array grande, el porcentaje de desperdicio cae a medida que se llena.

La pregunta real es: ¿con qué capacidad deberías empezar? Si eliges demasiado pequeña (capacity=1), necesitarás reasignar constantemente a medida que el array crece (Post 2). Si eliges demasiado grande (capacity=10000), desperdicias memoria en arrays que solo almacenan 5 elementos. No hay una respuesta universal: depende de tu patrón de uso. Por eso los arrays dinámicos de producción te permiten especificar una capacidad inicial como sugerencia.


### Por Qué Códigos de Retorno y No Aserciones {#por-qué-códigos-de-retorno-y-no-aserciones}

`array_push` devuelve 0 en caso de éxito y -1 en caso de fallo. No aborta el programa. Esta es una decisión de diseño consciente: el llamador debe decidir qué hacer cuando una inserción falla. Quizás quiera registrar el error y continuar. Quizás quiera redimensionar y reintentar. Quizás quiera salir. Al devolver un código de error, le damos esa elección.

La alternativa, `assert(arr->size < arr->capacity)`, mata el programa sin recuperación posible. Eso es apropiado para errores del programador (como pasar NULL donde no deberías), pero no para condiciones esperadas como "el array está lleno." Discutiremos estrategias de manejo de errores en profundidad en el Post 6.


## Prueba Esto y Mira Cómo Falla {#prueba-esto-y-mira-cómo-falla}

Antes de continuar, prueba estos experimentos con el código:

**Experimento 1: La Fuga de Memoria**. En `array_destroy`, comenta `free(arr->data)`. Compila y ejecuta bajo valgrind (`valgrind --leak-check=full ./post_01`). Verás "definitely lost: 20 bytes": ese es el buffer huérfano. El struct fue liberado, pero el buffer al que apuntaba no. Este es exactamente el bug que pregunta la prueba de conocimiento.

**Experimento 2: Uso Tras Liberación**. En `main`, añade `printf("%d\n", arr->data[0]);` después de `array_destroy(arr)`. Compila y ejecuta. Puede que imprima `10`. Puede que imprima basura. Puede que se cuelgue. Eso es comportamiento indefinido: los datos están liberados, pero la memoria no ha sido necesariamente sobreescrita todavía. Valgrind marcaría esto como "Invalid read of size 4."

**Experimento 3: Desbordamiento de Buffer**. Cambia la guarda de push para que siempre tenga éxito (elimina la comprobación de capacidad). Inserta 100 elementos en un array de capacidad 5. Ejecuta bajo AddressSanitizer (`gcc -fsanitize=address`). Mira cómo detecta el heap-buffer-overflow.


## Prueba de Conocimiento {#prueba-de-conocimiento}

> ¿Qué ocurre si llamas a `free(arr)` pero olvidas llamar antes a `free(arr->data)`?

El struct se devuelve al asignador del heap, pero el buffer al que apuntaba, los `capacity * sizeof(int)` bytes en `arr->data`, permanece reservado. Ya no existe ningún puntero hacia él (el struct que tenía el puntero está liberado), así que la memoria se pierde. Nunca será liberada durante el resto de la vida del programa. En un programa de larga ejecución, fugas repetidas como esta se acumulan y eventualmente agotan la memoria. Valgrind reportaría "definitely lost: N bytes in 1 blocks."


## Qué Viene Después {#qué-viene-después}

Nuestro array funciona, pero tiene una limitación paralizante: cuando está lleno, push falla. El usuario tiene que adivinar la capacidad correcta de antemano, y si se equivoca, está atascado.

En el **Post 2: "Dolores de Crecimiento: realloc y Gestión Automática de Capacidad"**, eliminamos esta limitación. Introduciremos `realloc`, la llamada que dice "dame más espacio, y copia mis datos a la nueva ubicación si hace falta." Aprenderás por qué los punteros antiguos se invalidan tras un realloc, por qué el factor de crecimiento importa (adelanto: determina tu coste amortizado), y por qué nunca debes escribir `arr->data = realloc(arr->data, new_size)` directamente.

El array de capacidad fija que has construido hoy es la base. Todo lo que viene a partir de aquí se construye sobre él.
