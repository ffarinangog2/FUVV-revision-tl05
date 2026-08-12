# Issues a publicar en https://github.com/carlospatroner-boop/PFC-SOPORTE-ISP

Todos sobre el commit auditado `479b637`. Reparto: 6 para Carlos Carpio, 6 para Robinson Cando.

> **Antes de publicar:** sustituir `@carpio` y `@cando` por los usuarios reales de GitHub, y
> confirmar que `2026-08-14` es anterior al cierre de la Semana 17.

---

## H01 · Extraer la publicación de eventos y la política de autorización de TicketService (SRP)

**Labels:** `solid`, `mayor` · **Assignee:** @carpio · **Fecha de resolución:** 2026-08-14

### Contexto
- Sistema revisado: Soporte Técnico ISP (equipo ACC)
- Commit auditado: `479b637`
- Bloque e ítem: A1 (SRP)

### Evidencia
- Archivo: `services/svc-principal/src/main/java/ec/edu/uteq/soporte/ticketservice/service/TicketService.java`
- Clase: `TicketService`
- Líneas: 51–64 (colaboradores), 99–128 (publicación de eventos)

```java
private final TicketRepository ticketRepository;
private final KafkaTemplate<String, String> kafkaTemplate;
private final ObjectMapper objectMapper;
private final CrdbMetrics crdbMetrics;
```

### Diagnóstico
- Principio afectado: **SRP**
- Severidad: **Mayor**
- Atributo de calidad: **Mantenibilidad**
- Consecuencia: la clase tiene cuatro razones distintas para cambiar (regla de negocio,
  autorización, mensajería, métricas). Ninguna prueba unitaria puede ejercitar la regla de
  negocio sin simular también `KafkaTemplate`.

### Propuesta de corrección
1. Extraer `TicketEventPublisher` con los tres métodos `publishTicket*`, recibiendo
   `KafkaTemplate` y `ObjectMapper`.
2. Extraer `TicketAuthorizationPolicy` con `assertCanView` y `assertCanManage`.
3. Dejar `TicketService` como coordinador de casos de uso, dependiendo de ambas abstracciones.

### Criterio de aceptación
`TicketService` no importa `KafkaTemplate` ni `ObjectMapper`, y existe una prueba de
`TicketEventPublisher` independiente de la lógica de tickets.

---

## H02 · Centralizar la matriz de roles en una política sustituible (OCP)

**Labels:** `solid`, `mayor` · **Assignee:** @cando · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: A2 (OCP)

### Evidencia
- Archivo: `services/svc-principal/.../service/TicketService.java`
- Métodos: `createTicket` (74), `listTickets` (141–169), `assertCanView` (225–239),
  `assertCanManage` (243–254)

```java
if (ROLE_ADMIN.equals(role)) { ... }
if (ROLE_TECNICO.equals(role)) { ... }
if (ROLE_CLIENTE.equals(role)) { ... }
```

### Diagnóstico
- Principio afectado: **OCP**
- Severidad: **Mayor**
- Atributo de calidad: **Mantenibilidad**
- Consecuencia: añadir un rol (p. ej. `SUPERVISOR`) obliga a editar cuatro métodos ya
  probados. Olvidar uno produce una política inconsistente: un rol que puede ver un ticket
  que no puede gestionar, o al revés.

### Propuesta de corrección
1. Trasladar la matriz documentada en el comentario de clase (líneas 23–39) a una estructura
   de datos o a un conjunto de implementaciones de `RolePolicy`.
2. Resolver la política por rol y consultarla en un único punto de decisión.

### Criterio de aceptación
Añadir un cuarto rol requiere crear una clase nueva y registrarla, sin modificar los métodos
existentes de `TicketService`. Existe una prueba parametrizada que cubre la matriz completa
rol × operación.

---

## H03 · Separar la entidad JPA del modelo de dominio Ticket (DIP)

**Labels:** `solid`, `menor` · **Assignee:** @carpio · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: A5 (DIP)

### Evidencia
- Archivo: `services/svc-principal/.../domain/Ticket.java`, líneas 3–9 y 26–34

```java
import jakarta.persistence.Entity;
import jakarta.persistence.IdClass;

@Entity
@Table(name = "tickets")
@IdClass(TicketId.class)
public class Ticket {
```

### Diagnóstico
- Principio afectado: **DIP**
- Severidad: **Menor**
- Atributo de calidad: **Mantenibilidad**
- Consecuencia: si se modifica la estrategia de fragmentación descrita en ADR-0003, las
  anotaciones `@Id` sobre `created_at` obligan a tocar el modelo de negocio para resolver un
  problema puramente de persistencia.

