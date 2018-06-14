---
layout: post
title: "Desarrollo de una aplicación desde 0 (Parte 1)"
description: ""
category:
tags: []
---
{% include JB/setup %}

En esta serie de artículos voy a desarrollar una aplicación REST API desde cero bajo mi criterio, después de ya varios años de experiencia.

Una de mis obsesiones a nivel profesional es tener un modo de desarrollar software con calidad, que sea ágil de construir y que siga una arquitectura flexible y adecuada en cada caso.

De esto tratará esta serie de artículos, de encontrar y exponer un modo de hacer las cosas, que no es ni mejor ni peor que otros, pero con el que, habiendome documentado durante estos años, yo me sienta a gusto con el trabajo realizado.

## Integración y despliegue contínuo (CI/CD) 

Una de mis premisas a la hora de desarrollar software es que éste sea de calidad, que tenga una buena red de tests que verifiquen el comportamiento de mi sistema y que esta red de tests me permita realizar refactorizaciones de código sin modificar el comportamiento de la aplicación. Me gusta refactorizar, qué le vamos a hacer, siempre estoy en contínuo aprendizaje y lo que hoy me parece que está bien, mañana me parece que está regular...

Pero otro aspecto importante cuando desarrollas software es que tengas una infraestructura ágil y rápida para realizar builds y despliegues automáticos del sistema, de forma que se puedan desplegar automáticamente nuevas features para que los usuarios den feedback rápido de estas features. Y para ello, lo ideal es que antes de ponernos a desarrollar features como locos paremos, y pensemos bien qué herramientas se van a utilizar para realizar estas tareas de integración y despliegue contínuo.

Básicamente, el proceso de CI/CD se desencadenaría cada vez que se realiza un **push** a master en nuestro repositorio de código fuente, y las tareas más importantes que, a mi juicio, debería realizar este proceso de CI/CD serían:
  - realizar el build del proyecto
  - pasar los tests unitarios y de integración
  - empaquetar el artefacto del proyecto
  - desplegar el artefacto en alguna máquina
  - pasar los tests end-to-end o de aceptación

### Tests end-to-end

En cuanto a los tests end-to-end, a veces son los tests más difíciles de realizar, pero si es posible deberíamos intentar hacerlos, puesto que son los que van a marcar que nuestra aplicación debe hacer lo que se espera que haga, son los que nos van a cubrir cuando queramos abordar refactorizaciones y confiar en que todo siga funcionando bien. Vuelvo a repetir que son difíciles de hacer, pero son bastante importantes a mi parecer.

Entonces, lo ideal sería que, a la vez que pensamos y seleccionamos las herramientas necesarias para hacer CI/CD, deberíamos pensar e intentar hacer un test end-to-end de la feature más pequeña que fuésemos a desarrollar para, desde un primer momento, en el comienzo de nuestro proyecto, todo el engranaje de automatizar los procesos de CI/CD funcionen y seamos capaces de implementar y desplegar una mínima feature.

## Manos a la obra

Pues vamos al lío, voy a intentar desarrollar un proyecto personal, un API REST desarrollado con una aplicación Spring Boot que tratará sobre gestionar la despensa particular de una casa.

### Travis CI

Como herramienta de intración contínua (CI) he elegido Travis CI, entre otras cosas, por su buena integración con GitHub que es el repositorio de código que utilizo para mis proyectos personales. Esta herramienta me va a permitir:
 - Hacer checkout del último código de mi repositorio de código fuente
 - Compilar y ejecutar test unitarios y de integración
 - Ejecutar una herramienta de cobertura de código como JaCoCo para que la build falle si no supera un determinado porcentaje
 - Ejecutar una herramienta que compruebe la calidad del código, como SonarQube
 - Construir una imagen Docker y subirla de DockerHub
 - Desplegar la aplicación en algún servicio Cloud como Amazon AWS, Heroku u OpenShift

### Paso 1: Crear una aplicación SpringBoot

Lo primero es crear la aplicación SpringBoot, en mi caso parto de una aplicación Spring Boot 2 gestionada con  Gradle, crear el mínimo test de aceptación (end-to-end) comprobando que el API está levantada, y subir el código al repositorio de código.

