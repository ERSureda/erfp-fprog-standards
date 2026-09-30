# Especificación Técnica de la Semilla Base — [<PATRÓN_O_LENGUAJE>]

> **Naturaleza del documento.** Especificación técnica normativa y Fuente Única de Verdad (SSOT) del repositorio base ejecutable para el Preset `[<NOMBRE_DEL_PRESET>]`. Define el código transversal que se construye una sola vez, la delimitación estricta frente a los generadores de negocio, la suite de verificación automatizada y los criterios de aceptación ejecutables para cualquier proyecto que clone esta semilla.
>
> **Documentos normativos asociados.** `ARCHITECTURE.md`, `EXCEPTIONS.md` y `TESTING.md`. Toda regla y convención técnica citada en esta especificación deriva directamente de dichos manuales.
>
> **Fuerza normativa.** Las reglas etiquetadas como `MUST` son obligaciones técnicas estrictas cuyo incumplimiento bloquea el pipeline de integración continua o la aprobación de Pull Requests. Las reglas `NEVER` son prohibiciones taxativas que solo admiten excepciones mediante un registro de decisión formal (`ADR`) con condición de salida escrita.

---

## 1. Metadatos de Gobernanza y Vinculación Normativa

* **Preset / Ecosistema:** `[<NOMBRE_DEL_PRESET> — Ej: Java Spring Modulith / Python FastAPI / Node.js Clean Architecture]`
* **Versión del Estándar:** `v1.0.0`
* **Estado:** `[Propuesta / Normativo / En Deprecación]`
* **Repositorio Semilla Asociado:** `[<URL_O_NOMBRE_DE_LA_SEMILLA_EJECUTABLE>]`
* **Documentos Normativos Vinculados:**
  * Directrices de arquitectura y fronteras: [`docs/standards/ARCHITECTURE.md`](docs/standards/ARCHITECTURE.md)
  * Modelo y catálogo de excepciones: [`docs/standards/EXCEPTIONS.md`](docs/standards/EXCEPTIONS.md)
  * Estrategia de pruebas y verificación: [`docs/standards/TESTING.md`](docs/standards/TESTING.md)
* **Alcance:** Obligatorio para el mantenimiento del repositorio semilla y para todo nuevo servicio generado a partir de esta base técnica.

---

## 2. Delimitación de Fronteras: Semilla vs. Código de Negocio

El repositorio semilla contiene exclusivamente **el código transversal, agnóstico a subdominios específicos y de configuración técnica** que compila y se ejecuta de forma autónoma en el pipeline de integración continua:

| Componente / Artefacto | Responsable | Justificación Técnica |
| --- | --- | --- |
| **Kernel Compartido (`shared/`)** | **Semilla** | Invariable entre módulos; define tipos base de dominio, contexto de seguridad, envolturas de paginación, puertos de salida y jerarquía de excepciones. |
| **Maquinaria Asíncrona (Outbox Relay & DLQ)** | **Semilla** | Infraestructura de publicación atómica de eventos, reintentos con backoff, recuperación de eventos incompletos y purga periódica. |
| **Filtros de Contexto y Gateway Offloading** | **Semilla** | Reconstrucción perimetral de identidad, propagación de contexto (`ExecutionContext`) e inyección de correlación en logging. |
| **Línea Base de Migraciones (DDL Baseline)** | **Semilla** | Scripts iniciales de base de datos que crean extensiones, funciones auxiliares y la tabla `outbox_events`. |
| **Guardianes CI (Fitness Functions)** | **Semilla** | Suite de análisis estático estructural y pruebas de arquitectura que bloquean el build ante violaciones normativas. |
| **Módulo Canónico de Referencia** | **Semilla** | Slice vertical mínimo funcional para que los tests de arquitectura verifiquen código real compilado. |
| **Toolchain y Dockerfile Multi-Stage** | **Semilla** | Configuración base del runtime, perfiles de ejecución, virtual threads/asincronía y empaquetado seguro sin privilegios. |
| **Agregados, Entidades y Value Objects** | Desarrollador / Generador | Específico del modelo de negocio de cada Bounded Context. |
| **Casos de Uso (`*UseCase`) y Servicios** | Desarrollador / Generador | Lógica de orquestación propia de cada intención funcional. |
| **Persistencia de Dominio (ORM, SQL, Mappers)** | Desarrollador / Generador | Adaptadores acoplados al esquema de base de datos de cada subdominio. |
| **Controladores REST y DTOs de Transporte** | Desarrollador / Generador | Contratos de interfaz específicos definidos en `ENDPOINTS.md`. |
| **Migraciones de Negocio por Esquema** | Desarrollador / Generador | Evolución DDL de las tablas propias del subdominio modeladas en `DATABASE.md`. |

