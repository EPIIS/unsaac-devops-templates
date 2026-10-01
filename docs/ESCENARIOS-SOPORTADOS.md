# Escenarios soportados

La implementación utiliza un solo workflow reutilizable para desplegar componentes contenerizados. La organización se basa en dos conceptos:

```text
projectName = proyecto académico = namespace
appName     = componente desplegable
```

## Solo frontend

Un repositorio puede contener únicamente un frontend. El componente utiliza un `projectName` y un `appName` propios y se despliega en el namespace del proyecto.

## Solo backend

Un repositorio puede contener únicamente un backend web/API. El requisito es que pueda construirse con Docker, ejecutarse en Linux y exponer un puerto principal conocido.

## Frontend y backend en el mismo repositorio

El workflow del repositorio invoca el workflow reutilizable una vez por componente. Ambas invocaciones utilizan el mismo `projectName` y distintos `appName` y `buildContext`.

```text
Repositorio tutorias
├── frontend/  -> projectName=tutorias, appName=frontend
└── backend/   -> projectName=tutorias, appName=backend

MicroK8s
└── namespace tutorias
    ├── frontend
    └── backend
```

## Frontend y backend en repositorios separados

Cada repositorio invoca el workflow independientemente. Si ambos utilizan el mismo `projectName`, los componentes quedan en el mismo namespace.

## Varios servicios backend

Cada servicio HTTP/API se trata como un componente independiente. Un proyecto puede desplegar, por ejemplo, `backend`, `auth`, `notificaciones` y `reportes` dentro del mismo namespace.

Un servicio que solo deba ser consumido dentro del clúster puede utilizar `ingressEnabled=false`; mantiene su Service de Kubernetes sin ser publicado mediante Ingress.

## Por qué no se separa por tecnología

La responsabilidad se divide de la siguiente manera:

```text
Dockerfile del proyecto
    -> define cómo compilar y ejecutar la tecnología

Workflow reutilizable
    -> construye, transfiere, registra y despliega la imagen

Helm Chart
    -> representa el componente en Kubernetes
```

Por ello, Angular, React, .NET, Java, Node.js, Python u otra tecnología pueden utilizar el mismo flujo si cumplen el contrato de compatibilidad.

## Dependencias externas

Las bases de datos no son aprovisionadas por esta implementación. Un backend puede consumir una base de datos u otro servicio externo siempre que exista conectividad desde el clúster y la configuración necesaria sea proporcionada de forma externa.

## Fuera del alcance inicial

- aprovisionamiento de bases de datos;
- aplicaciones móviles nativas como artefacto final;
- cargas IoT que dependan de hardware especializado;
- cargas de IA con requerimientos especializados;
- workers o procesos batch que no sigan el modelo web/API;
- múltiples ambientes de despliegue.
