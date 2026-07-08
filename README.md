# Sigle RedNorte — Eureka Server

Servidor de *service discovery* para el ecosistema de microservicios de **Sigle RedNorte**. Cada microservicio (CoreService, CitasService, ListasService, PacientesService) se registra aquí al arrancar, y el `ApiGateway` lo consulta para saber a qué instancia enrutar cada petición — sin tener que hardcodear URLs entre servicios.

## Stack

- Java 17
- Spring Boot 4.1.0
- Spring Cloud 2025.1.2 (`spring-cloud-starter-netflix-eureka-server`)

## Servicios que se registran acá

| Servicio | Repositorio |
|---|---|
| API Gateway | `sigle--ApiGateway` |
| Core Service (auth, roles, establecimientos, dashboard) | `sigle-CoreService` |
| Citas Service | `sigle-CitasService` |
| Listas Service (listas de espera, pacientes) | `sigle-ListasService` |
| Pacientes Service (notificaciones, email) | `sigle-PacientesService` |

## Cómo correr en local

```bash
./mvnw spring-boot:run
```

Por defecto levanta en el puerto `8761`. El panel de Eureka queda disponible en:
http://localhost:8761

Ahí puedes ver qué servicios están registrados y su estado (`UP`, `DOWN`, etc.) en tiempo real.

## Variables de entorno

| Variable | Default | Descripción |
|---|---|---|
| `PORT` | `8761` | Puerto en el que escucha el servidor |
| `EUREKA_INSTANCE_HOSTNAME` | `localhost` | Hostname público con el que este Eureka se anuncia a sí mismo. **En despliegues remotos (Render, etc.) debe ser el dominio público real**, no `localhost` |

## Configuración relevante (`application.yml`)

```yaml
eureka:
  client:
    register-with-eureka: false   # Eureka no se registra a sí mismo
    fetch-registry: false         # ni descarga el registro de otro Eureka
  server:
    enable-self-preservation: false
```

`enable-self-preservation: false` hace que Eureka elimine instancias que dejaron de mandar heartbeat, en vez de asumir que es un problema de red y conservarlas igual. Para este proyecto (pocos servicios, entorno con reinicios frecuentes en Render) conviene así — si algún día hay muchos más servicios y la red es inestable, vale la pena revisar si conviene reactivarlo.

## Despliegue en Render

Este servicio corre en Render en:
https://sigle-eurekaserver.onrender.com

Variables a configurar en el dashboard de Render:
EUREKA_INSTANCE_HOSTNAME=sigle-eurekaserver.onrender.com

**Importante para los otros microservicios:** Render solo expone tráfico público por HTTPS en el puerto `443` — el puerto interno de cada app (el que asigna `PORT`) no es alcanzable desde afuera. Por eso, cualquier servicio que se registre acá estando desplegado en Render debe anunciarse con el puerto seguro `443` en vez de su puerto interno, o el `ApiGateway` no va a poder enrutarle tráfico (se cae por timeout). Los microservicios Node (`CoreService`, `CitasService`, `ListasService`, `PacientesService`) ya tienen esta lógica resuelta en su `src/config/eureka.js`.

## Docker

```bash
docker build -t sigle-eureka-server .
docker run -p 8761:10000 -e PORT=10000 sigle-eureka-server
```

## Notas

- Este servicio no tiene lógica de negocio ni base de datos — es infraestructura pura.
- Debe ser el **primer** servicio en levantar (o al menos, estar arriba) antes que el resto, ya que todos dependen de él para descubrirse entre sí.
