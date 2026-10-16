# Manual de uso: despliegue de proyectos académicos con GitHub Actions

> **EPIIS – UNSAAC | Guía paso a paso para estudiantes, docentes y egresados**  
> Este documento explica cómo desplegar una aplicación web o API utilizando el workflow reutilizable institucional, **sin tener que administrar Kubernetes, Helm ni el servidor**.  
> **Repositorio de las plantillas:** https://github.com/EPIIS/unsaac-devops-templates  
> **Workflow central:** `.github/workflows/reusable-deploy.yml` del repositorio anterior, en su rama `main`.

## 1. ¿Qué lograrás al finalizar?

Si tu proyecto ya se ejecuta correctamente dentro de un contenedor Docker y tienes autorización para usar el servidor EPIIS, podrás:

1. crear **un archivo YAML** en tu repositorio GitHub;
2. registrar de forma segura el acceso que te proporcione el administrador;
3. iniciar el despliegue con el botón **Run workflow**;
4. consultar el resultado en **Actions**;
5. publicar nuevas versiones sin copiar archivos manualmente al servidor.

**No necesitas instalar MicroK8s, Helm ni Kubernetes en tu computadora.** Tampoco necesitas copiar el workflow central ni las plantillas Helm a tu repositorio: el sistema reutiliza los que ya mantiene EPIIS.

### Conceptos mínimos

| Palabra | ¿Qué significa aquí? |
|---|---|
| **Repositorio** | Carpeta de tu proyecto almacenada en GitHub. |
| **Dockerfile** | Archivo con las instrucciones para construir la imagen de tu aplicación. |
| **Workflow** | Archivo YAML que le indica a GitHub Actions cuándo y cómo ejecutar tareas. |
| **GitHub Actions** | Servicio de GitHub que ejecuta el workflow. |
| **Rama (`main`)** | Versión principal del código desde la que realizaremos los despliegues. |
| **Secret** | Valor confidencial guardado en GitHub, sin escribirlo en el código fuente. |
| **`projectName`** | Nombre del proyecto completo; será también su namespace en Kubernetes. |
| **`appName`** | Nombre del componente que se despliega, como `frontend` o `backend`. |
| **Workflow reutilizable** | Workflow ya implementado por EPIIS que construye y despliega tu aplicación. |

---

## 2. Antes de empezar: lista de verificación

Verifica lo siguiente antes de crear ningún archivo:

- [ ] Tu aplicación está publicada en un **repositorio de GitHub** al que puedes enviar cambios.
- [ ] El repositorio tiene una rama principal identificada; en esta guía se utiliza **`main`**.
- [ ] Tu proyecto contiene un **`Dockerfile` que funciona** y construye una imagen Linux.
- [ ] Sabes en qué **puerto escucha la aplicación dentro del contenedor**, por ejemplo `80`, `3000` o `8080`.
- [ ] Si necesita una base de datos, esta **ya existe** y es accesible desde la infraestructura EPIIS; el sistema no crea bases de datos.
- [ ] Has solicitado al **administrador EPIIS** autorización y el secreto de conexión al servidor.
- [ ] GitHub Actions está habilitado para el repositorio y puedes utilizar workflows reutilizables públicos.

**Importante:** el workflow **no genera un Dockerfile ni arregla errores de compilación** de tu aplicación. Si no sabes construir una imagen Docker de tu proyecto, pide apoyo para preparar el Dockerfile antes de continuar. El procedimiento de despliegue será el mismo independientemente de que uses React, Angular, .NET, Java, Node.js o Python.

### ¿Dónde está mi rama principal?

En GitHub entra a tu repositorio y consulta **Settings → General → Default branch** (si tienes permisos). También puedes ver la rama seleccionada en el menú situado encima de la lista de archivos.

- Si la rama principal es **`main`**, sigue los ejemplos sin cambiar ese valor.
- Si la rama principal es **`master`** o **`develop`**, usa el nombre real de esa rama en los ejemplos que contienen `branches` para el evento `push`.
- **No confundas** la rama de tu proyecto con la rama del repositorio EPIIS: la parte `@main` en `uses: EPIIS/...@main` selecciona la versión del **workflow central**, no la rama de tu aplicación.

