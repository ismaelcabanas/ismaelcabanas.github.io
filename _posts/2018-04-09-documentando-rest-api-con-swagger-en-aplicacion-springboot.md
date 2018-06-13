---
layout: post
title: "Documentando un Spring Boot RESTFul API con Swagger 2"
description: ""
category:
tags: [springboot, swagger, REST, API]
---
{% include JB/setup %}

Cuando tenemos en mente crear un proyecto en el que exponemos un API queremos que alguien
o nosotros mismos sepamos cómo utilizar ese API, en este punto, nos tenemos que plantear
en tener una buena documentación de qué hace nuestro API y cómo utilizarla.

Documentar un API debería estar estructurada para que fuese informativa, fácil de leer y concisa.

En este artículo intentaré cubrir con un ejemplo cómo generar una documentación de una API con Swagger 2
en un proyecto Spring Boot 2.

## Swagger 2 en Spring Boot

Swagger 2 es un proyecto open source que se usa para describir y documentar APIs Restful. Swagger 2 es agnóstico del lenguaje y es extensible en nuevas tecnologías y protocolos más allá de HTTP. La versión actual define un conjunto de elementos HTML, JavaScript y CSS para generar documentación dinámicamente desde un API Swagger. Estos ficheros están incluidos por el proyecto Swageer UI para mostrar el API en un navegador. Además de mostrar documentación, Swagger UI permite a otros desarrolladores de API interactuar con los recursos del API sin tener lógica de implementation.

La especificación Swagger 2, conocida como OpenAPI specification tiene varias implementaciones. Actualemente,  Springfox es la implementación más popular para las aplicaciones Spring Boot. Springfox soporta Swagger 1.2 and 2.0.

Usaremos Springfox en nuestro proyecto.

Para usarlo en nuestro proyecto Spring Boot debemos añadir la dependencia de Springfox

```compile("io.springfox:springfox-swagger2:2.8.0")```

Y como también usaremos Swagger UI, incluiremos la dependencia

```compile("io.springfox:springfox-swagger-ui:2.8.0")```
