# TP Middlewares Orientados a Mensajes

## 1. Introducción

El objetivo del trabajo práctico es implementar una interfaz simplificada de un Middleware Orientado a Mensajes (MOM), utilizando RabbitMQ como sistema de mensajería subyacente.

Para la implementación se eligió el lenguaje **Python**, utilizando la biblioteca `pika` para realizar la comunicación con RabbitMQ.

La implementación se realizó respetando la interfaz provista por el trabajo práctico, de manera que los componentes que utilizan el middleware no necesiten conocer los detalles específicos de RabbitMQ.

## La consigna establece como requisitos el correcto funcionamiento de la interfaz, el manejo de errores, el uso de buenas prácticas, Docker/Docker Compose y la demostración de los modelos Producer-Consumer y Publisher-Subscriber.

## 2. Elección de la implementación

Se optó por realizar el trabajo en **Python**.

La implementación se encuentra en:

```text
python/src/common/middleware/middleware_rabbitmq.py
```

Este archivo contiene las implementaciones concretas de:

* `MessageMiddlewareQueueRabbitMQ`
* `MessageMiddlewareExchangeRabbitMQ`

La implementación concreta utiliza RabbitMQ mediante `pika`, pero mantiene ocultos los detalles de conexión y comunicación con el broker.

---

## 3. Modelo Producer-Consumer

El modelo Producer-Consumer se implementa mediante la abstracción de `Queue`.

Un productor genera mensajes y los envía a una cola. Los consumidores se conectan a esa misma cola y procesan los mensajes disponibles.

El flujo general es:

```text
Producer
   |
   | mensaje
   v
Queue
   |
   | mensaje
   v
Consumer
```

La implementación de `MessageMiddlewareQueueRabbitMQ` realiza las operaciones necesarias para:

1. Establecer una conexión con RabbitMQ.
2. Declarar la cola.
3. Enviar mensajes a la cola.
4. Consumir mensajes.
5. Confirmar el procesamiento mediante ACK.
6. Rechazar mensajes mediante NACK cuando corresponde.
7. Detener el consumo.
8. Cerrar correctamente la conexión.

Para enviar un mensaje a una cola se utiliza el exchange por defecto de RabbitMQ, utilizando como `routing_key` el nombre de la cola.

De esta manera, el usuario de la interfaz no necesita conocer cómo se realiza internamente la comunicación con RabbitMQ.

---

## 4. Modelo Publisher-Subscriber

El modelo Publisher-Subscriber se implementa mediante la abstracción de `Exchange`.

En este modelo, los productores publican mensajes en un exchange y los consumidores se suscriben mediante sus propias colas y determinadas `routing_keys`.

El flujo general es:

```text
                 +--> Queue --> Subscriber 1
                 |
Publisher --> Exchange
                 |
                 +--> Queue --> Subscriber 2
```

La implementación utiliza un exchange de tipo `direct`.

Cada instancia del consumidor obtiene una cola propia y la vincula al exchange utilizando las `routing_keys` recibidas durante la inicialización.

De esta manera, un mensaje publicado con una determinada `routing_key` puede ser recibido por los consumidores que estén vinculados utilizando esa clave.

Esto permite implementar tanto comunicación dirigida mediante routing keys como escenarios en los que varios consumidores reciben los mensajes correspondientes a una misma clave.

---

## 5. Encapsulamiento de RabbitMQ

Uno de los objetivos principales del trabajo es respetar la abstracción de la interfaz proporcionada.

El código que utiliza:

```python
MessageMiddlewareQueueRabbitMQ
```

o:

```python
MessageMiddlewareExchangeRabbitMQ
```

no necesita conocer detalles de `pika`, conexiones TCP, canales de RabbitMQ ni las operaciones concretas utilizadas para publicar o consumir mensajes.

Los detalles específicos de RabbitMQ quedan encapsulados dentro de:

```text
middleware_rabbitmq.py
```

Esto permite separar:

* la interfaz que define el comportamiento esperado;
* la implementación concreta utilizando RabbitMQ.

Por lo tanto, la implementación sigue el principio de programación contra una interfaz y no contra una implementación concreta.

---

## 6. Manejo de errores

La interfaz provista por el trabajo práctico define excepciones específicas para representar distintos problemas del middleware.

La implementación traduce los errores provenientes de RabbitMQ/Pika a las excepciones definidas por la interfaz.

Entre los casos contemplados se encuentran:

* errores de desconexión;
* errores durante el procesamiento o comunicación con RabbitMQ;
* errores durante el cierre de la conexión.

De esta manera, los componentes que utilizan el middleware no dependen directamente de las excepciones específicas de `pika`.

Por ejemplo, ante una desconexión del broker, el error se representa mediante la excepción definida por la interfaz para una desconexión:

```text
MessageMiddlewareDisconnectedError
```

En caso de un error interno de comunicación o procesamiento se utiliza:

```text
MessageMiddlewareMessageError
```

Y ante un problema durante el cierre:

```text
MessageMiddlewareCloseError
```

Esto permite mantener encapsulados los detalles de la implementación de RabbitMQ.

---

## 7. ACK y NACK

El consumo de mensajes utiliza callbacks que reciben tres elementos:

```text
message
ack
nack
```