---

## 3. Paso 1: organiza tu repositorio

### Caso más simple: un solo componente

Si el repositorio solo contiene un frontend o un backend y su `Dockerfile` está en la raíz, tendrá una estructura parecida a esta:

```text
mi-proyecto/
├── Dockerfile
├── ...archivos de la aplicación...
└── .github/
    └── workflows/
        └── deploy.yml  ← ESTE ARCHIVO LO CREARÁS EN EL PASO 3
```

**La ruta obligatoria para los workflows que GitHub reconoce es `.github/workflows/`.** El nombre `deploy.yml` es una convención de este manual: puedes utilizar otro nombre terminado en `.yml` o `.yaml`, pero debe encontrarse dentro de esa carpeta.

**No crees** el archivo en `src/`, `docs/`, `.github/` sin `workflows/`, ni en el repositorio central de EPIIS. Debe estar **en el repositorio que contiene tu aplicación**.

### Si el Dockerfile está dentro de una subcarpeta

```text
mi-proyecto/
├── frontend/
│   ├── Dockerfile
│   └── ...
├── backend/
│   ├── Dockerfile
│   └── ...
└── .github/
    └── workflows/
        └── deploy.yml
```

En este caso `buildContext` será `./frontend` o `./backend`, según el componente. Si `buildContext: ./backend`, normalmente debes configurar `dockerfile: Dockerfile`, **no** `dockerfile: ./backend/Dockerfile`, porque el nombre se resuelve dentro del contexto que seleccionaste.

---

## 4. Paso 2: configura los secretos de GitHub

Los secretos se configuran **en el repositorio de tu proyecto**, no en el repositorio de plantillas EPIIS.

1. Abre tu repositorio en GitHub.
2. Entra a **Settings**.
3. En el menú lateral, selecciona **Secrets and variables → Actions**.
4. En **Repository secrets**, selecciona **New repository secret**.
5. Completa `Name` y `Secret`, y guarda el valor.

### Secreto obligatorio para el despliegue

| Nombre que debes crear en tu repositorio | Valor |
|---|---|
| `EPIIS_SSH_PASSWORD` | Contraseña o credencial aprobada y proporcionada por el administrador del servidor EPIIS. |

**No inventes el valor ni pegues una contraseña personal.** El administrador debe gestionar quién puede desplegar en la infraestructura institucional. Los usuarios sin autorización no podrán completar el despliegue aunque el YAML esté correcto.

El workflow de tu repositorio entregará ese valor al workflow central de esta forma:

```yaml
secrets:
  SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
```

Los nombres son diferentes **a propósito**: `EPIIS_SSH_PASSWORD` es el nombre que guardas en tu repositorio y `SSH_PASSWORD` es el nombre que espera recibir el workflow central.

### Secreto opcional para aplicaciones con variables de entorno

Si tu backend usa una base de datos o servicios externos, crea un segundo secreto:

| Nombre | Contenido |
|---|---|
| `APP_ENV` | Variables de entorno requeridas por la aplicación, una por línea. |

Ejemplo de **formato** (no contiene credenciales reales):

```dotenv
PORT=3000
DB_CNN=mongodb+srv://USUARIO:CLAVE@SERVIDOR/BASE
```

Agrega al YAML:

```yaml
secrets:
  SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
  APP_ENV: ${{ secrets.APP_ENV }}
```

El workflow central convierte estas variables en un **Secret de Kubernetes** llamado `<appName>-env`. Por ejemplo, si `appName` es `backend`, el Secret del clúster será `backend-env` dentro del namespace del proyecto. La aplicación lo lee normalmente mediante variables de entorno (`process.env.DB_CNN`, por ejemplo, en Node.js).

**Si tu aplicación no necesita variables de entorno, no crees `APP_ENV` y elimina esa línea del workflow.** No subas a GitHub un `.env` con credenciales, tokens o contraseñas.

> **Sobre seguridad:** la implementación actual del sistema utiliza una credencial SSH. Su distribución y el permiso para modificar workflows deben estar controlados por los administradores; para entornos de mayor criticidad conviene reforzar la autenticación SSH, la verificación de identidad del servidor y las aprobaciones de despliegue.