### Propuesta de corrección
1. Crear `TicketEntity` en `repository`/infraestructura con las anotaciones JPA.
2. Dejar `Ticket` como clase de dominio sin anotaciones.
3. Añadir un mapeador entre ambas en la capa de repositorio.

### Criterio de aceptación
`grep -r "jakarta.persistence" src/main/java/**/domain/` no devuelve resultados.

---

## H04 · Acotar TicketRepository a las operaciones realmente usadas (ISP)

**Labels:** `solid`, `menor` · **Assignee:** @cando · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: A4 (ISP)

### Evidencia
- Archivo: `services/svc-principal/.../repository/TicketRepository.java`, líneas 15–23

```java
public interface TicketRepository extends JpaRepository<Ticket, TicketId> {
    // findById(TicketId) heredado exige ambos componentes de la PK,
    // que el llamador normalmente no tiene disponibles.
    @Query("select t from Ticket t where t.id = :id")
    Optional<Ticket> findByTicketId(@Param("id") UUID id);
```

### Diagnóstico
- Principio afectado: **ISP**
- Severidad: **Menor**
- Atributo de calidad: **Mantenibilidad**, **Seguridad**
- Consecuencia: se heredan ~20 métodos, incluidos `deleteAll` y `deleteAllInBatch`, que
  ningún cliente usa y que quedan disponibles por accidente. El propio código advierte por
  comentario cuál de los métodos heredados no debe usarse.

### Propuesta de corrección
1. Sustituir `JpaRepository` por `Repository<Ticket, TicketId>` declarando solo los métodos
   necesarios (`save`, `findByTicketId` y las cinco consultas por criterio).
2. Alternativa mínima: definir una interfaz propia `TicketStore` que el repositorio implemente
   y que sea la que reciba `TicketService`.

### Criterio de aceptación
`TicketService` depende de una interfaz cuyos métodos son todos invocados por al menos un caso
de uso, y no existe ningún método de borrado accesible desde la capa de servicio.

---

## H05 · Introducir una puerta de enlace única y eliminar el filtro duplicado

**Labels:** `patrones`, `mayor` · **Assignee:** @carpio · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: B4 y B5 (patrones y antipatrones distribuidos)

### Evidencia
- Archivo: `frontend/app.js`, líneas 5–7

```javascript
const API_BASE = "http://localhost:8002/api/v1";
const AUTH_API_BASE = "http://localhost:8001/api/v1/auth";
const REPORTS_API_BASE = "http://localhost:8005/api/v1/reports";
```

- Archivos: `services/svc-principal/.../config/AuthGatewayFilter.java` y
  `services/report-service/.../config/AuthGatewayFilter.java` — dos clases prácticamente
  idénticas (ambas crean el cliente en la línea 39–40 y capturan en la 76–77).

### Diagnóstico
- Antipatrón afectado: **ausencia de puerta de enlace**; **cadena síncrona de llamadas**
- Severidad: **Mayor**
- Atributo de calidad: **Mantenibilidad**, **Disponibilidad**
- Consecuencia: el navegador conoce tres servicios; cualquier cambio de puerto o de esquema de
  despliegue obliga a editar el cliente. La política transversal de autenticación se mantiene
  por duplicado en dos servicios.

### Propuesta de corrección
1. Introducir una puerta de enlace (Spring Cloud Gateway o Nginx) como único punto de entrada.
2. Mover allí la validación de token y eliminar uno de los dos `AuthGatewayFilter`.
3. Dejar en el frontend una sola URL base.

### Criterio de aceptación
`frontend/app.js` declara una única constante de URL base y existe un solo componente que
valide el token en todo el sistema.

---

## H06 · Sustituir el DTO web por un comando de aplicación en la capa de servicio

**Labels:** `capas`, `menor` · **Assignee:** @cando · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: C2 (dirección de dependencias)

### Evidencia
- `services/svc-principal/.../service/TicketService.java`, línea 12
- `services/auth-service/.../service/AuthService.java`, líneas 14–18 (cinco imports de `web.dto`)

```java
import ec.edu.uteq.soporte.ticketservice.web.dto.CreateTicketRequest;

public Ticket createTicket(CreateTicketRequest request, UUID clientId, String role)
```

### Diagnóstico
- Severidad: **Menor** · Atributo: **Mantenibilidad**
- Consecuencia: un cambio en el contrato HTTP (renombrar un campo del formulario) se propaga
  hasta la capa de aplicación, que no debería conocer el transporte.

### Propuesta de corrección
1. Crear `CreateTicketCommand` en el paquete `service`.
2. Convertir `CreateTicketRequest → CreateTicketCommand` en `TicketController`.
3. Repetir el patrón en `AuthService`.