El callback puede procesar el mensaje y confirmar su procesamiento mediante `ack()`.

También puede utilizar `nack()` cuando el mensaje no pudo ser procesado correctamente.

El uso de ACK permite que el consumidor confirme explícitamente que recibió y procesó el mensaje.

Esto es importante para el comportamiento de las colas y para evitar considerar procesado un mensaje que todavía no fue confirmado.

---

## 8. Inicio y finalización del consumo

La interfaz proporciona:

```python
start_consuming(callback)
```

para comenzar a recibir mensajes.

El consumidor permanece procesando mensajes hasta que se solicita detener el consumo mediante:

```python
stop_consuming()
```

La implementación utiliza el mecanismo de consumo proporcionado por RabbitMQ y permite que el callback solicite la finalización del consumo.

También se implementa:

```python
close()
```

para cerrar correctamente la conexión con RabbitMQ.

Esto permite liberar los recursos utilizados por el middleware.

---

## 9. Uso de Docker

El trabajo se ejecuta utilizando Docker y Docker Compose.

El entorno proporcionado por el trabajo contiene:

* un contenedor para RabbitMQ;
* un contenedor para ejecutar las pruebas de Python.

Esto permite disponer de un entorno reproducible para ejecutar el middleware y las pruebas.

La ejecución de las pruebas se realiza mediante:

```bash
make test
```

El uso de Docker permite que la implementación no dependa directamente de una instalación local de RabbitMQ.

---

## 10. Pruebas realizadas

Se ejecutó la batería de pruebas proporcionada por el trabajo práctico utilizando:

```bash
make test
```

La ejecución creó correctamente los contenedores de RabbitMQ y de pruebas.

El resultado final fue:

```text
test_queue.py .............                                              [ 68%]
test_exchange.py ......                                                  [100%]

19 passed in 4.67s
```

Por lo tanto, las pruebas automáticas disponibles finalizaron correctamente, incluyendo las pruebas correspondientes a colas y exchanges.

Además, luego de finalizar las pruebas, Docker Compose detuvo y eliminó correctamente los contenedores y la red utilizada por el entorno de testing.

---

## 11. Decisiones de diseño

### 11.1 Uso de una cola por consumidor en los exchanges

Para implementar el comportamiento Publisher-Subscriber, cada consumidor dispone de una cola propia asociada al exchange.

Esto permite que diferentes consumidores puedan recibir independientemente los mensajes publicados en las routing keys a las que están suscriptos.

### 11.2 Exchange de tipo direct

Se utiliza un exchange `direct` porque el modelo requerido utiliza `routing_keys` para determinar qué consumidores deben recibir cada mensaje.

La clave de routing permite asociar los mensajes publicados con las colas que corresponden.

### 11.3 Uso de colas temporales para los consumidores de exchanges

Las colas utilizadas por los consumidores de los exchanges se crean como colas exclusivas y de eliminación automática.

Esto permite que la cola esté asociada al consumidor que la creó y que no permanezca innecesariamente en RabbitMQ una vez finalizada su utilización.

### 11.4 No utilización de funcionalidades específicas avanzadas de RabbitMQ

La implementación se mantiene sobre las funcionalidades básicas necesarias para cumplir la abstracción definida por el trabajo práctico.

No se utilizan mecanismos avanzados específicos de RabbitMQ que modifiquen la interfaz o agreguen una dependencia innecesaria de una funcionalidad particular del broker.

---

## 12. Cumplimiento de los requisitos

La implementación contempla los principales requisitos planteados:

| Requisito                    | Implementación                              |
| ---------------------------- | ------------------------------------------- |
| Interfaz simplificada de MOM | Implementada                                |
| Python                       | Implementado en Python                      |
| Producer-Consumer            | `MessageMiddlewareQueueRabbitMQ`            |
| Publisher-Subscriber         | `MessageMiddlewareExchangeRabbitMQ`         |
| RabbitMQ                     | Utilizado como broker                       |
| Encapsulamiento              | Detalles de `pika` ocultos en el middleware |
| Manejo de errores            | Excepciones propias de la interfaz          |
| ACK/NACK                     | Implementados                               |
| Inicio/detención del consumo | Implementados                               |
| Cierre de conexiones         | Implementado                                |
| Docker                       | Docker Compose                              |
| Pruebas automáticas          | 19/19 pruebas exitosas                      |

La consigna establece explícitamente que se debe mostrar conocimiento de los modelos Publisher-Subscriber y Producer-Consumer y respetar la abstracción de RabbitMQ junto con el encapsulamiento de sus errores.

---

## 13. Conclusión

Se implementó una interfaz simplificada de Middleware Orientado a Mensajes utilizando Python y RabbitMQ.

La solución permite trabajar con los modelos Producer-Consumer y Publisher-Subscriber mediante las abstracciones de cola y exchange proporcionadas por el trabajo práctico.

Los detalles específicos de RabbitMQ quedan encapsulados dentro de la implementación concreta del middleware, mientras que los errores del broker son transformados en las excepciones definidas por la interfaz.

Finalmente, la implementación fue validada mediante la batería de pruebas proporcionada, obteniendo:

```text
19 passed
```

El entorno de ejecución también fue probado mediante Docker Compose, permitiendo ejecutar RabbitMQ y las pruebas de forma reproducible.