---

## 5. Paso 3: crea `.github/workflows/deploy.yml`

### Opción A: crear el archivo desde la página de GitHub

Esta es la opción recomendada para una persona que todavía no utiliza Git desde la terminal.

1. Entra al repositorio de **tu proyecto**.
2. Selecciona la rama **`main`** en el selector de ramas.
3. Pulsa **Add file → Create new file**.
4. En el campo de nombre, escribe **`.github/workflows/deploy.yml`**. GitHub creará las carpetas que falten.
5. Copia el YAML del apartado siguiente y modifica los valores de tu proyecto.
6. Selecciona **Commit changes**. Si GitHub propone crear una rama y un pull request, el workflow no estará disponible en `main` hasta que el cambio se integre.
7. Comprueba que, al volver a `main`, aparece el archivo en `.github/workflows/deploy.yml`.

### Opción B: crear el archivo en tu computadora

Desde la raíz de tu repositorio local:

```bash
mkdir -p .github/workflows
# Crea .github/workflows/deploy.yml con tu editor y pega el YAML.
git add .github/workflows/deploy.yml
git commit -m "Configurar despliegue con GitHub Actions"
git push origin main
```

**En Windows** puedes crear las carpetas con el explorador de archivos o con el editor y hacer el `git commit` desde el entorno Git que utilices.

### 5.1. Plantilla inicial: ejecución manual (recomendada para la primera prueba)

Este ejemplo supone un proyecto llamado `mi-proyecto` con **un frontend** que escucha en el puerto `80` y tiene un `Dockerfile` en la raíz. Cambia los valores indicados en la tabla de la sección 8 antes de ejecutarlo.

```yaml
name: Desplegar frontend

on:
  workflow_dispatch:

jobs:
  deploy:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@main
    with:
      projectName: mi-proyecto
      appName: frontend
      buildContext: .
      dockerfile: Dockerfile
      containerPort: "80"
      servicePort: "80"
      ingressEnabled: true
      ingressPath: /mi-proyecto
      internalPath: /
    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
```

**¿Por qué se usa `@main`?** Porque el workflow reutilizable y su Helm Chart están publicados en la rama `main` de `EPIIS/unsaac-devops-templates`. La referencia anterior a `@feature/thesis-reusable-deploy-v2` corresponde a la etapa de desarrollo y no es la recomendación para proyectos nuevos. Para reproducibilidad a largo plazo, el administrador puede proporcionar posteriormente una versión fija (`tag` o SHA de commit) para usar en lugar de `@main`.

**¿Qué debes modificar?** `projectName`, `appName`, `buildContext`, `dockerfile`, `containerPort`, `ingressPath` e `internalPath` según tu aplicación. No cambies la ruta del workflow reutilizable salvo indicación del administrador EPIIS.

### ¿Por qué no hay comandos `docker`, `ssh` o `helm` en este archivo?

Porque están implementados en el workflow **reutilizable**. Tu archivo solamente indica **cuándo ejecutar** ese mecanismo, **qué aplicación desplegar** y **qué secretos entregarle**. Esta separación es precisamente el objetivo del sistema.

---

## 6. Paso 4: inicia tu primer despliegue manual

Una vez que `.github/workflows/deploy.yml` esté en la **rama predeterminada** (`main` en esta guía):

1. Ingresa al repositorio en GitHub.
2. Abre la pestaña **Actions**.
3. En la columna izquierda, elige **Desplegar frontend** (o el nombre de tu workflow).
4. Selecciona **Run workflow**.
5. En **Use workflow from**, selecciona la rama que contiene la versión de tu aplicación que deseas desplegar (en este ejemplo, `main`).
6. Pulsa el botón verde **Run workflow**.
7. Espera a que aparezca la ejecución en la lista y ábrela para consultar cada paso.

### ¿Qué ocurre si no aparece «Run workflow»?

Comprueba, en este orden:

