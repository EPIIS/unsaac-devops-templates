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
| `ingressPath` | Sí cuando Ingress está habilitado | Ruta pública del componente. Debe ser `/projectName` o comenzar con `/projectName/`. |
| `internalPath` | No | Ruta base que espera la aplicación dentro del contenedor. Por defecto `/`. |
| `ingressHost` | No | Host utilizado por Ingress. Por defecto `epiis.unsaac.edu.pe`. |
| `user` | No | Usuario del servidor. Por defecto `epiis`. |

## 6. Secretos

### SSH_PASSWORD

Es requerido por la implementación actual porque GitHub Actions utiliza SCP y SSH para transferir artefactos y ejecutar el despliegue en el servidor institucional.

### APP_ENV

`APP_ENV` es un secreto opcional utilizado para proporcionar variables de configuración requeridas por la aplicación, como cadenas de conexión a bases de datos, credenciales de servicios externos o parámetros propios del entorno, sin almacenarlas directamente en el repositorio.

Cuando un componente requiera estas variables, se debe crear un secreto denominado `APP_ENV` en:

`Settings → Secrets and variables → Actions`

dentro del repositorio de GitHub correspondiente al componente.

El contenido del secreto debe utilizar el formato `CLAVE=valor`, pudiendo incluir varias variables en líneas independientes.

Ejemplo:

```env
PORT=3000
DB_CNN=mongodb+srv://usuario:password@cluster.mongodb.net/base
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=https://developers.google.com/oauthplayground
GOOGLE_REFRESH_TOKEN=...
```

El workflow consumidor debe proporcionar este secreto al workflow reutilizable mediante la sección `secrets`:

```yaml
secrets:
  SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
  APP_ENV: ${{ secrets.APP_ENV }}
```

Por ejemplo, un backend puede utilizar el workflow reutilizable de la siguiente manera:

```yaml
name: Deploy Backend

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@feature/thesis-reusable-deploy-v2

    with:
      projectName: bunker
      appName: backend
      buildContext: .
      dockerfile: Dockerfile
      containerPort: "3000"
      servicePort: "80"
      ingressEnabled: true
      ingressPath: /bunker/api
      internalPath: /

    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
      APP_ENV: ${{ secrets.APP_ENV }}
```

Durante la ejecución, el workflow crea o actualiza automáticamente un Secret de Kubernetes denominado:

```text
<appName>-env
```

Por ejemplo, si el componente utiliza:

```yaml
appName: backend
```

el Secret generado será:

```text
backend-env
```

Este Secret se crea dentro del namespace correspondiente al proyecto y el Deployment lo consume mediante `envFrom`.

De esta forma, las variables quedan disponibles dentro del contenedor utilizando el mecanismo habitual de variables de entorno de cada tecnología.

Por ejemplo, en una aplicación Node.js:

```javascript
process.env.DB_CNN
```

La conexión a MongoDB puede realizarse normalmente mediante la variable de entorno:

```javascript
mongoose.connect(process.env.DB_CNN)
```

El flujo realizado por la plataforma es el siguiente:

```text
GitHub Secret APP_ENV
        ↓
Workflow consumidor
        ↓
Workflow reutilizable
        ↓
Archivo temporal de configuración
        ↓
Secret de Kubernetes <appName>-env
        ↓
Deployment
        ↓
Contenedor
        ↓
Variables de entorno de la aplicación
```

Por ejemplo, para un proyecto denominado `bunker` cuyo componente es `backend`, se tendría:

```text
Proyecto: bunker
Componente: backend
Namespace: bunker
Secret: backend-env
Deployment: backend-deployment
```

La aplicación puede acceder a las variables configuradas en `APP_ENV` directamente desde su entorno de ejecución.

### Bases de datos externas

Las bases de datos no son creadas ni administradas por la plataforma de despliegue.

Un backend puede utilizar una base de datos externa siempre que exista conectividad desde la infraestructura institucional hacia dicho servicio y que las credenciales necesarias sean proporcionadas mediante `APP_ENV`.

Por ejemplo:

```env
DB_CNN=mongodb+srv://usuario:password@cluster.mongodb.net/base
```

Este mecanismo puede utilizarse para conexiones hacia servicios como:

- MongoDB Atlas.
- PostgreSQL externo.
- SQL Server externo.
- MySQL externo.
- Otros servicios de persistencia accesibles desde la infraestructura institucional.

Antes de realizar el despliegue, se recomienda verificar que el servidor institucional pueda resolver el dominio y establecer conexión con el servicio externo correspondiente.

La plataforma se encarga del despliegue de la aplicación, pero el aprovisionamiento, configuración y administración de la base de datos permanecen bajo responsabilidad del proyecto que la utiliza.

### Servicios externos

`APP_ENV` también puede utilizarse para proporcionar credenciales o parámetros necesarios para consumir servicios externos.

Por ejemplo, una aplicación que utilice Google Drive API podría requerir:

```env
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=https://developers.google.com/oauthplayground
GOOGLE_REFRESH_TOKEN=...
```