---

## 3. Catálogo Normativo del Repositorio Base (`SED`)

### 3.1 Identificadores de Regla

La notación sigue la estructura unívoca: **`SED-nn · FUERZA [TIPO]`**.

* **`MUST`**: Obligación técnica estricta.
* **`NEVER`**: Prohibición técnica absoluta.
* **`[A]`**: Verificación automatizada obligatoria en el pipeline de CI.
* **`[R]`**: Verificación manual obligatoria en Pull Request mediante checklist.

### 3.2 Reglas de la Semilla

* **`SED-01 · NEVER` [A]** El paquete o namespace transversal `shared/` contendrá lógica, dependencias o tipos acoplados a ningún subdominio de negocio específico.
* **`SED-02 · MUST` [A]** Todos los identificadores únicos globales del sistema deben generarse en la aplicación con formato secuencial en el tiempo conforme a RFC 9562 (**UUIDv7**) mediante una abstracción `UuidGeneratorPort` respaldada por un generador libre de bloqueos (*lock-free CAS*).
* **`SED-03 · MUST` [A]** El contexto de ejecución (`ExecutionContext`) debe implementarse como una estructura inmutable y con seguridad de subprocesos (*thread-safe*), compatible con hilos virtuales o arquitecturas asíncronas y con limpieza obligatoria en bloques `finally` perimetrales.
* **`SED-04 · MUST` [A]** La semilla debe incluir obligatoriamente un módulo canónico de referencia (`<subdominio-referencia>`) que compile y ejecute los 3 flujos de operación (Command, Query CQRS y Worker con Idempotencia).
* **`SED-05 · MUST` [A]** La maquinaria de Transactional Outbox debe incluir sondeo no contencioso con bloqueo pesimista (`FOR UPDATE SKIP LOCKED`), backoff exponencial ante fallos y proceso de purga periódica programada de registros entregados.
* **`SED-06 · MUST` [A]** La semilla debe incorporar de serie un motor de migraciones versionadas (Flyway o equivalente) con su migración inicial baseline para la infraestructura compartida.
* **`SED-07 · MUST` [A]** La aplicación debe implementar parada limpia (*Graceful Shutdown*) certificando que los workers de outbox finalizan las transacciones en curso antes de liberar conexiones y terminar el proceso.
* **`SED-08 · MUST` [A]** El build completo del repositorio base debe compilar en verde sin advertencias, pasando el 100% de los tests unitarios, de integración y las aserciones de los guardianes de arquitectura.

---

## 4. Inventario Técnico del Kernel Compartido (`shared/`)

Configurado bajo el paquete transversal `<namespace.base>.shared` y declarado como módulo abierto para consumo de todos los subdominios de negocio.

### 4.1 Dominio Base (`shared.domain`)

* **`AggregateRoot<ID>`:** Clase base abstracta genérica:
  * Campo de versión opaco (`version`) accesible mediante accesores de lectura para concurrencia optimista (`TRX-02`).
  * Colección protegida de `DomainEvent` acumulados durante la mutación.
  * Métodos de ciclo de vida de eventos: `registerEvent()`, `hasDomainEvents()` y vaciado atómico inmutable `pullDomainEvents()`.