1. **El archivo está en `.github/workflows/deploy.yml`**, no en otra carpeta.
2. **La rama predeterminada** del repositorio contiene ese archivo. Para que GitHub habilite `workflow_dispatch`, su definición debe existir en la rama predeterminada.
3. El YAML contiene exactamente `on:` seguido de `workflow_dispatch:` con la indentación correcta.
4. GitHub Actions está habilitado y dispones de los permisos adecuados.
5. Estás abriendo el workflow correcto dentro de la pestaña **Actions**.

Con `workflow_dispatch`, GitHub permite seleccionar la rama que se va a ejecutar. El ejemplo no incluye filtros adicionales por rama: se desplegará el código de la rama seleccionada. Usa únicamente ramas y credenciales autorizadas por EPIIS.

### ¿Y si la rama predeterminada no es `main`?

El archivo del workflow debe existir en la **rama predeterminada real** para que aparezca el botón de ejecución manual. Puedes seleccionar la rama disponible desde **Run workflow**. Si habilitas despliegue automático por `push`, configura el nombre de esa rama en `on.push.branches`. No cambies el `@main` del workflow central por este motivo.

---

## 7. Paso 5: interpreta el resultado

En **Actions**, abre la ejecución y luego el job `deploy`. Los pasos corresponden al workflow central EPIIS. Entre ellos encontrarás:

1. **Checkout del proyecto:** obtiene tu código.
2. **Validar parámetros:** comprueba nombres, rutas, puertos y existencia del Dockerfile.
3. **Preparar Helm Chart:** genera la configuración de Kubernetes.
4. **Construir y exportar imagen Docker:** compila tu contenedor.
5. **Transferir imagen y Helm Chart:** envía los archivos al servidor institucional.
6. **Cargar imagen y publicar en el registro local:** incorpora la imagen al registro del servidor.
7. **Desplegar componente con Helm:** instala o actualiza los recursos.
8. **Verificar despliegue:** consulta Helm, pods, Service e Ingress.

### ¿Cómo saber si funcionó?

- Una ejecución **verde / Success** significa que el workflow terminó correctamente y las verificaciones automáticas definidas en él se completaron.
- Revisa el paso **Verificar despliegue**, especialmente el estado de los pods y los recursos expuestos.
- Si configuraste Ingress y el dominio institucional es accesible para ti, prueba la ruta correspondiente, por ejemplo `https://epiis.unsaac.edu.pe/mi-proyecto` **solo si el administrador confirmó que el dominio dispone de HTTPS y esa ruta está habilitada**.
- Un workflow verde **no garantiza** que cada funcionalidad de tu aplicación esté correcta; realiza una prueba funcional en el navegador o con una llamada HTTP a tu API.

### ¿Qué crea el sistema?

Para el ejemplo anterior:

```text
Proyecto:      mi-proyecto
Namespace:     mi-proyecto
Componente:    frontend
Deployment:    frontend-deployment
Service:       frontend-service
Ingress:       frontend-ingress (si está habilitado)
Imagen:        localhost:5000/mi-proyecto/frontend:<8-caracteres-del-commit>
```

No necesitas conectarte por SSH ni ejecutar `kubectl` tú mismo para efectuar el despliegue.

---

## 8. Paso 6: elige los valores correctos de tu aplicación

Todos los ejemplos usan el mismo workflow central. Lo que cambia son los parámetros del componente.

| Parámetro | Explicación simple | Ejemplo |
|---|---|---|
| `projectName` | Nombre común del proyecto; se convierte en namespace. | `bunker` |
| `appName` | Identificador único del componente dentro del proyecto. | `frontend` o `backend` |
| `buildContext` | Carpeta del repositorio que Docker utilizará para la construcción. | `.` o `./backend` |
| `dockerfile` | Nombre o ruta del Dockerfile **dentro del contexto anterior**. | `Dockerfile` |
| `containerPort` | Puerto **interno** de tu aplicación. No es el puerto de tu laptop. | `80`, `3000`, `8080` |
| `servicePort` | Puerto del Service de Kubernetes. Generalmente puedes dejar `80`. | `"80"` |
| `ingressEnabled` | `true` si quieres una ruta pública; `false` para un servicio solo interno. | `true` |
| `ingressPath` | Ruta pública, siempre dentro de `/<projectName>`. | `/bunker` o `/bunker/api` |
| `internalPath` | Ruta que espera tu aplicación después del enrutamiento. | `/` o `/api` |
| `ingressHost` | Dominio institucional, opcional; no modificar sin autorización. | `epiis.unsaac.edu.pe` |
| `user` | Usuario SSH del servidor, opcional; definido por EPIIS. | `epiis` |