Estas variables serán proporcionadas al contenedor mediante el mismo Secret de Kubernetes generado durante el despliegue.

En una aplicación Node.js pueden ser utilizadas de la siguiente manera:

```javascript
const idClient = process.env.GOOGLE_CLIENT_ID
const secretClient = process.env.GOOGLE_CLIENT_SECRET
const redirectUri = process.env.GOOGLE_REDIRECT_URI
const refreshToken = process.env.GOOGLE_REFRESH_TOKEN
```

### Recomendaciones de seguridad

No se deben almacenar credenciales o información sensible directamente en:

- `Dockerfile`.
- `values.yaml`.
- archivos `.env` versionados.
- código fuente.
- archivos YAML del workflow.
- repositorios Git.

Las credenciales deben almacenarse utilizando GitHub Secrets y proporcionarse al workflow reutilizable mediante `APP_ENV`.

El archivo temporal generado durante el despliegue se utiliza únicamente para crear o actualizar el Secret correspondiente en Kubernetes.

El uso de `APP_ENV` permite separar la configuración sensible del código fuente y reutilizar la misma imagen de la aplicación con diferentes configuraciones de ejecución.

### Ejemplo para un backend con MongoDB

Para un backend que utiliza MongoDB, el procedimiento sería:

1. Ingresar al repositorio del backend en GitHub.

2. Ir a:

   `Settings → Secrets and variables → Actions`

3. Crear un nuevo Repository Secret con el nombre:

   ```text
   APP_ENV
   ```

4. Colocar como valor las variables requeridas por la aplicación:

   ```env
   PORT=3000
   DB_CNN=mongodb+srv://usuario:password@cluster.mongodb.net/base
   ```

5. En el workflow consumidor agregar:

   ```yaml
   secrets:
     SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
     APP_ENV: ${{ secrets.APP_ENV }}
   ```

6. Ejecutar el workflow.

7. Durante el despliegue se creará automáticamente:

   ```text
   namespace: bunker
   secret: backend-env
   deployment: backend-deployment
   ```

8. El backend podrá utilizar la cadena de conexión mediante:

   ```javascript
   process.env.DB_CNN
   ```

No es necesario copiar manualmente el archivo `.env` al servidor ni incluir las credenciales dentro de la imagen Docker.

## 7. Enrutamiento con Service e Ingress

El sistema utiliza un único dominio institucional por defecto:

```text
epiis.unsaac.edu.pe
```

Los distintos proyectos y componentes se diferencian mediante rutas.

Ejemplo conceptual:

```text
epiis.unsaac.edu.pe/tutorias       -> frontend del proyecto Tutorías
epiis.unsaac.edu.pe/tutorias/api   -> backend principal del proyecto Tutorías
epiis.unsaac.edu.pe/tutorias/auth  -> servicio de autenticación del proyecto Tutorías
```

El flujo de una solicitud es:

```text
Cliente
  |
  v
Ingress
  |
  v
Service de Kubernetes
  |
  v
Pod del componente
```

El Ingress decide qué Service debe recibir una solicitud utilizando el host y la ruta pública. El Service dirige el tráfico al puerto real del contenedor mediante `targetPort`.

Ejemplo conceptual:

```text
Solicitud pública:
/tutorias/api/usuarios

Ingress:
/tutorias/api -> backend-service

Service:
puerto 80 -> puerto 8080 del contenedor
```

### Ruta pública y ruta interna

La plataforma distingue dos conceptos:

- `ingressPath`: ruta pública visible para el usuario.
- `internalPath`: ruta base que espera realmente la aplicación dentro del contenedor.

El Ingress elimina el prefijo público y reconstruye la ruta que recibe la aplicación.

Ejemplo de backend:

```text
ingressPath  = /tutorias/api
internalPath = /api

Solicitud externa:
/tutorias/api/usuarios

Ruta recibida por el backend:
/api/usuarios
```

Ejemplo de frontend:

```text
ingressPath  = /tutorias
internalPath = /

Solicitud externa:
/tutorias/assets/logo.png

Ruta recibida por el contenedor:
/assets/logo.png
```

Esto evita obligar a modificar las rutas internas de una API únicamente porque el proyecto se publique debajo de una ruta institucional.

### ¿Es necesario modificar Nginx dentro del proyecto?

No como requisito general del sistema. El Ingress institucional se encarga del enrutamiento y del reemplazo del prefijo público antes de enviar la solicitud al contenedor.

Sin embargo, una aplicación frontend que se publica bajo una subruta debe generar sus enlaces, assets y navegación considerando esa ruta pública. Este comportamiento depende del framework utilizado y no puede ser inferido de forma segura por un workflow genérico.

Por ejemplo, si un frontend se publica en:

```text
/tutorias
```

el navegador debe solicitar recursos bajo esa misma base pública, por ejemplo:

```text
/tutorias/main.js
/tutorias/assets/logo.png
```

El proyecto debe configurar el mecanismo de base path propio de su tecnología durante la construcción. En Angular puede resolverse mediante su configuración de base href o una opción equivalente del proceso de build; en React, Vue u otros frameworks se utiliza el mecanismo equivalente para definir la ruta pública base.

