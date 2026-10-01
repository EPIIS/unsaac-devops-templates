# UNSAAC DevOps Templates - EPIIS

Repositorio de plantillas para estandarizar el despliegue continuo de proyectos académicos de la Escuela Profesional de Ingeniería Informática y de Sistemas de la UNSAAC.

Esta versión incorpora un workflow reutilizable orientado a **componentes contenerizados** y un Helm Chart base común. La unidad de organización es el **proyecto académico**, mientras que la unidad de despliegue es cada **componente** del proyecto.

## Modelo de organización

- `projectName`: identifica al proyecto académico y se utiliza como `namespace` de Kubernetes.
- `appName`: identifica a un componente desplegable dentro del proyecto.
- Todos los componentes que utilizan el mismo `projectName` quedan agrupados en el mismo namespace.
- Cada componente genera su propia imagen, Deployment y Service.
- Ingress puede habilitarse o deshabilitarse por componente.

Ejemplo conceptual:

```text
Proyecto: tutorias
Namespace: tutorias

tutorias
├── frontend
├── backend
└── auth
```

## Escenarios soportados

La misma lógica permite trabajar con:

- un proyecto formado únicamente por un frontend;
- un proyecto formado únicamente por un backend web/API;
- frontend y backend almacenados en repositorios separados;
- frontend y backend almacenados en un mismo repositorio;
- proyectos con varios componentes backend o microservicios HTTP/API.

No se crean workflows diferentes por tecnología. La compilación y preparación específica de Angular, React, .NET, Java, Node.js, Python u otra tecnología se define en el `Dockerfile` de cada componente. El workflow central recibe una interfaz común y ejecuta el mismo proceso de despliegue.

## Flujo implementado

```text
Repositorio del proyecto
        |
        v
GitHub Actions
        |
        +--> docker build
        +--> docker save
        |
        v
Servidor EPIIS (SCP / SSH)
        |
        +--> docker load
        +--> docker push localhost:5000/<projectName>/<appName>:<sha>
        |
        v
Registro local de contenedores
        |
        v
Helm
        |
        v
MicroK8s
        |
        +--> Namespace = projectName
        +--> Deployment = appName-deployment
        +--> Service = appName-service
        +--> Ingress opcional
```

## Archivos principales

- `.github/workflows/reusable-deploy.yml`: workflow reutilizable principal.
- `helm-base-chart/_base/`: Helm Chart común utilizado por el workflow.
- `docs/MANUAL-USO.md`: procedimiento para adaptar y desplegar un proyecto.
- `docs/ESCENARIOS-SOPORTADOS.md`: explicación de cómo se resuelven las distintas estructuras de proyectos.
- `docs/DISENO-TECNICO.md`: decisiones técnicas, alcance y contrato de compatibilidad.

Los workflows `wf-ci-cd-general.yml` y `wf-ci-cd-upload-project.yml` se conservan durante esta etapa para no afectar integraciones existentes. Los nuevos proyectos deben utilizar `reusable-deploy.yml`.

## Alcance de esta implementación

La implementación inicial está orientada a frontend web y backend web/API contenerizables. Cada componente debe:

- disponer de un `Dockerfile` funcional;
- poder ejecutarse en un contenedor Linux;
- exponer un puerto principal conocido;
- permitir externalizar configuración sensible cuando corresponda.

Las bases de datos no son aprovisionadas por este repositorio. Un backend puede consumir una base de datos o servicio externo siempre que exista conectividad desde el clúster y la configuración sea proporcionada mediante variables o secretos.

## Documentación

Comenzar por [docs/MANUAL-USO.md](docs/MANUAL-USO.md).