**Reglas:** `projectName` admite hasta 63 caracteres y `appName` hasta 50; ambos deben usar minúsculas, números y guiones, y empezar y terminar con una letra o número. Los puertos deben estar entre 1 y 65535. Con Ingress habilitado, `ingressPath` no puede ser `/` y debe ser exactamente `/<projectName>` o comenzar por `/<projectName>/`.

### Cómo determinar `containerPort`

No uses un valor al azar. Debe coincidir con el puerto donde el proceso escucha **dentro del contenedor**:

- Frontend estático servido por Nginx: normalmente `80`.
- Backend Node.js: con frecuencia `3000`, según su código y configuración.
- Backend .NET o Java: verifica su Dockerfile y configuración efectiva; no presupongas `8080`.

Una línea `EXPOSE 3000` orienta sobre el puerto, pero **no inicia por sí misma** un servidor ni garantiza que la aplicación realmente escuche allí.

### `buildContext` y `dockerfile`: dos ejemplos

| Situación | `buildContext` | `dockerfile` |
|---|---|---|
| `Dockerfile` en la raíz del repo | `.` | `Dockerfile` |
| `backend/Dockerfile` dentro del repo | `./backend` | `Dockerfile` |

---

## 9. Ejemplo completo A: backend de ElBunker en un repositorio separado

**Ejemplo ilustrativo basado en la configuración de ElBunker:** comprueba que el backend realmente escuche en el puerto `3000` y que su Dockerfile esté en la raíz antes de copiarlo. Los nombres de repositorios no afectan a la agrupación en Kubernetes.

En el **repositorio del backend**, crea `.github/workflows/deploy.yml` con:

```yaml
name: Desplegar backend Bunker

on:
  workflow_dispatch:

jobs:
  deploy:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@main
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

En ese mismo repositorio crea los secretos `EPIIS_SSH_PASSWORD` y `APP_ENV`. En `APP_ENV` agrega **los nombres de variables que utiliza realmente el backend**, por ejemplo `DB_CNN=...` si el código utiliza `process.env.DB_CNN`. No inventes nombres de variables: revisa el código de tu aplicación.

Si tu API internamente atiende `GET /usuarios`, usa `internalPath: /`: una petición a `/bunker/api/usuarios` llegará al backend como `/usuarios`. Si tu API internamente atiende `GET /api/usuarios`, configura `internalPath: /api` para que reciba `/api/usuarios`. La ruta correcta depende de tu aplicación.

**La base de datos MongoDB Atlas debe existir y permitir conexiones desde el servidor EPIIS.** Este workflow solo despliega el backend; no crea cuentas ni bases de datos externas.

---

## 10. Ejemplo completo B: frontend de ElBunker en otro repositorio

En el **repositorio del frontend**, crea también `.github/workflows/deploy.yml`:

```yaml
name: Desplegar frontend Bunker

on:
  workflow_dispatch:

jobs:
  deploy:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@main
    with:
      projectName: bunker
      appName: frontend
      buildContext: .
      dockerfile: Dockerfile
      containerPort: "80"
      servicePort: "80"
      ingressEnabled: true
      ingressPath: /bunker
      internalPath: /
    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
