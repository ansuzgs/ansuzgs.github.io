---
title: "Hola Mundo: Punteros, Memoria y C a Bajo Nivel"
date: 2026-05-22
draft: false
tags: ["C", "blog", "pablogs.dev"]
---

Todo proyecto en C empieza con un `printf("Hello, World!\n");`, y este blog no iba a ser diferente. Bienvenidos a **pablogs.dev**.

Llevo un tiempo queriendo crear un espacio propio para documentar mis proyectos, organizar mis ideas y, sobre todo, compartir lo que voy aprendiendo sobre programación de sistemas y C a bajo nivel. A menudo, cuando programamos en lenguajes de alto nivel, damos por sentada la magia que ocurre por debajo: cómo se asigna la memoria, cómo se limpian los recursos o cómo crecen las estructuras de datos.

Mi objetivo con este blog es quitarle el velo a esa magia e implementar estas herramientas desde cero. 

**¿Qué vas a encontrar aquí?**

En las próximas semanas y meses, voy a publicar **series de artículos** donde ensuciaremos nuestras manos con código C puro. Algunas de las cosas que tengo preparadas en la hoja de ruta son:

*   **Estructuras dinámicas:** Cómo implementar tus propios *dynamic arrays* y listas enlazadas en C sin morir en el intento.
*   **Gestión de memoria:** Una serie completa escribiendo un *Custom Allocator* (implementando nuestras propias versiones de `malloc` y `free`).
*   **Garbage Collectors:** Sí, vamos a escribir un recolector de basura para C, explorando cómo rastrear punteros y limpiar el *heap* automáticamente.
*   **Reflexiones y trucos:** Configuraciones de mi entorno, Makefiles, y pequeños proyectos paralelos.

**¿Por qué bilingüe?**

He decidido montar el blog en Hugo configurado para soportar tanto **Inglés** como **Castellano**. Quiero que este conocimiento sea accesible para la comunidad hispanohablante, pero también quiero compartir estos proyectos con la comunidad global. Puedes cambiar de idioma en cualquier momento usando el botón de la cabecera.

Prepara tu compilador favorito, ten cuidado con los *Segmentation Faults*, y nos vemos en el próximo post.