Esto no significa que el estudiante deba administrar el Ingress de Kubernetes ni modificar el Nginx institucional. Solo debe asegurarse de que su frontend esté preparado para ejecutarse bajo la ruta pública asignada.

## 8. Convención de rutas

Para evitar colisiones, la implementación utiliza la siguiente convención:

```text
Frontend principal:       /<projectName>
Backend principal:        /<projectName>/api
Otros servicios públicos: /<projectName>/<appName>
```

Ejemplo:

```text
Proyecto tutorias

frontend -> /tutorias
backend  -> /tutorias/api
auth     -> /tutorias/auth
```

Un componente interno que no necesita acceso desde Internet debe utilizar `ingressEnabled=false`. El Service continuará disponible dentro del namespace para la comunicación entre componentes.

## 9. Invocación mínima

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
      internalPath: /
    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
      APP_ENV: ${{ secrets.APP_ENV }}
```

Si el componente no utiliza `APP_ENV`, se omite esa asignación.

Durante la validación se utiliza la rama `feature/thesis-reusable-deploy-v2`. Después de estabilizar el flujo conviene referenciar una versión estable mediante tag o SHA.

## 10. Un solo componente

Si el repositorio contiene solamente frontend o solamente backend, se realiza una única invocación del workflow.

Ejemplo conceptual para un frontend:

```text
projectName = portal-academico
appName = frontend
ingressPath = /portal-academico
internalPath = /
```

Ejemplo conceptual para un backend cuya API interna comienza en `/api`:

```text
projectName = tutorias
appName = backend
ingressPath = /tutorias/api
internalPath = /api
```

## 11. Frontend y backend en un mismo repositorio

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

A nivel de acceso público puede utilizarse:

```text
/tutorias      -> frontend
/tutorias/api  -> backend
```

## 12. Frontend y backend en repositorios separados

Cada repositorio invoca el workflow de manera independiente. Para que ambos componentes queden agrupados deben utilizar el mismo `projectName`.

El namespace se crea de manera idempotente: la primera ejecución lo crea y las posteriores lo reutilizan.

Aunque los repositorios sean diferentes, las rutas pueden compartir la misma estructura pública:

```text
/tutorias      -> frontend
/tutorias/api  -> backend
```

## 13. Proyectos con varios servicios backend

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

Si un servicio debe ser público puede asignarse una ruta como:

```text
/tutorias/auth
/tutorias/reportes
```

## 14. Base de datos y servicios externos

La plataforma no aprovisiona bases de datos en la implementación inicial.

Un backend puede consumir SQL Server, PostgreSQL, MySQL, MongoDB u otro servicio externo si existe conectividad desde MicroK8s y la configuración necesaria se proporciona externamente.

La disponibilidad, respaldo y administración de la base de datos quedan fuera del alcance del workflow.

## 15. Flujo ejecutado

Para cada componente el workflow:

1. obtiene el código del repositorio que lo invoca;
2. valida parámetros y existencia del Dockerfile;
3. valida las rutas públicas e internas cuando Ingress está habilitado;
4. obtiene el Helm Chart base asociado a la misma revisión del workflow reutilizable;
5. genera los valores de despliegue;
6. construye la imagen Docker;
7. etiqueta la imagen con los primeros ocho caracteres del commit;
8. exporta la imagen como TAR;
9. transfiere imagen y Helm Chart al servidor mediante SCP;
10. ejecuta `docker load` en el servidor;
11. publica la imagen en `localhost:5000/<projectName>/<appName>:<sha>`;
12. crea el namespace si todavía no existe;
13. crea o actualiza el Secret opcional de configuración;
14. ejecuta `helm upgrade --install`;
15. espera el resultado del rollout;
16. verifica Helm, pods, Service e Ingress cuando corresponda.

## 16. Límites de la primera implementación

No forman parte del alcance inicial:

- aprovisionamiento de bases de datos;
- aplicaciones móviles nativas como artefacto final;
- cargas IoT dependientes de hardware especializado;
- cargas de IA que requieran recursos especializados;
- workers o procesos batch fuera del modelo web/API;
- múltiples ambientes de despliegue;
- monitoreo o escaneo de vulnerabilidades.

## 17. Verificación básica

Si una ejecución falla, revisar primero:

- que `buildContext` exista;
- que el Dockerfile exista;
- que `containerPort` sea el puerto real de la aplicación;
- que la aplicación escuche en una interfaz accesible desde el contenedor;
- que `projectName` y `appName` cumplan las reglas de nombres;
- que `ingressPath` sea `/projectName` o esté debajo de esa ruta;
- que `internalPath` coincida con la ruta base que espera la aplicación;
- que un frontend esté construido para utilizar su ruta pública base;
- que el secreto de acceso al servidor esté configurado;
- que el backend tenga conectividad hacia sus dependencias externas;
- que las variables requeridas estén disponibles en `APP_ENV` cuando corresponda.