```

Observa estas diferencias:

- **Los dos repositorios usan `projectName: bunker`** → los dos componentes quedan agrupados dentro del namespace `bunker`.
- Uno utiliza `appName: frontend` y el otro `appName: backend` → sus recursos Kubernetes son independientes.
- El frontend se publica bajo `/bunker` y el backend bajo `/bunker/api`.
- Cada repositorio tiene **su propio archivo YAML y sus propios secretos**. No basta con crear el secreto en el repositorio del backend para que también esté disponible en el del frontend.

### Importante para React, Vite o Angular

Si el frontend se sirve bajo la ruta `/bunker`, debe generar enlaces y archivos estáticos con esa ruta base. Por ejemplo, en un proyecto React que usa Vite podría ser necesario configurar `base: '/bunker/'` en `vite.config.js` y adecuar las rutas del cliente. En Angular se emplearía la configuración equivalente de `base href` para `/bunker/`.

El workflow **no modifica automáticamente** el código del frontend ni la configuración interna de Nginx. Si aparece una pantalla en blanco o fallan archivos `.js` o `.css`, revisa primero la ruta base y las peticiones que hace el navegador.

### Resultado esperado de los dos despliegues

```text
MicroK8s
└── namespace: bunker
    ├── frontend-deployment
    ├── frontend-service
    ├── frontend-ingress → /bunker
    ├── backend-deployment
    ├── backend-service
    └── backend-ingress  → /bunker/api
```

Este es el mismo escenario de **frontend y backend alojados en repositorios diferentes** que se utiliza para explicar la reutilización del sistema.

---

## 11. ¿Cómo se ejecuta: manualmente, por push o por pull request?

No tienes que instalar ninguna herramienta adicional: **esto se configura dentro del campo `on:` del archivo YAML**.

### Opción 1 — Solo manual (recomendada para el primer despliegue)

```yaml
on:
  workflow_dispatch:
```

El estudiante entra a **Actions → Nombre del workflow → Run workflow**. Un commit normal no inicia un despliegue automáticamente.

### Opción 2 — Manual y automática cada vez que haya un push a `main`

Una vez que el despliegue manual funcione, cambia **solo** el bloque `on:` por:

```yaml
on:
  workflow_dispatch:
  push:
    branches:
      - main
```

Ahora ocurrirán dos cosas:

- El usuario podrá ejecutar manualmente el workflow desde la pestaña Actions.
- Cada **push** que actualice `main` iniciará un nuevo despliegue, incluida la integración de un pull request en `main` si produce un nuevo commit.

No es necesario agregar condiciones al job. En este ejemplo, el evento `push` solo inicia despliegues automáticos al actualizar `main`, mientras que `workflow_dispatch` permite ejecutar manualmente desde la rama seleccionada. El estudiante debe disponer de autorización para desplegar en el servidor EPIIS.

**Atención:** al guardar en `main` un archivo que incluya `push` a `main`, ese mismo commit puede activar una ejecución. Configura **los secretos antes de hacer el commit** y comprueba que el YAML esté correcto.

### Opción 3 — Validar un pull request SIN desplegar

**No se recomienda incluir `pull_request` como disparador de un job que usa el workflow de despliegue**, porque ejecutaría operaciones hacia el servidor institucional antes de aprobar e integrar los cambios, y las ejecuciones desde forks externos no reciben los secretos habituales.

Si el curso necesita revisar los cambios antes de desplegar, crea **otro archivo opcional**, `.github/workflows/validar-pr.yml`, con:

```yaml
name: Validar pull request

on:
  pull_request:
    branches:
      - main

jobs:
  construir-imagen:
    runs-on: ubuntu-latest
    steps:
      - name: Obtener código
        uses: actions/checkout@v4
      - name: Comprobar que Docker construye el proyecto
        run: docker build -f Dockerfile -t prueba-academica .
