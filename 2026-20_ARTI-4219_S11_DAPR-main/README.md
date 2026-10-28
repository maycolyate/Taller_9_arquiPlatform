# Sesión 11: Abstracción de Mensajería y Estandarización con Dapr

En esta sesión aplicamos el **patrón sidecar** de Dapr para construir un sistema de mensajería **agnóstico al broker**, demostrando cómo la infraestructura de red y el estándar **CloudEvents 1.0** se delegan al runtime, liberando al desarrollador del SDK propietario.

Construiremos una PoC completa de Pub/Sub sobre Kubernetes (Kind):

| Componente | Rol en la PoC |
|---|---|
| **Dapr Control Plane** | Gestiona la inyección del sidecar y los componentes declarativos |
| **Redis** | Broker de mensajería subyacente (invisible para el código de la aplicación) |
| **Subscriber (Python/Flask)** | Microservicio que escucha el tópico `orders` |
| **Publisher (Python)** | Microservicio que publica eventos de órdenes |

---

## Requisitos Previos

Antes de comenzar, asegúrate de tener instalado:

```bash
# Verificar kind
kind version        # >= 0.20.0

# Verificar kubectl
kubectl version --client

# Verificar helm
helm version        # >= 3.x

# Verificar Docker
docker info

# Verificar Python
python3 --version   # >= 3.9
```

---

## Guías de la PoC

Sigue las guías en orden:

1. **[Fase 1 — Arquitectura Sidecar: Clúster y Plano de Control Dapr](01-arquitectura-sidecar/README.md)**
2. **[Fase 2 — Componentes Declarativos: El Contrato de Mensajería](02-componentes-declarativos/README.md)**
3. **[Fase 3 — Aplicaciones Pub/Sub: Subscriber y Publisher](03-aplicaciones/README.md)**
4. **[Fase 4 — Reto de Gobernanza: TTL y Multi-Tenant Namespaces](04-reto-gobernanza/README.md)**
