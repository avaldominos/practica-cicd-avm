KeepCoding - Tech School\
BootCamp - DevOps & Cloud Computing - Edicion X.

Módulo - Ciclo de vida de un desarrollo - CI/CD\
Alumno: Alberto Valdominos Martín

> ⚠️ **Warning:**\
Este proyecto es un ejemplo de un pipeline CI/CD de despliegue de una aplicación Flask en Kubernetes. No está destinado para entornos de producción.


### Práctica Final.

La aplicación desplegada está basada en la existente en este [repositorio](https://github.com/benc-uk/python-demoapp)

Para la construcción del pipeline CI/CD se ha utilizado como herramientas [CircleCI](https://circleci.com/) y [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).

La practica cuenta con dos repositorios:
- Repositorio de la App: repositorio que realiza el trigger en CircleCI. https://github.com/avaldominos/practica-cicd-avm

- Repositorio de manifiestos: repositorio que monitoriza ArgoCD para el despliegue continuo. https://github.com/avaldominos/practica-cicd-avm-manifest


## Tabla de Contenidos

1. [Descripción de la App](#1-descripción-de-la-app)
2. [Descripción Pipeline CI/CD en CircleCI](#2-descripción-pipeline-cicd-en-circleci)
3. [Descripción Pipeline CI/CD en ArgoCD](#3-descripción-pipeline-cicd-en-argocd)
4. [Acceder a la aplicación](#4-acceder-a-la-aplicación)
5. [Entregables](#5-entregables)
6. [Recursos](#6-recursos)
7. [Licencia](#7-licencia)



## 1. Descripción de la App

La aplicación desplegada es una una aplicación web sencilla de Python Flask. La aplicación proporciona información del sistema y una pantalla de monitoreo en tiempo real con diales que muestran información de CPU, memoria, E/S y procesos.

La aplicación ha sido diseñada teniendo en cuenta las demostraciones y contenedores nativos de la nube, con el fin de proporcionar una aplicación que funcione realmente para la implementación, algo más que un "hola mundo", pero con el mínimo de requisitos previos. No está pensada como un ejemplo completo de una arquitectura en pleno funcionamiento o un diseño de software complejo.

Los usos típicos serían la implementación en Kubernetes, demostraciones de Docker, CI/CD, implementación en la nube, monitoreo y escalado automático.

**Screenshot**\
![Descripción de la imagen][def]


## 2. Descripción Pipeline CI/CD en CircleCI

El archivo `/.circleci/config.yaml` del repositorio de la App configura un pipeline de CI en CircleCI para ejecutar pruebas, análisis de calidad de código, y despliegues automáticos de una aplicación Python. La configuración incluye integración con SonarCloud para análisis de código y GitGuardian para detectar secretos. Además, se configura la creación y publicación de una imagen Docker y la actualización automática de un manifiesto de Kubernetes.

### 2.1. Estructura General

El pipeline está dividido en tres partes principales:

- Escaneo de seguridad y pruebas de código.
- Construcción y publicación de imagen Docker.
- Actualización del manifiesto de Kubernetes con la nueva imagen.

**Orbs Importados**

El archivo utiliza dos orbs para simplificar configuraciones complejas:

- SonarCloud (sonarsource/sonarcloud@2.0.0): Para análisis de calidad y seguridad del código.
- GitGuardian (gitguardian/ggshield@volatile): Para detectar secretos en el código antes de cada despliegue.

### 2.2. Configuración de los Jobs

1. `test`: Job de Pruebas y Escaneo

    Este job se encarga de:

    1. Descargar el código del proyecto (checkout).
    2. Restaurar dependencias almacenadas en un caché basado en el archivo requirements.txt.
    3. Instalar y ejecutar el comando make lint-fix para verificar y corregir errores de estilo.
    4. Ejecutar pruebas de la aplicación y generar un reporte (test-results.xml).
    5. Ejecutar un análisis de calidad de código con SonarCloud.
    6. Guardar las dependencias en el caché para futuras ejecuciones.
    7. Persistir el proyecto en un workspace para el siguiente job.


2. `build_and_push`: Job de Construcción y Publicación

    Este job realiza los siguientes pasos:

    1. Adjuntar el workspace del job test.
    2. Iniciar Docker remoto para permitir la creación de imágenes Docker.
    3. Iniciar sesión en Docker Hub usando credenciales almacenadas en variables de entorno.
    4. Construir y publicar la imagen Docker en Docker Hub.
    5. Guardar la imagen Docker como archivo .tar para su almacenamiento como artefacto.
    6. Clonar el repositorio de manifiestos de Kubernetes.
    7. Actualizar el manifiesto de Kubernetes con la nueva etiqueta de la imagen.
    8. Subir los cambios al repositorio de manifiestos.

### 2.3. Workflows

`scan_test_build_push`

Este flujo de trabajo coordina la ejecución de los jobs:

- GitGuardian Scan (ggshield/scan): Realiza un escaneo de seguridad en el código para evitar que secretos sean expuestos.
- Test (test): Ejecuta pruebas y el escaneo de SonarCloud. Este job depende del éxito de ggshield-scan.
- Build and Push (build_and_push): Construye y publica la imagen Docker y actualiza el manifiesto de Kubernetes. Este job solo se ejecuta en la rama main y depende de que test termine con éxito.

### 2.4. Variables de Entorno

Para el funcionamiento completo, es necesario definir las siguientes variables en CircleCI:

- SonarCloud y GitGuardian requieren contextos configurados para autenticación.
- Docker Hub:
    - `DOCKER_HUB_USER_ID`: Usuario de Docker Hub.
    - `DOCKER_HUB_PASSWORD`: Contraseña de Docker Hub.
    - `IMAGE_NAME`: Nombre de la imagen.
- GitHub Token: `GITHUB_TOKEN` para autenticar y actualizar el repositorio de manifiestos.

Este pipeline de CircleCI asegura que el código esté libre de errores de estilo y posibles secretos, ejecuta pruebas, y finalmente despliega una imagen Docker en este [repositorio de Docker Hub](https://hub.docker.com/repository/docker/avaldominos/python-app/)  lista para desplegar. Además, actualiza automáticamente para reflejar los cambios el manifiesto `/k8s/deployment.yaml` de Kubernetes ubicado en el [repositorio de manifiestos](https://github.com/avaldominos/practica-cicd-avm-manifest), permitiendo una entrega continua y controlada en la rama main.


## 3. Descripción Pipeline CI/CD en ArgoCD


El archivo `application.yaml` ubicado en el [repositorio de manifiestos](https://github.com/avaldominos/practica-cicd-avm-manifest) define una configuración de despliegue automatizado para ArgoCD. Permite gestionar la sincronización de la aplicación basada en los manifiestos de configuración de la misma almacenados en la carpeta `/k8s` del [repositorio de manifiestos](https://github.com/avaldominos/practica-cicd-avm-manifest).
A continuación, se describe la configuración y funcionalidades principales del archivo.

Descripción General
El archivo configura una aplicación en ArgoCD que:

- Recupera manifiestos Kubernetes desde un repositorio en el [repositorio de manifiestos](https://github.com/avaldominos/practica-cicd-avm-manifest).
- Despliega la aplicación en un clúster Kubernetes específico.
- Permite la sincronización automática para mantener la infraestructura actualizada según el estado de los manifiestos en el [repositorio de manifiestos](https://github.com/avaldominos/practica-cicd-avm-manifest).



## 4. Acceder a la aplicación

   Para comprobar que la aplicación funciona correctamente realiza los siguientes pasos:
   
   1. Abre tu navegador y visita http://python-app-127.0.0.1.nip.io para verificar que la aplicación está funcionando correctamente.:

   2. Deberias ver una salida similar a:

**Screenshot**\
![screenshot de la app][def2]

   Si es así, la aplicación se ha desplegado correctamente.


## 5. Entregables

Los entregables solicitados en la practica se encuentran en:

1. [El enlace al repositorio de GitHub donde se encuentra el código de la aplicación](https://github.com/avaldominos/practica-cicd-avm/tree/main/src).
2. [El enlace al repositorio de artefactos donde se encuentra el artefacto de la aplicación](https://hub.docker.com/repository/docker/avaldominos/python-app/general).
3. [El fichero de configuración del pipeline de CI/CD](https://github.com/avaldominos/practica-cicd-avm/blob/main/.circleci/config.yml).
4. Enlace o screenshot del pipeline de CI/CD
    - [Enlace CircleCI](https://app.circleci.com/pipelines/github/avaldominos/practica-cicd-avm)
    - [Screenshot CircleCI](https://github.com/avaldominos/practica-cicd-avm/blob/main/screenshots/CircleCI_Pipeline.jpg)    
5. [Los manifestos de Kubernetes para el despliegue de la aplicación](https://github.com/avaldominos/practica-cicd-avm-manifest/tree/main).
6. Enlace o screenshot de la aplicación desplegada.
    - [Screenshot App Desplegada en cluster Kind](https://github.com/avaldominos/practica-cicd-avm/blob/main/screenshots/App.jpg)
7. Enlace o screenshot del proyecto en ArgoCD.
    - [Screenshot ArgoCD](https://github.com/avaldominos/practica-cicd-avm/blob/main/screenshots/ArgoCD..jpg)
8. Enlace o screenshot del proyecto en SonarCloud.
    - [Enlace SonarCloud](https://sonarcloud.io/project/overview?id=avaldominos_practica-cicd-avm)
    - [Screenshot SonarCloud](https://github.com/avaldominos/practica-cicd-avm/blob/main/screenshots/SonarCloud.jpg)
9. Enlace o screenshot del proyecto en Snyk o GitGuardian.
    - [Screenshot GitGuardian](https://github.com/avaldominos/practica-cicd-avm/blob/main/screenshots/GitGuardian.jpg)
10. Enlace a un vídeo de Youtube donde se explique la práctica.

## 6. Recursos

- [Documentación oficial CircleCI](https://circleci.com/docs/)
- [Documentación oficial ArgoCD](https://argo-cd.readthedocs.io/en/stable/)
- [Documentación oficial de Kubernetes](https://kubernetes.io/docs/home/)\
      - [Documentación oficial Kind](https://kind.sigs.k8s.io/)


## 7. Licencia
Este proyecto está bajo la licencia GNU GPL.




[def]: ./screenshots/App.jpg
[def2]: ./screenshots/App.jpg