### Criterio de aceptación
`grep -r "web.dto" src/main/java/**/service/` no devuelve resultados en ninguno de los dos
servicios.

---

## H07 · Eliminar el secreto JWT y la contraseña de administrador por defecto

**Labels:** `capas`, `mayor` · **Assignee:** @carpio · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: C5 (configuración externalizada)

### Evidencia
- Archivo: `services/auth-service/src/main/resources/application.yml`, líneas 33 y 42–43

```yaml
secret: ${AUTH_JWT_SECRET:dev-only-insecure-secret-please-change-me-3242}
email: ${ADMIN_BOOTSTRAP_EMAIL:admin@soporte.local}
password: ${ADMIN_BOOTSTRAP_PASSWORD:Admin123!}
```

### Diagnóstico
- Severidad: **Mayor** · Atributo: **Seguridad**
- Consecuencia: si la variable de entorno no se define, el sistema arranca con un secreto de
  firma publicado en el repositorio y una cuenta administrativa de contraseña conocida. Un
  despliegue con una variable mal escrita no produce ningún error visible.

### Propuesta de corrección
1. Suprimir los valores de reserva: dejar `${AUTH_JWT_SECRET}` y `${ADMIN_BOOTSTRAP_PASSWORD}`
   sin valor por omisión.
2. Añadir validación de arranque que falle si el secreto tiene menos de 32 bytes.
3. Rotar el secreto actual, que ya está expuesto en el historial.

### Criterio de aceptación
La aplicación no inicia sin `AUTH_JWT_SECRET` ni `ADMIN_BOOTSTRAP_PASSWORD`, y ningún secreto
funcional permanece versionado.

---

## H08 · Externalizar las URL de los servicios en el frontend

**Labels:** `capas`, `menor` · **Assignee:** @cando · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: C5

### Evidencia
- `frontend/app.js`, líneas 5–7 y `frontend/auth/auth.js`, línea 8: `http://localhost:8001`
  aparece cableado en dos archivos distintos.

### Diagnóstico
- Severidad: **Menor** · Atributo: **Portabilidad**
- Consecuencia: el frontend solo funciona en local. Desplegarlo exige editar el código fuente,
  y la URL de auth está duplicada en dos ficheros que pueden divergir.

### Propuesta de corrección
1. Centralizar las URL en un `config.js` único.
2. Poblarlo en tiempo de despliegue (variable de entorno o fichero de configuración servido).

### Criterio de aceptación
Ningún archivo bajo `frontend/` contiene la cadena `localhost:`.

---

## H09 · Configurar tiempo de espera y respuesta degradada al validar el token

**Labels:** `excepciones`, `critica` · **Assignee:** @carpio · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: D2 y D4

### Evidencia
- Archivo: `services/svc-principal/.../config/AuthGatewayFilter.java`, líneas 40, 60–65, 76–78

```java
this.restClient = RestClient.create();
...
ApiResponse<ValidateResponse> validated = restClient.get()
        .uri(validateUrl).header("Authorization", header)
        .retrieve().body(...);
...
} catch (RestClientException e) {
    writeUnauthorized(response, "Token invalido o auth-service no disponible: " + e.getMessage());
}
```

### Diagnóstico
- Severidad: **Crítica** · Atributo: **Fiabilidad**, **Disponibilidad**
- Consecuencia: `RestClient.create()` no fija tiempo de espera de lectura. El filtro corre en
  **cada** petición a `/api/v1/tickets/**`, de modo que un `auth-service` lento —no caído, sino
  lento— retiene un hilo del contenedor por petición hasta agotar el pool. Además, un fallo de
  infraestructura se comunica al abonado como un 401, es decir, como si sus credenciales
  fueran inválidas.

### Propuesta de corrección
1. Construir el `RestClient` con `connectTimeout` y `readTimeout` explícitos (p. ej. 2 s y 3 s).
2. Distinguir `HttpClientErrorException.Unauthorized` (401 real) de `ResourceAccessException`
   (503 con mensaje de servicio no disponible).
3. Añadir un disyuntor (Resilience4j) sobre la llamada.

### Criterio de aceptación
Una prueba con `auth-service` detenido termina dentro del tiempo de espera configurado y
devuelve **503**, no 401. El mismo arreglo se aplica a `report-service`.

---

## H10 · Dejar de exponer ex.getMessage() en las respuestas de error

**Labels:** `excepciones`, `mayor` · **Assignee:** @cando · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: D5

