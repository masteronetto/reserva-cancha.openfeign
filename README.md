# reserva-cancha.openfeign
#Arquitectura OpenFeign entre serviciocanchas y servicioreservas
##  Objetivo

Construir integración de microservicios con:

- `serviciocanchas` (puerto `8081`)
- `servicioreservas` (puerto `8082`)
- Comunicación bidireccional usando Spring Cloud OpenFeign
- Fallback + Resilience4j para alta disponibilidad

---

##  Estructura FEIGN en servicioreservas

1) `ServicioreservasApplication.java`
   - `@SpringBootApplication`
   - `@EnableFeignClients`
   - `main(...)` arranca Spring Boot

2) `client/CanchaClient.java`
   - `@FeignClient(name = "canchas-service", url = "${canchas.service.url}", fallback = CanchaClientFallback.class)`
   - `GET /api/canchas/{id}`
   - `GET /api/canchas`

3) `client/CanchaClientFallback.java`
   - `@Component`
   - `obtenerCancha(...) -> null`
   - `listarCanchas() -> Collections.emptyList()`

4) `services/ReservaServicesImpl.java`
   - inyector `@Autowired private CanchaClient canchaClient;`
   - `crearReserva()` valida cancha antes de guardar (uso FEIGN)
   - `listarCanchas()` devuelve datos via FEIGN

5) `application.properties`
   - `canchas.service.url=http://localhost:8081`
   - `feign.circuitbreaker.enabled=true`
   - `spring.cloud.circuitbreaker.resilience4j.enabled=true`

---

## Estructura FEIGN en serviciocanchas

1) `ServiciocanchasApplication.java`
   - `@SpringBootApplication`
   - `@EnableFeignClients`
   - `main(...)`

2) `client/ReservaClient.java`
   - `@FeignClient(name = "reservas-service", url = "${reservas.service.url}", fallback = ReservaClientFallback.class)`
   - `GET /api/reservas/cancha/{canchaId}`

3) `client/ReservaClientFallback.java`
   - `@Component`
   - `obtenerReservasPorCancha(...) -> Collections.emptyList()`

4) `services/ServicesImpl.java`
   - inyector `@Autowired private ReservaClient reservaClient;`
   - `obtenerReservasPorCancha(canchaId)` llama a `reservaClient.obtenerReservasPorCancha(canchaId)`

5) `application.properties`
   - `reservas.service.url=http://localhost:8082`

---

## Endpoints expuestos

### serviciocanchas

- `GET /api/canchas`
- `POST /api/canchas`
- `GET /api/canchas/{id}`
- `GET /api/canchas/{id}/reservas` (FEIGN a reservas)

### servicioreservas

- `GET /api/reservas`
- `POST /api/reservas`
- `GET /api/reservas/cancha/{id}`
- `GET /api/reservas/canchas` (FEIGN a canchas)

---

## Dependencias clave (pom.xml)

- `spring-cloud-starter-openfeign`
- `spring-cloud-starter-circuitbreaker-resilience4j`
- `spring-boot-starter-web`
- `spring-boot-starter-data-jpa`
- `mysql-connector-j`

---

## Flujo de validación (Postman)

1. `GET http://localhost:8081/api/canchas`
2. `POST http://localhost:8081/api/canchas`
3. `GET http://localhost:8082/api/reservas`
4. `POST http://localhost:8082/api/reservas` (with `canchaId`)
5. `GET http://localhost:8081/api/canchas/{id}/reservas` (usando FEIGN a reservas)
6. `GET http://localhost:8082/api/reservas/canchas` (usando FEIGN a canchas)

---

## Notes

- Archivo `pom.xml` se ajustó para Spring Boot `3.2.12` + Spring Cloud `2023.0.5` (evita incompatibilidades).
- `spring-boot-maven-plugin` con `mainClass` correcta.
- Configuración FEIGN con fallback y circuit-breaker para evitar caídas en cascada.