* **`BaseEntity<ID>`:** Clase base genérica con igualdad y cálculo de hash basados estrictamente en la identidad persistente inmutable (`id`).
* **`ValueObject`:** Interfaz marcadora para conceptos de dominio inmutables basados en sus atributos.
* **`DomainEvent`:** Contrato inmutable para eventos de dominio:
  * `eventId`: UUIDv7 único del evento.
  * `aggregateId`: Identificador del agregado emisor en formato String.
  * `occurredAt`: Marca temporal UTC inmutable.
  * `eventType`: Identificador semántico versionado (ej. `<subdominio>.<entidad>.created.v1`).
* **Jerarquía de Excepciones de Coste Cero:**
  * `BaseException`: Excepción abstracta no comprobada con flag `capturesDiagnostics()` para desactivar la captura de trazas de pila en errores funcionales.
  * `ErrorCode`: Interfaz funcional que obliga a exponer un código alfanumérico estable (`code() -> String`).
  * `ErrorCategory`: Enum semántico (`VALIDATION`, `UNAUTHENTICATED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `INTERNAL`).
  * `FieldViolation`: Estructura inmutable `(field, message)` para detallar fallos específicos por atributo.
  * Subclases tipadas estándar: `ResourceNotFoundException`, `ConflictException`, `ValidationException`, `ForbiddenException`, `UnauthenticatedException`, `InfrastructureException`, `ExternalServiceException`.

### 4.2 Aplicación Base (`shared.application`)

* **`ExecutionContext`:** Record o estructura inmutable `(userType, tenantId, userId, roles, correlationId)`:
  * Instancia para operaciones públicas o no autenticadas (`anonymous()`).
  * Predicados de consulta: `isAuthenticated()`, `hasTenant()`, `hasRole(role)`, `hasAnyRole(roles...)`.
  * Aserciones inmediatas *fail-fast*: `requireUserId()` y `requireTenantId()`.
* **`UserType`:** Clasificación del actor (`ANONYMOUS`, `CLIENT`, `TENANT_USER`, `SAAS_ADMIN`).
* **Paginación Transversal:**
  * **`PageResult<T>`:** Envoltura inmutable para paginación por desplazamiento (offset): `items`, `page`, `size`, `totalElements`, `totalPages`, factoría `of(...)` con cálculo exacto de páginas y función de mapeo `map(Function)`.
  * **`CursorResult<T>`:** Envoltura inmutable para paginación por cursor/keyset: `items`, `nextCursor`, factoría `of(...)` basada en la estrategia de búsqueda `limit + 1`.
* **Puertos Secundarios Transversales (`shared.application.port.out`):**
  * `EventPublisherPort`: Publicación local y remota de eventos de dominio.
  * `ExecutionContextPort`: Recuperación del contexto de ejecución activo en el hilo de trabajo.
  * `UtcClockPort`: Abstracción de consulta temporal en UTC (`now()`, `todayUtc()`).
  * `UuidGeneratorPort`: Abstracción para la generación determinista de UUIDv7 (`generateId()`).

### 4.3 Infraestructura Base (`shared.infrastructure`)

* **Adaptadores Web Perimetrales:**
  * `ApiHeaders`: Constantes canónicas de cabeceras HTTP (`X-Tenant-Id`, `X-User-Id`, `X-Roles`, `X-Correlation-Id`).
  * `ExecutionContextFilter`: Filtro HTTP prioritario que extrae cabeceras, vincula el contexto al hilo de ejecución, puebla el contexto de diagnóstico de logging (MDC) y garantiza su purga total en el bloque `finally`.
  * `ErrorResponse`: Estructura serializable inmutable `(status, code, detail, errors)` configurada para omitir el campo `errors` cuando esté vacío.
  * `GlobalExceptionHandler`: Controlador centralizado de excepciones que intercepta `BaseException`, fallos de validación sintáctica y excepciones imprevistas, enmascarando los fallos técnicos con `"An unexpected error occurred"`.
* **Adaptadores de Salida Transversales:**
  * `UtcClockAdapter`: Implementación basada en `Clock.systemUTC()` con soporte de inyección para pruebas.
  * `ExecutionContextHolder`: Almacén estático respaldado por `ThreadLocal` con métodos zero-allocation.
  * `UuidGeneratorAdapter`: Implementación lock-free CAS de UUIDv7 conforme a RFC 9562 combinando 48 bits de timestamp Unix epoch con contador monotónico de 12 bits.

---

## 5. Maquinaria Asíncrona: Transactional Outbox y Recuperación

La semilla proporciona el ciclo de vida completo de entrega asíncrona fiable para dar cumplimiento a `TRX-03`, `TRX-06` y `SED-05`:

### 5.1 Esquema DDL Base (`outbox_events`)

Migración baseline empaquetada de serie en el motor de persistencia relacional:

```sql
CREATE TABLE outbox_events (
    event_id UUID PRIMARY KEY,
    tenant_id UUID,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(128) NOT NULL,
    payload JSONB NOT NULL,
    correlation_id VARCHAR(120),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    retry_count INT NOT NULL DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    locked_at TIMESTAMPTZ,
    next_retry_at TIMESTAMPTZ
);

-- Índice optimizado para sondeo no contencioso O(1)
CREATE INDEX idx_outbox_processing
ON outbox_events (status, created_at ASC)
WHERE status IN ('PENDING', 'PROCESSING');

-- Índice para reintentos programados
CREATE INDEX idx_outbox_retry
ON outbox_events (status, next_retry_at ASC)
WHERE status = 'FAILED' AND next_retry_at IS NOT NULL;
```

### 5.2 Circuito de Relay y Despacho

1. **Reserva Atómica de Lotes (`OutboxRelay`):**
   * Tarea programada periódica configurable (`outbox.poll-interval`).
   * Consulta con bloqueo pesimista no contencioso que permite a múltiples instancias de la aplicación competir de forma segura:
     ```sql
     SELECT * FROM outbox_events
     WHERE status = 'PENDING'
     ORDER BY created_at ASC
     LIMIT :batchSize
     FOR UPDATE SKIP LOCKED;
     ```
   * Transición inmediata del lote a estado `PROCESSING` sellando `locked_at = clock_timestamp()`.
2. **Publicación y Gestión de Fallos:**
   * Entrega mediante el bus local de eventos o publicador de mensajería.
   * Entrega confirmada ➔ Transición a `DELIVERED`.
   * Error en entrega ➔ Transición a `FAILED`, incremento de `retry_count` y programación de `next_retry_at` con backoff exponencial.
   * Si `retry_count >= 5` ➔ Transición definitiva a `DEAD_LETTER` registrando la traza en `last_error`.
3. **Purga Automática de Entregados (`SED-05`):**
   * Tarea programada (`outbox.purge-cron`) que ejecuta el borrado físico de tuplas con estado `DELIVERED` cuya antigüedad supere la retención legal (por defecto: 7 días).

---

## 6. Módulo Canónico de Referencia (El Slice Demostrador Vivo)

El repositorio semilla incluye un Bounded Context funcional mínimo (`<subdominio-referencia>`). Su propósito no es aportar lógica corporativa, sino actuar como **demostrador vivo de código real compilado y testeado**:

```text
<raíz>/<subdominio-referencia>/
├── application/
│   ├── command/CreateSampleCommand.java            # Record inmutable de mutación
│   ├── query/FindSampleQuery.java                  # Record plano de búsqueda
│   ├── port/in/CreateSampleUseCase.java            # Interfaz UseCase única (APP-01)
│   ├── port/out/SampleRepositoryPort.java          # Puerto secundario de persistencia
│   ├── service/CreateSampleService.java            # Servicio orquestador @Transactional
│   └── result/SampleResult.java                    # DTO de salida
├── domain/
│   ├── model/SampleAggregate.java                  # Agregado con factorías y sin setters (DOM-02/04)
│   ├── event/SampleCreatedEvent.java               # DomainEvent inmutable
│   └── SampleError.java                            # Catálogo tipado ErrorCode (SHR-04)
└── infrastructure/adapter/
    ├── in/web/SampleController.java                # Controlador REST delgado (INP-01)
    ├── in/worker/SampleEventWorker.java            # Consumidor con Idempotency Gate (TRX-05)
    └── out/persistence/SamplePersistenceAdapter.java # Adaptador real desacoplado de BD (OUT-01)
```

---

## 7. Toolchain, Bootstrap, Migraciones y Operabilidad (*Production-Ready*)

### 7.1 Protocolo de Bootstrap y Parametrización

La semilla implementa un script de inicialización (`init-project.sh` o tarea de build) que automatiza la configuración de nuevos proyectos en el "Día 1":

1. **Renombrado de Namespace:** Sustitución segura del paquete raíz `<namespace.base>` y del nombre del artefacto (`rootProject.name`) en toda la estructura de archivos.
2. **Aislamiento de Perfiles de Configuración:**
   * `application.yml`: Configuración base compartida.
   * `application-local.yml`: Configuración local conectada a Docker Compose.
   * `application-test.yml`: Configuración determinista para CI con contenedores efímeros.

### 7.2 Línea Base de Migraciones de Base de Datos

* La infraestructura de base de datos se versiona formalmente mediante **Flyway** bajo `src/main/resources/db/migration/`.
* La migración `V1__init_shared_infrastructure.sql` crea de forma obligatoria:
  * Extensiones requeridas (`btree_gist`, `uuid-ossp`).
  * Funciones de mantenimiento (`fn_set_updated_at`, `fn_enforce_optimistic_locking`).
  * Tabla central de `outbox_events` con sus índices parciales.

### 7.3 Empaquetado y Contenedorización Multi-Stage

El archivo `Dockerfile` de la semilla sigue el estándar de seguridad industrial:

* **Etapa 1 (Build):** Compilación y empaquetado del binario desacoplado del entorno final.
* **Etapa 2 (Runtime):** Imagen base mínima (*distroless* o Alpine/Debian Slim), sin compiladores ni herramientas de build instaladas.
* **Principio de Mínimo Privilegio:** Ejecución forzada bajo usuario del sistema sin privilegios (`USER nonroot:nonroot` o UID 10001).

### 7.4 Observabilidad y Graceful Shutdown

* **Sondas de Salud (Healthchecks):** Exposición de endpoints perimetrales `/health/live` (liveness) y `/health/ready` (readiness verificando conectividad a BD).
* **Parada Limpia (*Graceful Shutdown*):** Configuración de un periodo de gracia (por defecto: 30 segundos) ante señales `SIGTERM`, permitiendo que el servidor HTTP rechace nuevas peticiones, el `OutboxRelay` finalice el lote en curso y el pool de conexiones se cierre de forma ordenada.

---

## 8. Criterios de Aceptación (DoD) y Matriz de Anti-Patrones

### 8.1 Definition of Done (DoD) de la Semilla

El repositorio base se declara completo, estable y apto para ser clonado cuando supera con éxito estos **6 escenarios ejecutables obligatorios en el pipeline de CI**:

1. **Compilación Limpia y Guardianes Verdes:**
   * El comando de build compila sin advertencias y pasa el 100% de los tests unitarios, de integración, la verificación de fronteras modulares y las reglas de arquitectura.
2. **Camino de Escritura y Outbox Atómico:**
   * Una petición HTTP al endpoint del módulo canónico devuelve `201 Created`.
   * La entidad y su correspondiente evento en `outbox_events` con estado `PENDING` quedan persistidos en la misma transacción ACID de base de datos.
3. **Despacho Asíncrono Exitoso:**
   * El relay en background toma el evento pendiente mediante `SKIP LOCKED`, lo publica y transiciona la fila a estado `DELIVERED`.
4. **Resiliencia ante Fallos e Idempotencia:**
   * Ante un consumidor fallido, el evento agota los reintentos con backoff y se traslada a `DEAD_LETTER`.
   * El reenvío intencionado del mismo evento al worker es detectado por la compuerta de idempotencia y descartado sin duplicar mutaciones.
5. **Captura Estructurada de Excepciones sin Traza:**
   * Una petición inválida aborta la transacción y devuelve el JSON estandarizado `ErrorResponse`, verificando en los logs que no se volcó traza de pila de la JVM para errores funcionales.
6. **Parada Limpia (*Graceful Shutdown*):**
   * El envío de una señal `SIGTERM` durante el procesamiento de un lote permite finalizar la tupla activa y libera las conexiones de base de datos sin errores ni registros huérfanos.

### 8.2 Matriz Pedagógica de Anti-Patrones de la Semilla

| ❌ Anti-Patrón Común | 💥 Por qué falla | ✅ Solución Normativa de la Semilla | Regla Asociada |
| --- | --- | --- | --- |
| **Kernel "Cajón de Sastre" (*Junk Drawer*)** | Acumular DTOs, utilidades o modelos de negocio en `shared/` destruye el aislamiento modular. | Limitar `shared/` exclusivamente a tipos base abstractos, contexto y contratos universales. | `SED-01` |
| **Generar IDs en Motor (Serial / UUIDv4)** | Causa contención y fragmentación de índices B-Tree en bases de datos con alto volumen de inserción. | Generar identificadores UUIDv7 secuenciales en la capa de aplicación con generador lock-free. | `SED-02` |
| **Semilla Vacía sin Módulo Canónico** | Impide verificar la arquitectura con código real y obliga a cada programador a inventar la primera implementación. | Incluir un módulo canónico de referencia que implemente los 3 flujos operativos completos. | `SED-04` |
| **Doble Escritura sin Outbox en la Semilla** | Delegar la publicación de eventos a llamadas de red directas genera inconsistencia de datos irrecuperable. | Integrar de serie la tabla `outbox_events` y el proceso de relay con `SKIP LOCKED`. | `SED-05` |
| **Esquemas de Base de Datos sin Migraciones** | Crear tablas manualmente o mediante auto-generación de ORMs (`ddl-auto=update`) provoca desastres en producción. | Incluir de serie la migración Flyway baseline versionada en el código fuente. | `SED-06` |
| **Contenedores Docker con Usuario Root** | Ejecutar aplicaciones en producción con permisos de administrador viola estándares básicos de seguridad. | Empaquetado multi-stage con ejecución forzada bajo usuario no privilegiado (`nonroot`). | `SED-07` |

### 8.3 Checklist de Pull Request para Mantenimiento de la Semilla

Antes de aprobar modificaciones sobre el repositorio semilla, el revisor debe certificar:

* [ ] ¿El paquete `shared/` permanece completamente agnóstico y libre de reglas de negocio específicas? (`SED-01`)
* [ ] ¿Todos los generadores y entidades base preservan la generación de identificadores UUIDv7? (`SED-02`)
* [ ] ¿El contexto de ejecución (`ExecutionContext`) mantiene inmutabilidad y limpieza en hilos virtuales? (`SED-03`)
* [ ] ¿El módulo canónico de referencia compila y sirve de base ejecutable para los tests de arquitectura? (`SED-04`)
* [ ] ¿Las consultas del Outbox Relay preservan el bloqueo no contencioso `SKIP LOCKED` y la purga periódica? (`SED-05`)
* [ ] ¿Toda nueva tabla o función base cuenta con su script de migración versionado en la línea base? (`SED-06`)
* [ ] ¿El Dockerfile compila en modo multi-stage y ejecuta bajo usuario no privilegiado? (`SED-07`)
* [ ] ¿El build de CI pasa el 100% de los tests y los 6 criterios de la Definition of Done en verde? (`SED-08`)
