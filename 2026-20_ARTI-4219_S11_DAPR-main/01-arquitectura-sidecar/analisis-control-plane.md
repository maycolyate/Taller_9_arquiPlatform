# Análisis del plano de control de Dapr

## TODO 1: Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

`dapr-sidecar-injector` no es un solo objeto de Kubernetes: el proceso se ejecuta dentro de los pods administrados por un **Deployment** llamado `dapr-sidecar-injector`, normalmente creado en el namespace `dapr-system`.

El Deployment instala una API HTTP que actúa como un webhook de admisión. Kubernetes lo relaciona con una configuración de tipo **`MutatingWebhookConfiguration`**. Esta configuración necesita interceptar operaciones de creación de pods y modificar (`mutate`) el objeto antes de que sea admitido y programado.

En este caso, el webhook detecta cuando el pod tiene la anotación `dapr.io/enabled: "true"` y agrega al pod el contenedor sidecar `daprd`, además de realizar ajustes como puertos, variables de entorno y configuración básica. No sería suficiente tener solamente un webhook `Validating`, porque ese tipo solo puede aceptar o rechazar un objeto, pero no puede añadir ni modificar contenedores.

### 2. ¿Cuántas réplicas del `dapr-operator` están corriendo? ¿Por qué el plano de control de Dapr es más liviano que el de Istio?

Con la instalación predeterminada de Dapr mostrada en la guía, el `dapr-operator` debe aparecer con **una réplica**:

```text
READY   STATUS    RESTARTS   AGE
DAPR-OPERATOR-...  1/1  Running
```

La cantidad exacta debe confirmarse con:

```bash
kubectl get deployment dapr-operator -n dapr-system -o jsonpath='{.spec.replicas}{"\n"}'
```

El plano de control de Dapr es, en términos generales, más liviano que el de Istio porque Dapr no intenta implementar toda la infraestructura de una malla de servicios dentro de su plano de control. Dapr delega la mayoría de las capacidades a sus sidecars y usa el plano de control principalmente para administrar componentes, configuraciones, inyección y funciones de actores y seguridad.

Por su parte, Istio administra una infraestructura más amplia de malla: descubrimiento y enrutamiento de tráfico, reglas de entrada y salida, políticas, identidad, certificados, telemetría y el procesamiento avanzado de la configuración hacia los proxies Envoy. Esto exige componentes de control y modelos de configuración más complejos.

Es importante distinguir que Dapr coloca la lógica de infraestructura en el sidecar `daprd`, mientras que Istio mantiene gran parte de su plano de datos en Envoy, con un plano de control más enfocado en la orquestación del tráfico de red.

### 3. Flujo de inyección del sidecar

El flujo es el siguiente:

1. Se crea un `Deployment`, `Pod` o recurso similar cuyo pod template incluye:

```yaml
metadata:
  annotations:
    dapr.io/enabled: "true"
    dapr.io/app-id: "pedido-service"
    dapr.io/app-port: "8080"
```

2. El usuario aplica el recurso mediante `kubectl apply`. Kubernetes crea primero la solicitud de admisión del pod y no inicia todavía los contenedores.

3. Los admission controllers de Kubernetes consultan la `MutatingWebhookConfiguration` que fue registrada por Dapr.

4. El webhook `dapr-sidecar-injector` recibe el pod y detecta `dapr.io/enabled: "true"`. También lee anotaciones relevantes, como el identificador de aplicación y el puerto del servicio.

5. El webhook modifica el objeto `Pod` antes de que se guarde definitivamente y agrega el contenedor `daprd`, las variables de entorno necesarias, los puertos y otros ajustes definidos por Dapr.

6. Kubernetes acepta el pod resultante y el scheduler selecciona un nodo. Ambos contenedores pasan a estar disponibles en el mismo pod:

```text
+------------------------------------------+
| Pod                                     |
|                                          |
|  +----------------+  +----------------+ |
|  | Aplicación     |  | daprd          | |
|  | puerto 8080    |  | puerto 3500    | |
|  +----------------+  +----------------+ |
+------------------------------------------+
```

7. La aplicación continúa ejecutando su lógica de negocio y se comunica con Dapr a través de `localhost:3500` mediante HTTP o gRPC. El `daprd` carga la configuración de Dapr y se conecta con los componentes declarados, como brokers Pub/Sub, almacenes de estado o administradores de secretos.

8. El pod funciona como unidad de red y almacenamiento compartido. Ambos contenedores pueden comunicarse por `localhost`, mientras que el sidecar encapsula la integración con la infraestructura externa.

---

## TODO 2: Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

Al consultar los CRDs de Dapr, pueden encontrarse varios recursos adicionales según la versión instalada. Los principales son:

| CRD | Recurso | Función |
|---|---|---|
| `components.dapr.io` | `Component` | Declara la infraestructura concreta usada por un building block, por ejemplo Kafka, Redis o un almacén de secretos. |
| `configurations.dapr.io` | `Configuration` | Define configuración global o por aplicación: trazabilidad, métricas, mTLS, políticas de acceso y reglas del sidecar. |
| `subscriptions.dapr.io` | `Subscription` | Suscribe una aplicación a un topic de Pub/Sub y asocia el evento con una ruta HTTP o gRPC. |
| `resiliencies.dapr.io` | `Resiliency` | Configura reintentos, timeouts y circuit breakers para operaciones de Dapr. |
| `httpendpoints.dapr.io` | `HTTPEndpoint` | Declara endpoints HTTP externos y permite invocarlos con el API de Dapr sin codificar sus detalles en la aplicación. |

Otros recursos pueden aparecer según la instalación y la versión. Además, Dapr puede usar CRDs propios para extensiones como el runtimes de workflows o nuevos componentes. Por eso es conveniente ejecutar:

```bash
kubectl get crds | grep dapr.io
```

Como comprobación de los nombres instalados en el entorno, la salida real debe agregarse a este documento o resumirse en el análisis.

### 5. Relación entre `components.dapr.io` y el concepto de Building Block

El building block es la **abstracción de la capacidad** que necesita la aplicación. Por ejemplo, "publicar y consumir mensajes" es el building block Pub/Sub. La aplicación habla con una API estable de Dapr y no conoce el broker concreto.

El objeto `Component` es la **implementación concreta que proporciona esa abstracción**. Por ejemplo:

- Building block Pub/Sub.
- Component `pubsub.kafka` → usa Kafka.
- Component `pubsub.redis` → usa Redis como broker.

Ambos conceptos están relacionados, pero no representan lo mismo:

```text
Building block Pub/Sub
        |
        v
Component pubsub.kafka
        |
        v
broker Kafka real
```

Por lo tanto, el building block es el contrato o modelo funcional, mientras que el CRD `components.dapr.io` configura la tecnología que realiza ese contrato. La separación permite cambiar de Kafka a Redis, RabbitMQ o cualquier otro componente compatible modificando la configuración de Dapr, sin cambiar el código que usa el building block.
