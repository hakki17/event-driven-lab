# Laboratorio: Event-Driven con Spring Boot, RabbitMQ y Docker Compose

> Laboratorio de arquitectura orientada a eventos usando dos microservicios Spring Boot que se comunican a través de RabbitMQ, orquestados con Docker Compose y desplegados desde GitHub Codespaces.

---

## Objetivo

> Crear un servicio Productor que expone un endpoint REST para enviar mensajes, y un servicio Consumidor que escucha y procesa esos mensajes a través de una cola RabbitMQ, todo ejecutado en contenedores Docker.

---

## Tecnologías utilizadas

> - Java 17
> - Spring Boot 3.5.0
> - Spring AMQP (RabbitMQ)
> - Docker y Docker Compose
> - GitHub Codespaces
> - Maven

---

## Estructura del repositorio

```
event-driven-lab/
├── .devcontainer/
│   └── devcontainer.json
├── producer-service/
│   ├── src/main/java/com/eci/arcn/producer_service/
│   │   ├── config/RabbitMQConfig.java
│   │   ├── controller/MessageController.java
│   │   └── ProducerServiceApplication.java
│   ├── src/main/resources/application.properties
│   ├── pom.xml
│   └── Dockerfile
├── consumer-service/
│   ├── src/main/java/com/eci/arcn/consumer_service/
│   │   ├── config/RabbitMQConfig.java
│   │   ├── listener/MessageListener.java
│   │   └── ConsumerServiceApplication.java
│   ├── src/main/resources/application.properties
│   ├── pom.xml
│   └── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## Arquitectura

> El Productor expone un endpoint `POST /api/messages/send` que recibe un mensaje por parámetro y lo publica en un `DirectExchange` de RabbitMQ usando una routing key. El Consumidor escucha la cola `messages.queue` con `@RabbitListener` y procesa cada mensaje que llega. RabbitMQ actúa como broker intermedio, desacoplando completamente los dos servicios.

```
[Cliente HTTP]
      |
      | POST /api/messages/send?message=Hola
      v
[Producer Service :8080]
      |
      | convertAndSend(exchange, routingKey, message)
      v
[RabbitMQ :5672]
  messages.exchange
      |
      | routing key: messages.routingkey
      v
  messages.queue
      |
      v
[Consumer Service]
  @RabbitListener
      |
      v
  Mensaje procesado en consola
```

---

## Paso 1: Configurar GitHub Codespaces

> Se configuró un entorno de desarrollo en la nube usando GitHub Codespaces con una imagen base de Java 17, Maven 3.8.6 y Docker-in-Docker habilitado mediante el archivo `.devcontainer/devcontainer.json`.

---

## Paso 2: Servicio Productor

> El productor es un microservicio Spring Boot con dependencias `spring-web` y `spring-amqp`. Declara la cola, el exchange y el binding en `RabbitMQConfig`, y expone un endpoint REST en `MessageController` que usa `RabbitTemplate` para publicar mensajes.

Configuración principal (`application.properties`):

```properties
server.port=8080
spring.rabbitmq.host=rabbitmq
app.rabbitmq.exchange=messages.exchange
app.rabbitmq.queue=messages.queue
app.rabbitmq.routingkey=messages.routingkey
```

---

## Paso 3: Servicio Consumidor

> El consumidor es un microservicio Spring Boot con dependencia `spring-amqp`. Usa la anotación `@RabbitListener` en `MessageListener` para suscribirse a la cola y procesar cada mensaje entrante de forma automática.

Configuración principal (`application.properties`):

```properties
spring.rabbitmq.host=rabbitmq
app.rabbitmq.queue=messages.queue
```

---

## Paso 4: Imágenes en Docker Hub

> Cada servicio fue compilado con Maven, empaquetado en una imagen Docker usando `eclipse-temurin:17-jre-jammy` como imagen base, y publicado en Docker Hub.

Comandos usados:

```bash
mvn package -DskipTests
docker build -t producer-service .
docker tag producer-service hakki17/producer-service
docker push hakki17/producer-service
```

> Las imágenes quedan disponibles públicamente en:
> - `hakki17/producer-service`
> - `hakki17/consumer-service`

---

## Paso 5: Docker Compose

> Se definió un archivo `docker-compose.yml` con tres servicios en la misma red `event_network`: RabbitMQ con la imagen `rabbitmq:management`, el productor mapeado al puerto 8080, y el consumidor sin puerto expuesto. Ambos servicios reciben la configuración de conexión a RabbitMQ por variables de entorno.

---

## Paso 6: Ejecución y prueba

> Los servicios se levantaron con `docker-compose up -d` desde la raíz del proyecto. Se verificó el funcionamiento enviando un mensaje mediante `curl` y comprobando los logs del consumidor.

Comando para enviar un mensaje:

```bash
curl -X POST "http://localhost:8080/api/messages/send?message=HolaMundo"
```

Respuesta esperada:

```
Mensaje 'HolaMundo' enviado!
```

Verificación en logs del consumidor:

```bash
docker-compose logs consumer
```

---

## Evidencia

### Cola messages.queue en RabbitMQ Management UI

> Se accedió a la interfaz de administración de RabbitMQ en el puerto 15672. La cola `messages.queue` aparece en estado `running`, confirmando que fue creada correctamente por los servicios al iniciar.

![RabbitMQ Management UI - Cola messages.queue](rabbitmq-queue.png)

### Mensaje procesado por el consumidor

> El log del consumidor confirma que el mensaje enviado por el productor fue recibido y procesado correctamente a través de la cola RabbitMQ.

![Consumer logs - Mensaje procesado](consumer-logs.png)

---

## Comandos de referencia

> Levantar los servicios:
```bash
docker-compose up -d
```

> Ver estado de los contenedores:
```bash
docker-compose ps
```

> Ver logs en tiempo real:
```bash
docker-compose logs -f
```

> Detener los servicios:
```bash
docker-compose down
```