```

Este workflow opcional solo comprueba que se pueda **construir** una imagen Docker; no prueba el funcionamiento completo de la aplicación y **no despliega al servidor**. Si el Dockerfile está en una subcarpeta, adapta el comando `docker build` a la estructura de tu proyecto.

### Tabla rápida de eventos

| Configuración | Cuándo se ejecuta | ¿Despliega? |
|---|---|---|
| `workflow_dispatch` | Cuando un usuario autorizado pulsa **Run workflow**. | Sí, desde la rama seleccionada. |
| `push` a `main` | Cuando se sube o integra un commit a `main`. | Sí, con el job de despliegue. |
| `pull_request` en `validar-pr.yml` | Cuando se abre o actualiza un PR hacia `main`. | No; solamente valida la construcción en nuestro ejemplo. |
| `push` a `feature/*` | Cuando se sube código a una rama de trabajo. | No, con los ejemplos de este manual. |

**¿Debe estar el workflow en `main`?** Para que aparezca el botón de ejecución manual, la definición con `workflow_dispatch` debe existir en la **rama predeterminada**, que en estos ejemplos es `main`. En **Run workflow** puedes elegir la rama del proyecto que deseas ejecutar, siempre que el archivo esté disponible allí. El evento `push` se activa solo para las ramas indicadas en `on.push.branches`.

---

## 12. ¿Y si frontend y backend están en el mismo repositorio?

Si los componentes comparten un repositorio, puedes utilizar **un solo** `.github/workflows/deploy.yml` con dos jobs que llaman al mismo workflow central:

```yaml
name: Desplegar proyecto completo

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  frontend:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@main
    with:
      projectName: tutorias
      appName: frontend
      buildContext: ./frontend
      dockerfile: Dockerfile
      containerPort: "80"
      ingressPath: /tutorias
      internalPath: /
    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}

  backend:
    uses: EPIIS/unsaac-devops-templates/.github/workflows/reusable-deploy.yml@main
    with:
      projectName: tutorias
      appName: backend
      buildContext: ./backend
      dockerfile: Dockerfile
      containerPort: "3000"
      ingressPath: /tutorias/api
      internalPath: /
    secrets:
      SSH_PASSWORD: ${{ secrets.EPIIS_SSH_PASSWORD }}
      APP_ENV: ${{ secrets.APP_ENV }}
```

Este ejemplo presupone dos subcarpetas `frontend/` y `backend/`, ambas con un Dockerfile propio. Comprueba los puertos y rutas de tu aplicación antes de copiarlo. Los dos jobs pueden ejecutarse de forma independiente; si uno falla, revisa su ejecución correspondiente.

---

## 13. Ejemplo de un componente interno, sin ruta pública

Para un servicio HTTP que solo será utilizado desde otros componentes del clúster:

```yaml
with:
  projectName: tutorias
  appName: auth
  buildContext: ./auth
  dockerfile: Dockerfile
  containerPort: "3000"
  ingressEnabled: false
```

Este es un **fragmento del bloque `with`**, no un archivo YAML independiente. Se crea el Deployment y el Service, pero **no** se publica un Ingress. No necesitas definir `ingressPath` cuando `ingressEnabled: false`.

---

## 14. Problemas frecuentes y cómo resolverlos

| Lo que observas | Causa probable | Qué revisar |
|---|---|---|
| No aparece **Run workflow** | No existe `workflow_dispatch` en la rama predeterminada, archivo mal ubicado o Actions deshabilitado. | Ruta `.github/workflows/deploy.yml`, rama predeterminada y permisos. |
| `buildContext no existe` | La carpeta indicada no está en tu repositorio. | Usa `.` si el Dockerfile está en la raíz o la carpeta correcta. |
| `no se encontró el Dockerfile` | Ruta/capitalización incorrecta del archivo. | Confirma dónde está `Dockerfile` y sus mayúsculas. |
| `docker build` falla | Dependencias, comandos o configuración incorrecta de tu aplicación. | Prueba el Dockerfile; lee el paso de construcción en Actions. |
| Error de `containerPort` | Puerto fuera de rango o distinto del real. | Comprueba el proceso dentro del contenedor. |
| Error `ingressPath debe ser...` | Ruta incompatible con `projectName`. | Usa `/mi-proyecto` o `/mi-proyecto/api`, según corresponda. |
| Error de SSH o SCP | Falta el secreto, credencial inválida o problema de conectividad/permisos. | Solicita revisión al administrador EPIIS; no publiques la contraseña en un issue. |
| Workflow verde, pero pantalla en blanco | Frontend compilado con ruta base incorrecta. | Configura `/mi-proyecto/` en tu framework y revisa assets en el navegador. |
| Backend no conecta a MongoDB | `APP_ENV` faltante, variable mal escrita o red externa sin acceso. | Nombre exacto de la variable, secreto, URL y acceso desde EPIIS. |
| `CrashLoopBackOff` o `rollout status` falla | El contenedor se inicia y se detiene, o no logra estar listo. | Logs del job, variables, comando de inicio, puerto y logs del pod con apoyo del administrador. |
| Publicaste cambios pero no se desplegaron | Solo tienes `workflow_dispatch` o hiciste push a otra rama. | Revisa la sección `on:` y la rama donde hiciste push. |
| Abres la ruta pero devuelve 404 | Ruta de Ingress o ruta interna incompatible. | `ingressPath`, `internalPath` y endpoints de tu aplicación. |

**Si falla un despliegue:** abre **Actions → ejecución fallida → job → paso en rojo**. Copia el nombre del paso y el mensaje de error, **ocultando credenciales**, para que el administrador o docente pueda ayudarte. No es necesario repetir el despliegue sin identificar la causa.

---

## 15. Responsabilidades: ¿qué hace el estudiante y qué hace EPIIS?

| Estudiante / propietario del proyecto | Plataforma EPIIS / administrador |
|---|---|
| Subir código a GitHub. | Mantener el clúster MicroK8s y el registro de imágenes. |
| Preparar y comprobar el Dockerfile. | Mantener el workflow reutilizable y el Helm Chart. |
| Crear `.github/workflows/deploy.yml`. | Autorizar el acceso y suministrar/configurar los secretos necesarios. |
| Configurar los parámetros de la aplicación. | Administrar el servidor, dominio, red y permisos. |
| Registrar variables de la aplicación en `APP_ENV` cuando corresponda. | Dar soporte cuando fallen los recursos de infraestructura. |
| Ejecutar Actions, revisar logs y comprobar el funcionamiento de su aplicación. | Establecer las políticas de uso de la infraestructura institucional. |

### Límites del sistema actual

La implementación automatiza la construcción, transferencia, registro y despliegue de **componentes web/API contenerizables**. No aprovisiona MongoDB, PostgreSQL u otras bases de datos. Tampoco incluye automáticamente pruebas unitarias, análisis de vulnerabilidades, entornos independientes de desarrollo/pruebas/producción, observabilidad avanzada ni soporte directo para aplicaciones móviles nativas o cargas especializadas de IA/IoT.

El workflow central utiliza actualmente una conexión SSH configurada por los administradores. **No modifica el servidor para que cualquier usuario de Internet pueda desplegar sin autorización.**

---

## 16. Lista final: ¿ya quedó desplegado?

- [ ] El repositorio tiene `Dockerfile` y el archivo `.github/workflows/deploy.yml` en la rama correcta.
- [ ] El secreto obligatorio está configurado por un usuario autorizado.
- [ ] `projectName`, `appName`, rutas y puerto corresponden a la aplicación.
- [ ] La ejecución en **Actions** aparece como **Success**.
- [ ] El paso **Verificar despliegue** informa recursos correctos.
- [ ] Abriste o probaste funcionalmente la aplicación o API desde la ruta habilitada.
- [ ] Guardaste el enlace de la ejecución y el identificador del commit como evidencia de tu despliegue.

**Si cumples estos puntos, el componente ya está integrado con el mecanismo de despliegue de EPIIS.** Para publicar una nueva versión, ejecuta otra vez el workflow manual o, si activaste `push` en `main`, sube un nuevo commit a esa rama.

---

## Referencias y soporte técnico

- [Repositorio institucional EPIIS](https://github.com/EPIIS/unsaac-devops-templates).
- [Workflow reutilizable implementado](https://github.com/EPIIS/unsaac-devops-templates/blob/main/.github/workflows/reusable-deploy.yml).
- [Arquitectura del sistema](ARQUITECTURA.md).
- [Escenarios soportados](ESCENARIOS-SOPORTADOS.md).
- [Documentación oficial de GitHub Actions](https://docs.github.com/en/actions).

*Este manual describe la implementación existente en `main` y contiene ejemplos didácticos. Los puertos, Dockerfiles, rutas y secretos deben ajustarse a cada aplicación. La disponibilidad de la infraestructura y las credenciales institucionales depende de la autorización del administrador EPIIS.*