### Paso 2: Crear el fichero .travis.yml

Para poder habilitar CI con Travis necesitamos crear el fichero **.travis.yml** en la raíz de nuestro proyecto.

La configuración siguiente del fichero .travis.yml es suficiente para nuestro proyecto Gradle:

**.travis.yml**
```
language: java
jdk:
  - oraclejdk8
```

Al ser un proyecto Gradle, por defecto Travis ejecutar **./gradlew check**. Pero, si queremos ejecutar un comando diferente o queremos ejecutar varios comandos, podemos usar el bloque **script** para personalizarlo.

Por último, deberíamos comitear y pushear a Github.

### Paso 3: Habilitar Travis-CI en Github

Vamos a Travis-CI y nos autenticamos con la cuenta de GitHub. Entramos en Travis-CI y sincronizamos el proyecto recién creado y habilitamos Travis. De este modo, cada vez que se haga un **push** a GitHub saltará el Travis-CI y se enviará una notificación con el estado del proceso de build.

### Paso 4: Añadir cobertura de código con JaCoCo

En este paso añadimos la cobertura de código con el plugin de JaCoCo con alguna opción como el mínimo porcentaje de cobertura de código que creamos aceptable, exclusión de clases o paquetes que no deseemos que entren en la análisis de la cobertura de código, etc...

```
apply plugin: 'jacoco'

jacoco {
    toolVersion = "0.7.7.201606060606"
}

jacocoTestReport {
    group = "Reporting"
    executionData tasks.withType(Test)
    reports {
        xml.enabled true
        csv.enabled false
        html.destination file ("${buildDir}/reports/coverage")
    }
    afterEvaluate {
        classDirectories = files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                    '**/*Application**'
            ])
        })
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.7 // our tests should cover minimun of 70%, if violate the rule the build will fail
            }
        }
    }
}

test {
    jacoco {
        append = false
        excludes = ['*Application*']
    }
}

check.dependsOn jacocoTestCoverageVerification
```

### Paso 5: Añadir al build tests unitarios, de integración y de aceptación

El código fuente de los tests de integración deberíamos colocarlos en un directorio separado, por ejemplo **integration-test**. A continuación la configuración básica para tests de integración en gradle

```
configurations {
    integrationtestCompile.extendsFrom testCompile
    integrationtestRuntime.extendsFrom testRuntime
}
sourceSets {
    integrationtest {
        java {
            compileClasspath += main.output + test.output
            runtimeClasspath += main.output + test.output
            // Use "java" if you don't use Scala as a programming language
            srcDir file('src/integration-test/java')
        }
        resources.srcDir file('src/integration-test/resources')
    }
}
task integrationTest(type: Test) {
    testClassesDir = sourceSets.integrationtest.output.classesDir
    classpath = sourceSets.integrationtest.runtimeClasspath
    // This is not needed, but I like to see which tests have run
    testLogging {
        events "passed", "skipped", "failed"
    }
}

integrationTest.mustRunAfter test
```

El código fuente de los tests de aceptación deberíamos colocarlos en un directorio separado, por ejemplo **acceptance-test**.La configuración básica para tests de aceptación con gradle

```
configurations {
    acceptancetestCompile.extendsFrom testCompile
    acceptancetestRuntime.extendsFrom testRuntime
}
sourceSets {
    acceptancetest {
        java {
            compileClasspath += main.output + test.output
            runtimeClasspath += main.output + test.output
            // Use "java" if you don't use Scala as a programming language
            srcDir file('src/acceptance-test/java')
        }
        resources.srcDir file('src/acceptance-test/resources')
    }
}
task acceptanceTest(type: Test) {
    testClassesDir = sourceSets.acceptancetest.output.classesDir
    classpath = sourceSets.acceptancetest.runtimeClasspath
    // This is not needed, but I like to see which tests have run
    testLogging {
        events "passed", "skipped", "failed"
    }
}

dependencies {
    // Testing
    // Karate acceptance test tool
    testCompile 'com.intuit.karate:karate-junit4:0.7.0'
    testCompile 'com.intuit.karate:karate-apache:0.7.0'
}

acceptanceTest.mustRunAfter integrationTest
```

