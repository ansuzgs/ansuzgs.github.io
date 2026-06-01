---
title: "Sobre mí"
date: 2026-05-22
layout: "single"
ShowToc: false
ShowReadingTime: false
description: "Pablo González — ingeniero de telecomunicaciones y doctor, escribiendo sobre programación en C de bajo nivel, allocators, estructuras de datos y diseño de lenguajes."
---

Soy Pablo, ingeniero de telecomunicaciones con un doctorado en simulación
electromagnética. Actualmente trabajo en ingeniería de sistemas en el sector de
defensa y espacio (infraestructura de satélites y sistemas de misión).

Empecé con la programación de bajo nivel porque quería entender qué estaba pasando
de verdad. No la abstracción, no la API, la cosa en sí. Cómo se reparte la
memoria. Cómo decide un array dinámico cuándo crecer. Qué hace `free()` realmente
con esos bytes. Este blog es donde resuelvo esas preguntas construyendo las
respuestas desde cero, en C.

## Sobre qué escribo

Allocators, estructuras de datos, patrones OOP en C, y un lenguaje pequeño llamado
**Zinc** que estoy construyendo desde cero (intérprete, máquina virtual de
bytecode, y eventualmente un compilador) todo en C99. Ahora mismo hay dos series
activas: una construyendo un
[allocator de memoria y garbage collector](/es/series/memory-allocation-and-gc-from-scratch/),
otra implementando
[arrays dinámicos](/es/series/dynamic-arrays-in-c/) con todos los edge cases que
la mayoría de tutoriales se saltan.

## Proyectos

- **[Zinc](https://github.com/ansuzgs/zinc)**: un lenguaje minimal con tipado
  estático, orientado a un subconjunto seguro de la semántica de C. Ownership
  explícito, sin GC en runtime, salida en C legible como target de compilación.
- **[bitstream.h](https://github.com/ansuzgs/bitstream.h)**: librería C de un
  solo header para leer y escribir bitstreams empaquetados. Útil para parsers de
  protocolos y codecs de compresión.
- **[dynamic-arrays-c](https://github.com/ansuzgs/dynamic-arrays-c)**: repo
  companion de la serie de Arrays Dinámicos. Cada post compila y ejecuta.

## El código

Todo lo que aparece en este blog compila y está testeado. Si no compila, es un
bug, abre un issue en [GitHub](https://github.com/ansuzgs).

## En otro sitio

Cuando no estoy escribiendo C suelo estar encima de una bici de carretera, en un
rocódromo, o en algún sitio en la montaña con una cámara.

GitHub: [@ansuzgs](https://github.com/ansuzgs)
LinkedIn: [pablogonzalez21191](https://www.linkedin.com/in/pablogonzalez21191/)
