# Manual de uso del sistema de despliegue continuo

## 1. Propósito

Este manual explica cómo preparar un proyecto académico para utilizar el workflow reutilizable de despliegue de EPIIS.

El sistema no depende de un lenguaje o framework específico. La construcción de cada tecnología se define en el `Dockerfile` del componente. A partir de allí, el flujo de despliegue es común.

## 2. Conceptos

### Proyecto académico

Es la unidad lógica completa. Puede contener uno o varios componentes. El valor `projectName` identifica al proyecto y se utiliza como namespace de Kubernetes.

### Componente desplegable

Es una aplicación que puede construirse y ejecutarse de manera independiente mediante un contenedor. Cada componente utiliza un `appName` distinto y genera su propia imagen, Deployment y Service. Ingress es opcional.

Un microservicio HTTP/API se considera otro componente backend del proyecto.

## 3. Requisitos

Cada componente que será desplegado debe:

1. estar versionado en GitHub;
2. disponer de un `Dockerfile` funcional;
3. poder construirse como una imagen Linux;
4. ejecutar un proceso persistente dentro del contenedor;
5. exponer un puerto principal conocido;
6. externalizar valores sensibles y configuración cuando corresponda;
7. utilizar recursos compatibles con la infraestructura disponible.

El `Dockerfile` es responsabilidad del proyecto. Debe contener todo lo necesario para compilar y ejecutar Angular, React, .NET, Java, Node.js, Python u otra tecnología.

## 4. Reglas de nombres

`projectName` y `appName` deben utilizar letras minúsculas, números y guiones. Deben comenzar y terminar con un carácter alfanumérico.

Ejemplos de `projectName`:

```text
tutorias
seguimiento-tesis
proyecto-2026
```

Ejemplos de `appName`:

```text
frontend
backend
auth
notificaciones
```

Todos los componentes del mismo proyecto deben compartir el mismo `projectName` y utilizar un `appName` diferente.

## 5. Parámetros del workflow

| Parámetro | Obligatorio | Descripción |
|---|---:|---|
| `projectName` | Sí | Nombre del proyecto y namespace de Kubernetes. |
| `appName` | Sí | Nombre del componente desplegable. |
| `buildContext` | No | Directorio usado como contexto de `docker build`. Por defecto `.`. |
| `dockerfile` | No | Dockerfile relativo a `buildContext`. Por defecto `Dockerfile`. |
| `containerPort` | Sí | Puerto en el que escucha la aplicación dentro del contenedor. |
| `servicePort` | No | Puerto expuesto por el Service. Por defecto `80`. |
| `ingressEnabled` | No | Indica si el componente tendrá Ingress. Por defecto `true`. |
| `ingressPath` | No | Ruta HTTP asignada al componente. |
| `ingressHost` | No | Host institucional. Por defecto `epiis.unsaac.edu.pe`. |
| `user` | No | Usuario del servidor. Por defecto `epiis`. |

## 6. Secretos

### SSH_PASSWORD

Es requerido por la implementación actual porque GitHub Actions utiliza SCP y SSH para transferir artefactos y ejecutar el despliegue en el servidor institucional.

### APP_ENV

Es opcional. Permite proporcionar variables de configuración de la aplicación en formato `CLAVE=valor` sin almacenarlas en el repositorio.

Cuando se proporciona, el workflow crea o actualiza un Secret de Kubernetes llamado `<appName>-env` y el Deployment lo consume mediante `envFrom`.

No se deben almacenar credenciales en el Dockerfile, en `values.yaml` ni en archivos versionados.

## 7. Invocación mínima

Cada repositorio debe tener un workflow que invoque al workflow reutilizable.

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@feature/thesis-reusable-deploy-v2
    with:
      projectName: nombre-proyecto
      appName: nombre-componente
      buildContext: .
      dockerfile: Dockerfile
      containerPort: "8080"
      servicePort: "80"
      ingressEnabled: true
      ingressPath: /nombre-proyecto
    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
      APP_ENV: ${{ secrets.APP_ENV }}