### Evidencia
- Archivo: `services/svc-principal/.../web/GlobalExceptionHandler.java`, líneas 37–41

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiResponse<Object>> handleGeneric(Exception ex) {
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.of(null, "Error interno: " + ex.getMessage()));
}
```

También en `AuthGatewayFilter.java`, línea 77.

### Diagnóstico
- Severidad: **Mayor** · Atributo: **Seguridad**
- Consecuencia: el mensaje de una excepción no controlada suele contener el nombre de la
  tabla, la sentencia SQL o la clase del driver. El formato uniforme entre servicios sí se
  cumple; lo que falla es la fuga de detalle interno.

### Propuesta de corrección
1. Registrar la excepción completa en servidor con `LOGGER.log(Level.SEVERE, ..., ex)`.
2. Devolver un mensaje genérico más un identificador de incidencia correlacionable.
3. Aplicar lo mismo en los tres `GlobalExceptionHandler`.

### Criterio de aceptación
Una excepción inesperada devuelve 500 sin traza, sin SQL, sin nombres internos y sin
`ex.getMessage()`, pero el detalle sí aparece en el log del servidor.

---

## H11 · Generar y propagar un identificador de correlación; unificar el registro

**Labels:** `observabilidad`, `mayor` · **Assignee:** @carpio · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: E1, E2 y E3

### Evidencia
- Archivo: `services/notification-service/src/kafkaConsumer.js`, líneas 51 y 56

```javascript
console.error(`No se pudo procesar un mensaje de ${topic}:`, err);
console.log(`notification-service escuchando: ${...}`);
```

También `index.js`, líneas 23 y 27. Ningún filtro HTTP ni productor de Kafka genera o propaga
cabecera de correlación.

### Diagnóstico
- Severidad: **Mayor** · Atributo: **Observabilidad / Diagnóstico**
- Consecuencia: una creación de ticket atraviesa frontend → svc-principal → auth-service →
  Kafka → ai-service / report-service / notification-service, y no existe ningún identificador
  común que permita reconstruir el recorrido. El único nexo es el `ticketId`, que ni siquiera
  aparece en los mensajes de Node.

### Propuesta de corrección
1. Sustituir `console.*` por `pino` en `notification-service`.
2. Generar `X-Correlation-ID` en el borde si no viene, propagarlo en cada llamada HTTP y como
   header de cada mensaje Kafka.
3. Incorporarlo al MDC en los servicios Java y emitir los logs en JSON con campos `service`,
   `operation`, `correlationId`.

### Criterio de aceptación
Una creación de ticket puede localizarse en los logs de los tres servicios participantes
filtrando por el mismo `correlationId`.

---

## H12 · Sanitizar los payloads registrados y auditar los eventos de autenticación

**Labels:** `observabilidad`, `mayor` · **Assignee:** @cando · **Fecha de resolución:** 2026-08-14

### Contexto
Commit `479b637` · Bloque e ítem: E4 y E5

### Evidencia
- Archivo: `services/report-service/.../config/ReportEventListener.java`, línea 82 (y 95, 105, 118)

```java
LOGGER.log(Level.WARNING, "No se pudo procesar un mensaje de ticket.created: " + payload, e);
```

`ticket.created` transporta `ticketId`, `zone`, `clientId`, `description` y `createdAt`.
El mismo patrón está en `services/ai-service/app/kafka_consumer.py`, línea 93. Además, ambos
`application.yml` fijan `show-sql: true` y `org.hibernate.SQL: DEBUG`.

- `services/auth-service/.../service/AuthService.java`, líneas 34–54: la clase implementa
  `login`, `refresh`, `logout`, `createUserAsAdmin` y la detección de reutilización de token
  (`TokenReuseDetectedException`), pero **no declara ningún `Logger`** ni recibe componente de
  auditoría.

### Diagnóstico
- Severidad: **Mayor** · Atributo: **Seguridad**, **Auditabilidad**
- Consecuencia: ante un mensaje que falla, el identificador del abonado y el texto libre que
  escribió quedan íntegros en el log. Y si se detecta la reutilización de un token de refresco,
  el sistema revoca la sesión sin dejar constancia de qué usuario, cuándo ni con qué resultado.

### Propuesta de corrección
1. Registrar solo `ticketId` y la clase de error, nunca el payload completo.
2. Bajar `org.hibernate.SQL` a `INFO` fuera de desarrollo.
3. Añadir una tabla/flujo de auditoría con: evento, usuario, marca de tiempo, IP y resultado,
   para login exitoso y fallido, logout, refresh, reutilización de token y altas
   administrativas.

### Criterio de aceptación
Los logs no contienen descripciones de ticket ni `clientId` innecesarios, y existe un registro
de auditoría consultable para los cinco eventos críticos de autenticación.