### Paso 6: Verificar la calidad de código con SonarQube utilizando SonarCloud

[SonarCloud](https://sonarcloud.io) , que está construído sobre SonarQube, ofrece la comprobación de calidad de código gratuita para proyectos Open Source y puedes crear una cuenta con tu cuenta de GitHub.

Una vez logados en SonarCloud, creamos una nueva organización o si ya tenemos una analizamos un nuevo proyecto y generamos un token nuevo para el proyecto. Este token nos valdrá para incluirlo en Travis CI.

Travis CI nos proporciona una utilidad para encriptar [información sensible](https://docs.travis-ci.com/user/encryption-keys/) para que podamos encriptar cualquier clave para posteriormente configurarla en el fichero .travis.yml.

Instalamos travis con el comando

> sudo gem install travis

Y desde el directorio raíz del proyecto ejecutamos el siguiente comando para encriptar un valor como secreto:

> travis encrypt SOMEVAR=”secretvalue”

Este comando generará una salida similar a

> secure: “…. encrypted data ….”

Ahora añadimos este valor secreto encriptado como variable de entorno global en el fichero .travis.yml:

```
env:
  global:
  - secure: "....encrypted data....."
```

Ahora encriptamos el token generado en SonarCloud:

> travis encrypt SONAR_TOKEN=”my-sonar-token-here”

añadimos en Travis el [AddOn de SonarCloud](https://docs.travis-ci.com/user/sonarcloud/):

```
addons:
  sonarcloud:
    organization: "...your-organization..."
    token:
      secure: "....encrypted token....."

script:
  - "./gradlew check test integrationTest jacocoTestCoverageVerification jacocoTestReport --stacktrace"
  - sonar-scanner

```
Y por último, creamos en nuestro proyecto el fichero sonar-project.properties

```
# must be unique in a given SonarQube instance
sonar.projectKey=<your-project-key>

# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
# This property is optional if sonar.modules is set.
sonar.sources=src/main/java

# List of the module identifiers separated by ,
sonar.modules=<your-module>

# Encoding of the source code. Default is default system encoding
#sonar.sourceEncoding=UTF-8

sonar.java.binaries=build/classes/java/main

# Path to the JaCoCo report files containing coverage data by unit tests. The path may be absolute or relative to the project base directory.
sonar.jacoco.reportPaths=build/jacoco/acceptanceTest.exec
```

### Paso 7: Construir la imagen Docker

Los builds de Travis CI pueden ejecutar y construir imágenes Docker. Para más información visitar [https://docs.travis-ci.com/user/docker/](https://docs.travis-ci.com/user/docker/)

Crear un Dockerfile en la raíz del proyecto para nuestra aplicación:

```
FROM anapsix/alpine-java:8_server-jre_unlimited

VOLUME /tmp

ADD build/libs/<your_project>.jar app.jar

RUN sh -c 'touch /app.jar'

ENV JAVA_OPTS="-Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8787,suspend=n"

EXPOSE 8000 8787

ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]

```

Para usar docker en Travis CI añade la siguiente configuración a .travis.yml:

```
sudo: required

services:
  - docker
```

Vamos a configurar Travis para que después que se haya ejecutado la tarea de SonarQube, y ésta se haya ejecutado satisfactoriamente, construir la imagen Docker de nuestro artefacto para poder usarla en pasos posteriores.

Para ello, añadimos en la sección *env* y *script* la siguiente configuración:

```
env:
  ....
  - DOCKER_IMAGE_NAME = <YOUR_IMAGE_NAME>
script:
  ....
  - docker build -t $DOCKER_IMAGE_NAME .

```

### Paso 8: Ejecutar tests end-to-end (e2e)

Una vez construída nuestra imagen Docker, sería interesante poder ejecutar contra ella nuestros e2e tests o tests de aceptación. Tendremos que ejecutar la imagen generada en el paso anterior y lanzar la tarea gradle que ejecuta los tests de aceptación.

Para ello, añadimos en la sección *script* la siguiente configuración:

```
script:
  ....
  - docker run -it -p 8000:8000 -d $DOCKER_IMAGE_NAME
  - ./gradlew acceptanceTest  
```

***Nota***: *el punto que no me gusta en esta configuración es indicar en la ejecución de la imagen de Docker el puerto en concreto por el que se comunicará la máquina de Travis con el contenedor Docker. Ese puerto debe coincidir con el puerto en el que se ejecutan los e2e tests. Podría considerarse como una convención, pero seguro que hay alguna forma más elegante de realizar esta acción.*

### Paso 9: Subir la imagen Docker a DockerHub

Una vez hayan pasado los e2e tests sobre la imagen Docker de nuestra aplicación, es hora de subir esta imagen a un repositorio de imágenes Docker, como DockerHub.

Para poder subir la imagen a DockerHub hay que establecer las credenciales de DockerHub y encriptarlas:

> travis encrypt DOCKER_USER=”dockerhub-username”

> travis encrypt DOCKER_PASS=”dockerhub-password”

y añadimos estos secretos a la sección env.globlal de .travis.yml

```
env:
  global:
  - secure: "....dockerhub-username data....."
  - secure: "....dockerhub-password data....."
```
Hay que tener cuidado con la encriptación de la password ya que si tiene caracteres especiales no funciona el docker login -u -p

Ahora, podemos añadir los comandos docker para construir y publicar la imagen a DockerHub en la sección **after_success**:

```
after_success:
  - docker login -u $DOCKER_USER -p $DOCKER_PASS
  - export TAG=`if [ "$TRAVIS_BRANCH" == "master" ]; then echo "latest"; else echo $TRAVIS_BRANCH; fi
  - docker tag $DOCKER_IMAGE_NAME $DOCKER_IMAGE_NAME:$TAG
  - docker push $DOCKER_IMAGE_NAME
```

### Paso 10: Desplegar la aplicación a Heroku

El último paso del ciclo de vida de nuestra aplicación es desplegarla en algún lugar. En este caso vamos a desplegarla en Heroku. Travis CI da soporte para desplegar una aplicación web por medio de Heroku. Para ello debemos tener una cuenta creada en Heroku y disponer de la HEROKU_API_KEY.

Lo primero que debemos hacer es encriptar HEROKU_API_KEY e incluirla en el fichero .travis.yml.

> travis encrypt HEROKU_API_KEY=”the-heroku-api-key”

Incluimos el valor generado en el fichero .travis.yml

```
env:
  global:
    - secure: "your encrypted heroku-api-key"
    .....
```

También incluimos la configuración de despliegue en la sección **deploy** del .travis.yml

```
deploy:
  skip_cleanup: true
  provider: heroku
  api_key: $HEROKU_API_KEY
  app: <your-app-name>
```

Además, hay que crear en la raíz del proyecto el fichero **Procfile**, donde se indican comandos que se ejecutan en el Heroku dynno de la aplicación. Para más información sobre este fichero consultar [https://devcenter.heroku.com/articles/procfile](https://devcenter.heroku.com/articles/procfile).

El contenido de este fichero es

```
web: java -Dserver.port=$PORT $JAVA_OPTS -jar <path-to-jar-file>
```

En *<path-to-jar-file>* se indicaría la ruta donde se encuentra el jar ejecutable de la aplicación, que en una aplicación SpringBoot suele generarse en build/libs.

*$PORT* viene definido por una variable de entorno configurada en Heroku. Esta variable la configuraremos con el valor que hayamos indicado en nuestra propiedad **server.port**. Para obtener más información sobre este asunto se puede consultar [https://devcenter.heroku.com/articles/config-vars](https://devcenter.heroku.com/articles/config-vars).


## Enlaces de interés

 - [https://sivalabs.in/2018/01/ci-cd-springboot-applications-using-travis-ci/](https://sivalabs.in/2018/01/ci-cd-springboot-applications-using-travis-ci/)