```

Si el componente no utiliza `APP_ENV`, se omite esa asignación.

Durante la validación se utiliza la rama `feature/thesis-reusable-deploy-v2`. Después de estabilizar el flujo conviene referenciar una versión estable mediante tag o SHA.

## 8. Un solo componente

Si el repositorio contiene solamente frontend o solamente backend, se realiza una única invocación del workflow.

## 9. Frontend y backend en un mismo repositorio

No se utiliza un workflow especial para proyectos full stack. El repositorio invoca el workflow reutilizable una vez para el frontend y otra para el backend.

Ambos utilizan el mismo `projectName`, pero diferente `appName` y `buildContext`.

```text
Repositorio tutorias
├── frontend/ -> appName=frontend
└── backend/  -> appName=backend

Resultado:
namespace tutorias
├── frontend-deployment
└── backend-deployment
```

## 10. Frontend y backend en repositorios separados

Cada repositorio invoca el workflow de manera independiente. Para que ambos componentes queden agrupados deben utilizar el mismo `projectName`.

El namespace se crea de manera idempotente: la primera ejecución lo crea y las posteriores lo reutilizan.

## 11. Proyectos con varios servicios backend

Cada servicio HTTP/API se considera un componente independiente.

Ejemplo conceptual:

```text
projectName=tutorias

appName=backend
appName=auth
appName=notificaciones
```

Todos se despliegan en el namespace `tutorias`.

Un servicio que solo deba ser consumido desde otros componentes del clúster puede utilizar `ingressEnabled=false`.

## 12. Base de datos y servicios externos

La plataforma no aprovisiona bases de datos en la implementación inicial.

Un backend puede consumir SQL Server, PostgreSQL, MySQL, MongoDB u otro servicio externo si existe conectividad desde MicroK8s y la configuración necesaria se proporciona externamente.

La disponibilidad, respaldo y administración de la base de datos quedan fuera del alcance del workflow.

## 13. Flujo ejecutado

Para cada componente el workflow:

1. obtiene el código del repositorio que lo invoca;
2. valida parámetros y existencia del Dockerfile;
3. obtiene el Helm Chart base asociado a la misma revisión del workflow reutilizable;
4. genera los valores de despliegue;
5. construye la imagen Docker;
6. etiqueta la imagen con los primeros ocho caracteres del commit;
7. exporta la imagen como TAR;
8. transfiere imagen y Helm Chart al servidor mediante SCP;
9. ejecuta `docker load` en el servidor;
10. publica la imagen en `localhost:5000/<projectName>/<appName>:<sha>`;
11. crea el namespace si todavía no existe;
12. crea o actualiza el Secret opcional de configuración;
13. ejecuta `helm upgrade --install`;
14. espera el resultado del rollout;
15. verifica Helm, pods, Service e Ingress cuando corresponda.

## 14. Límites de la primera implementación

No forman parte del alcance inicial:

- aprovisionamiento de bases de datos;
- aplicaciones móviles nativas como artefacto final;
- cargas IoT dependientes de hardware especializado;
- cargas de IA que requieran recursos especializados;
- workers o procesos batch fuera del modelo web/API;
- múltiples ambientes de despliegue;
- monitoreo o escaneo de vulnerabilidades.

## 15. Verificación básica

Si una ejecución falla, revisar primero:

- que `buildContext` exista;
- que el Dockerfile exista;
- que `containerPort` sea el puerto real de la aplicación;
- que la aplicación escuche en una interfaz accesible desde el contenedor;
- que `projectName` y `appName` cumplan las reglas de nombres;
- que la ruta de Ingress sea válida;
- que el secreto de acceso al servidor esté configurado;
- que el backend tenga conectividad hacia sus dependencias externas;
- que las variables requeridas estén disponibles en `APP_ENV` cuando corresponda.
