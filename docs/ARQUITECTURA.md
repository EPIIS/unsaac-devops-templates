# Arquitectura del flujo reutilizable

## Modelo

La solución distingue entre proyecto académico y componente desplegable.

- `projectName` identifica el proyecto y define el namespace de Kubernetes.
- `appName` identifica cada componente que se despliega dentro del proyecto.

Un proyecto puede contener un frontend, un backend o varios servicios backend. Todos comparten `projectName` y cada uno utiliza un `appName` diferente.

## Contrato de compatibilidad

Un componente puede utilizar el flujo cuando dispone de un Dockerfile funcional, puede ejecutarse en un contenedor Linux, conoce su puerto principal y permite proporcionar configuración externa cuando sea necesaria.

La tecnología concreta queda encapsulada en el Dockerfile; por ello el workflow no necesita lógica específica para Angular, React, .NET, Java, Node.js o Python.

## Flujo

```text
Repositorio GitHub
      |
      v
Workflow reutilizable
      |
      +-- construir imagen
      +-- exportar imagen
      +-- transferir al servidor
      |
      v
Registro local localhost:5000
      |
      v
Helm
      |
      v
MicroK8s
      |
      +-- namespace = projectName
      +-- Deployment = appName
      +-- Service = appName
      +-- Ingress opcional
```

## Organización de imágenes

Las imágenes siguen la convención:

```text
localhost:5000/<projectName>/<appName>:<short-sha>
```

## Repositorios

La misma lógica soporta componentes en repositorios separados y componentes en un monorepo. En un monorepo se invoca el workflow una vez por componente. En repositorios separados, todos los repositorios del mismo proyecto utilizan el mismo `projectName`.

## Persistencia

Las bases de datos no son creadas por el flujo. Un backend puede consumir servicios de persistencia externos cuando exista conectividad desde el clúster.

## Alcance inicial

La implementación inicial está orientada a frontend web y backend web/API contenerizables. Otras cargas de trabajo pueden evaluarse como extensiones futuras.
