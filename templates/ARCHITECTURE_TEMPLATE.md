# Manual de Arquitectura — [<PATRÓN_ARQUITECTÓNICO_Y_FRAMEWORK>]

> **Naturaleza del documento.** Especificación técnica normativa y Fuente Única de Verdad (SSOT) para desarrolladores, revisiones de código en Pull Requests, herramientas de análisis estático/linters de arquitectura y generadores automáticos o agentes de IA.
>
> **Fuerza normativa.** Las reglas etiquetadas como `MUST` son obligaciones técnicas estrictas cuyo incumplimiento bloquea el pipeline de integración continua o la aprobación de Pull Requests. Las reglas `NEVER` son prohibiciones taxativas que solo admiten excepciones mediante un registro de decisión formal (`ADR`) con condición de salida escrita.

---

## 1. Metadatos de Gobernanza y Fuerza Normativa

* **Preset / Ecosistema:** `[<NOMBRE_DEL_PRESET> — Ej: Java Spring Modulith / Python FastAPI / Node.js Clean Architecture]`
* **Versión del Estándar:** `v1.0.0`
* **Estado:** `[Propuesta / Normativo / En Deprecación]`
* **Repositorio Semilla Asociado:** `[<URL_O_NOMBRE_DE_LA_SEMILLA_EJECUTABLE>]`
* **Alcance:** Aplicable a todos los servicios y aplicaciones desarrolladas bajo este stack tecnológico dentro del marco de trabajo FProg.

---

## 2. Notación y Taxonomía de Reglas

### 2.1 Identificadores de Regla

La notación sigue la estructura unívoca: **`ÁMB-nn · FUERZA [TIPO]`**.

* **`MUST`**: Obligación técnica estricta.
* **`NEVER`**: Prohibición técnica absoluta.
* **`[A]`**: Verificación automatizada obligatoria en el pipeline de CI (linters, pruebas de análisis estático estructural o compilador).
* **`[R]`**: Verificación manual obligatoria en Pull Request mediante checklist técnico.

### 2.2 Catálogo Cerrado de Ámbitos (Acrónimos de 3 Letras)

* **`ARC`**: Principios macro, fronteras modulares, visibilidad pública y verificación estructural.
* **`DOM`**: Modelo de Dominio puro, entidades, invariantes, enums de estado y Value Objects.
* **`APP`**: Casos de uso (`UseCases`), orquestación de aplicación, separación CQRS y contratos.
* **`INP`**: Adaptadores de entrada / *driving* (Controladores REST, WebSockets, Workers, Schedulers).
* **`OUT`**: Adaptadores de salida / *driven* (Persistencia de base de datos, Clientes HTTP, Brokers).
* **`TRX`**: Delimitación transaccional, concurrencia optimista y Transactional Outbox.
* **`SHR`**: Kernel compartido técnico, contexto inmutable (`ExecutionContext`) y catálogo de errores.

---

## 3. Topología de Directorios y Fronteras Modulares

### 3.1 Estructura Canónica del Proyecto

El sistema se organiza en módulos de negocio independientes gobernados por fronteras estrictas. La infraestructura transversal técnica reside en el módulo abierto `shared/`.

```text
<raíz-del-proyecto>/
│
├── <shared>/                               # Kernel compartido transversal (abierto a todos los módulos)
│   ├── domain/                             # Tipos base genéricos (AggregateRoot, BaseEntity, DomainEvent)
│   ├── application/                        # Contexto transversal (ExecutionContext), paginación (PageResult/CursorResult)
│   │   └── port/out/                       # Puertos secundarios universales (Clock, UuidGenerator, EventPublisher)
│   └── infrastructure/                     # Adaptadores transversales (Filtros perimetrales, MDC, Handler de error)
│
└── <subdominio>/                           # Módulo / Bounded Context de Negocio
    │
    ├── application/                        # API PÚBLICA DEL MÓDULO (Contrato expuesto al resto del sistema)
    │   ├── command/                        # Comandos inmutables de mutación (records / estructuras planas)
    │   ├── query/                          # Modelos de consulta inmutables (parámetros y filtros)
    │   ├── port/
    │   │   ├── in/                         # Casos de uso / interfaces primarias (*UseCase)
    │   │   └── out/                        # Puertos secundarios (gateways externos y repositorios)
    │   ├── service/                        # Implementaciones orquestadoras transaccionales (*Service)
    │   └── result/                         # DTOs inmutables de salida de la aplicación (*Result)
    │
    ├── domain/                             # DETALLE PRIVADO INTERNO (Agnóstico a frameworks y transporte)
    │   ├── model/                          # Entidades raíz, entidades internas, Value Objects
    │   │   └── enums/                      # Enums de estado de negocio
    │   ├── event/                          # Eventos de dominio generados ante mutaciones
    │   └── <Subdominio>Error               # Catálogo tipado de errores de negocio (implements ErrorCode)
    │
    └── infrastructure/                     # DETALLE PRIVADO INTERNO (Tecnología, transporte y persistencia)
        └── adapter/
            ├── in/                         # Adaptadores Primarios (Driving)
            │   ├── web/                    # Controladores de red, Request DTOs, Mappers de transporte
            │   └── worker/                 # Consumidores de mensajes, tareas programadas (Schedulers)
            └── out/                        # Adaptadores Secundarios (Driven)
                ├── persistence/            # Implementaciones de almacenamiento (ORM, SQL nativo)
                ├── messaging/              # Publicadores a colas, brokers o relé outbox
                └── client/                 # Adaptadores de consumo de APIs externas
```

### 3.2 Contrato de Visibilidad Inter-Módulo

1. **API Pública del Módulo (`application`):** Único punto de entrada a través del cual otros módulos pueden interactuar con este Bounded Context. Expone exclusivamente interfaces de casos de uso (`port/in`), Comandos, Queries y DTOs de salida (`*Result`).
2. **Confinamiento Privado (`domain` e `infrastructure`):** Queda estrictamente prohibido que un módulo acceda directamente al modelo de dominio o a las clases de persistencia/adaptadores de otro módulo. La colaboración intermodular se efectúa mediante contratos en `application` o suscripciones asíncronas a eventos de dominio.

---

## 4. Matriz de Dependencias y Regla de Purificación de Contexto

### 4.1 Regla Direccional y Visibilidad entre Capas

Las dependencias fluyen estrictamente desde el exterior hacia el centro: **Adaptadores ➔ Aplicación ➔ Dominio**.

| Capa / Componente | Puede depender de | Prohibido depender de |
| --- | --- | --- |
| **Dominio** (`domain`) | Tipos base de `shared/domain`, tipos nativos del lenguaje. | Frameworks de transporte (HTTP/Web), ORMs/motores de base de datos, serializadores externos (Jackson/JSON), Aplicación, Adaptadores. |
| **Aplicación** (`application`) | `domain` local, `shared/domain`, `shared/application`, contratos de `application` expuestos por otros módulos. | Adaptadores locales (`infrastructure/adapter/*`), Bases de datos, Tecnologías de transporte (HTTP/REST), paquetes internos de otros módulos. |
| **Adaptadores Entrada** (`adapter.in`) | `application` local, `shared`, enums de estado inmutables de `domain`. | Entidades persistentes completas de base de datos, adaptadores de salida, cualquier paquete interno de otro módulo. |
| **Adaptadores Salida** (`adapter.out`) | Puertos de `application/port/out` locales, tipos de dominio para mapeo explícito, `shared`. | Adaptadores de entrada, controladores, cualquier paquete de otro módulo. |
| **Kernel Compartido** (`shared`) | Tipos nativos del lenguaje, utilidades transversales técnicas. | Cualquier módulo o subdominio de negocio específico. |

### 4.2 Regla de Purificación de Contexto y Seguridad

El contexto de seguridad y transporte (`ExecutionContext`: token, cabeceras, identidad y roles) **jamás debe penetrar en el modelo de dominio**:

* **Adaptadores de entrada (`adapter.in`):** Interceptan y extraen las cabeceras perimetrales o tokens de sesión, reconstruyendo el objeto inmutable de contexto.
* **Capa de Aplicación (`application`):** Valida la autorización o permisos invocando métodos de control sobre el contexto. Extrae los identificadores planos requeridos (`tenantId`, `userId`).
* **Capa de Dominio (`domain`):** Recibe exclusivamente tipos primitivos, identificadores planos o Value Objects. El dominio desconoce por completo la existencia de tokens de transporte, cookies, sesiones o frameworks de seguridad.

---

## 5. Los 3 Flujos Canónicos de Operación

### 5.1 Camino de Escritura Síncrono (Command Flow — Mutación de Estado)

Obliga a hidratar el Agregado de Dominio para validar invariantes y ejecutar lógica de negocio protegida:

```text
[Cliente / Consumidor]
      │
      ▼
1. [Adapter IN (Controller)]  ──► Valida sintaxis de la petición.
      │                           Mapea payload a Command inmutable (record/struct plano).
      ▼
2. [Application (Service)]    ──► Abre delimitación transaccional (@Transactional).
      │                           Hidrata el Agregado invocando puerto de salida (port/out).
      ▼
3. [Domain (AggregateRoot)]   ──► Ejecuta método de negocio semántico.
      │                           Valida invariantes legales y encola DomainEvent inmutable.
      ▼
4. [Adapter OUT (Repository)] ──► Persiste el nuevo estado del Agregado en BD.
      │                           Persiste el DomainEvent en la tabla 'outbox_events' (misma transacción ACID).
      ▼
5. [Application (Service)]    ──► Cierra transacción. Retorna DTO de salida (*Result)
      │                           o lanza excepción tipada de coste cero (BaseException).
      ▼
6. [Adapter IN (Controller)]  ──► Devuelve HTTP Status de éxito (200 OK, 201 Created).
                                  Si hay fallo, GlobalExceptionHandler mapea a ErrorResponse estándar.
```

### 5.2 Camino de Lectura Síncrono (Query Flow — CQRS Ligero)

Optimiza la memoria y el rendimiento proyectando directamente desde la base de datos a DTOs de salida. **Queda estrictamente prohibido hidratar Agregados de dominio para operaciones de consulta**:

```text
[Cliente / Consumidor]
      │
      ▼
1. [Adapter IN (Controller)]  ──► Mapea parámetros de búsqueda y paginación a Query plano.
      │
      ▼
2. [Application (Service)]    ──► Ejecuta caso de uso bajo transacción de solo lectura (read-only).
      │                           Invoca puerto de consulta optimizada en application/port/out.
      ▼
3. [Adapter OUT (Proyección)] ──► Ejecuta consulta SQL/NoSQL proyectada directamente a DTOs (*Result).
      │                           (Sin instanciar entidades pesadas de ORM ni Agregados de dominio).
      ▼
4. [Adapter IN (Controller)]  ──► Devuelve HTTP 200 con el DTO o PageResult paginado.
```

### 5.3 Camino de Consumo Asíncrono / Background (Worker & Event Flow)

Gobierna la recepción desacoplada de eventos, mensajes de broker o tareas programadas (*Schedulers*), garantizando entrega fiable y procesamiento seguro:

```text
[Message Broker / Schedulers / Outbox Relay]
      │
      ▼
1. [Adapter IN (Worker)]      ──► Recibe payload del evento/mensaje.
      │                           Inicializa ExecutionContext de sistema o propaga correlationId.
      ▼
2. [Idempotency Gate]         ──► VERIFICACIÓN DE IDEMPOTENCIA: Comprueba si eventId ya fue procesado.
      │                           - Si YA fue procesado ──► Cortocircuita, emite ACK y termina.
      │                           - Si NO fue procesado ──► Continúa ejecución.
      ▼
3. [Application (Service)]    ──► Mapea a Command inmutable e invoca el UseCase correspondiente.
      │                           Abre delimitación transaccional local.
      ▼
4. [Domain / Persistencia]    ──► Procesa lógica de negocio y registra mutaciones en base de datos.
      │
      ▼
5. [Adapter IN (Worker)]      ──► Registra eventId en almacén de procesados y emite confirmación (ACK).
                                  Si ocurre un fallo técnico: aplica backoff exponencial con reintentos;
                                  agotado el tope máximo, deriva el mensaje a Dead-Letter Queue (DLQ).
```

---

## 6. Catálogo Normativo de Reglas por Ámbito

### 6.1 Arquitectura y Fronteras Modulares (`ARC`)

* **`ARC-01 · MUST` [A]** Las fronteras entre subdominios se verificarán de forma automática en el pipeline de CI mediante suites de análisis estático o pruebas de arquitectura de compilación. El paquete `application` de cada subdominio debe ser el único expuesto al resto de módulos.
* **`ARC-02 · NEVER` [A]** Ningún módulo accederá a clases de los paquetes `domain` o `infrastructure` de otro módulo. La colaboración intermodular se efectúa exclusivamente mediante eventos asíncronos o contratos públicos de `application`.
* **`ARC-03 · MUST` [A]** La especificación modular y los diagramas estructurales deben generarse o validarse en compilación mediante herramientas del ecosistema.
* **`ARC-04 · MUST` [R]** Toda desviación consciente de este manual debe registrarse como un `ADR` formal en `/adr` con contexto, alternativa descartada y condición de salida.
* **`ARC-05 · NEVER` [R]** Se permitirá la adopción de capacidades a medias (*regla de adopción diferida*): queda prohibido incorporar tablas, eventos o infraestructura sin el circuito completo que los consuma y verifique.

### 6.2 Dominio (`DOM`)

* **`DOM-01 · NEVER` [A]** El dominio importará o dependerá de frameworks de transporte, anotaciones de persistencia/ORM, serializadores externos o librerías de terceros.
* **`DOM-02 · MUST` [A]** Las entidades y agregados tendrán constructores protegidos o privados, instanciándose únicamente mediante métodos factoría explícitos (`create` para creación inicial, `reconstruct` para hidratación desde persistencia).
* **`DOM-03 · MUST` [R]** Los conceptos de dominio con reglas de invariante o formatos complejos deben encapsularse en **Value Objects** inmutables que autovaliden su integridad en la instanciación.
* **`DOM-04 · NEVER` [A]** Los agregados expondrán métodos mutadores genéricos (`setters`). La mutación de estado se realiza exclusivamente mediante métodos de negocio semánticos.
* **`DOM-05 · MUST` [A]** Cada mutación exitosa en un agregado que deba ser conocida fuera de su límite transaccional registrará un `DomainEvent` inmutable en su lista interna de eventos pendientes.
* **`DOM-06 · MUST` [A]** Los puertos de salida de repositorio declarados en `application/port/out` operarán exclusivamente con agregados y tipos de dominio en sus operaciones de escritura, nunca con entidades de persistencia.

### 6.3 Aplicación (`APP`)

* **`APP-01 · MUST` [A]** Cada caso de uso debe representarse como una interfaz en `application/port/in` con el sufijo `UseCase` y un único método de ejecución (`execute`), implementado en una clase de servicio en `application/service` con el sufijo `Service`.
* **`APP-02 · NEVER` [A]** La capa de aplicación contendrá lógica de cálculo de negocio o validación de invariantes; su función se limita estrictamente a la orquestación técnica del flujo.
* **`APP-03 · MUST` [R]** La entrada a un caso de uso debe ser un Comando o Query inmutable:
  * **Comandos exclusivamente Web:** Si se consumen únicamente por HTTP, se definen como estructuras planas sin validación interna redundante, delegando en la validación sintáctica del adaptador web.
  * **Comandos Multicanal:** Si pueden ser invocados desde colas, brokers o schedulers, deben implementar comprobaciones defensivas inmediatas (*fail-fast*) de no-nulidad en su constructor o método inicial.
  * **Queries:** Se definen siempre como estructuras planas de parámetros sin lógica interna.
* **`APP-04 · NEVER` [A]** Un caso de uso devolverá agregados de dominio o entidades de base de datos hacia los adaptadores primarios o hacia otros módulos. La salida es siempre un DTO plano (`*Result`) o el lanzamiento de una excepción tipada.

### 6.4 Adaptadores de Entrada (`INP`)

* **`INP-01 · MUST` [A]** Los controladores delegan inmediatamente a la capa de aplicación tras mapear las peticiones a Comandos o Queries. Se autoriza el uso de enums puros de estado de dominio en controladores y mappers web del propio módulo para evitar duplicaciones.
* **`INP-02 · NEVER` [A]** Ningún controlador ni adaptador primario contendrá lógica de negocio ni gestionará transacciones de base de datos directamente.
* **`INP-03 · MUST` [A]** Todas las excepciones de la API deben traducirse de forma centralizada al contrato estandarizado `ErrorResponse` (`status`, `code`, `detail`, `errors`), omitiendo detalles técnicos internos en producción.
* **`INP-04 · MUST` [R]** Todo proceso en background (Workers, Schedulers) debe inicializar o propagar su `ExecutionContext` antes de invocar cualquier caso de uso.

### 6.5 Adaptadores de Salida y Persistencia (`OUT`)

* **`OUT-01 · NEVER` [A]** Las entidades anotadas o vinculadas al ORM/persistencia saldrán del adaptador de persistencia hacia la aplicación o el dominio.
* **`OUT-02 · MUST` [A]** El mapeo entre las entidades de base de datos y los agregados de dominio se realizará de forma explícita mediante mappers dedicados, prohibiéndose la reflexión implícita.
* **`OUT-03 · NEVER` [R]** Se permitirá la carga perezosa (*lazy loading*) fuera del adaptador de persistencia; los agregados se cargan completos y consistentes.
* **`OUT-04 · MUST` [R]** Las consultas de lectura y listados **no deben hidratar Agregados de dominio**; deben mapear desde base de datos directamente a DTOs de proyección optimizados (`*Result`).
* **`OUT-05 · MUST` [A]** Si se requieren consultas avanzadas, tipos SQL nativos o extensiones no gestionadas limpiamente por el ORM, se autoriza el uso de adaptadores directos de persistencia nativa confinados en su propio submódulo de salida.

### 6.6 Transacciones, Consistencia y Outbox (`TRX`)

* **`TRX-01 · MUST` [A]** La demarcación transaccional residirá exclusivamente en la capa de aplicación: transacciones de lectura/escritura en comandos y de solo lectura (`read-only`) en queries.
* **`TRX-02 · MUST` [A]** Los agregados de dominio deben implementar control de concurrencia optimista mediante un atributo o campo de versión opaco (`version`).
* **`TRX-03 · MUST` [A]** Todo evento de dominio generado por un agregado que deba ser emitido al exterior se persistirá en la misma transacción local de base de datos en la tabla `outbox_events` (**Transactional Outbox Pattern**).
* **`TRX-04 · NEVER` [A]** Se realizarán publicaciones síncronas por red hacia brokers de mensajería dentro de la transacción de negocio; la publicación la ejecuta un worker desacoplado que lee del outbox.
* **`TRX-05 · MUST` [R]** Todo consumidor de eventos debe implementar control de **idempotencia**, verificando si el identificador del evento (`eventId`) ya ha sido procesado antes de mutar estado.
* **`TRX-06 · MUST` [A]** Las tablas de outbox deben disponer de un proceso programado de purga para eliminar registros entregados con éxito que superen el periodo de retención legal configurado.

### 6.7 Kernel Compartido y Preocupaciones Transversales (`SHR`)

* **`SHR-01 · NEVER` [A]** El módulo compartido `shared` contendrá reglas de negocio específicas de ningún subdominio.
* **`SHR-02 · MUST` [A]** Todos los identificadores únicos globales del sistema se generarán en la capa de aplicación con formato **UUIDv7** secuencial en el tiempo conforme a RFC 9562.
* **`SHR-03 · MUST` [A]** Toda operación debe propagar el contexto inmutable de ejecución (`ExecutionContext`: tenant, actor, roles y correlationId) con métodos de aserción inmediata.
* **`SHR-04 · MUST` [R]** Los errores de negocio deben asociarse a un código alfanumérico inmutable y tipado mediante `ErrorCode` (`code()`), separando el código técnico del mensaje descriptivo.

---

## 7. Matriz de Automatización (Guardianes de CI)

La integridad arquitectónica de este manual se verifica obligatoriamente en el pipeline de CI mediante suites automatizadas:

| Regla | Mecanismo de Validación | Condición Técnica de Fallo Automatizada |
| --- | --- | --- |
| **`ARC-01`** | `[Linter Estructural / ArchUnit]` | Dependencias circulares entre módulos, falta de interfaces públicas o módulos mal ubicados. |
| **`ARC-02`** | `[Linter Estructural / ArchUnit]` | Clases de un módulo importan paquetes privados (`domain..` o `infrastructure..`) de otro módulo. |
| **`DOM-01`** | `[Linter de Capas / ArchUnit]` | Clases en `..domain..` importan librerías de frameworks web, persistencia/ORM o Jackson. |
| **`DOM-04`** | `[Linter de Estilo / ArchUnit]` | Métodos públicos en clases de `..domain.model..` tienen nombres que comienzan por `set`. |
| **`APP-01`** | `[Linter de Nomenclatura / ArchUnit]` | Clases en `..application.service..` no implementan exactamente una interfaz `UseCase` o no terminan en `Service`. |
| **`INP-01`** | `[Linter de Capas / ArchUnit]` | Clases en adaptadores de entrada dependen de `..domain.model..` (excluyendo enums puros de estado). |
| **`OUT-01`** | `[Linter de Capas / ArchUnit]` | Clases con anotaciones de base de datos/ORM residen fuera de los adaptadores de salida de persistencia. |
| **`TRX-01`** | `[Linter Transaccional / ArchUnit]` | Demarcación transaccional presente en clases o métodos situados fuera de la capa de aplicación. |

---

## 8. Matriz Pedagógica de Anti-Patrones Comunes

| ❌ Anti-Patrón Común | 💥 Por qué falla | ✅ Solución Normativa | Regla Asociada |
| --- | --- | --- | --- |
| **Acoplamiento Directo Inter-Módulo** | Un módulo inyecta repositorios o accede a entidades de otro módulo, destruyendo la modularidad y bloqueando la escalabilidad. | Comunicar módulos exclusivamente mediante contratos en `application` expuestos formalmente o vía eventos asíncronos. | `ARC-01`, `ARC-02` |
| **Hidratar Agregados para Lecturas** | Degrada drásticamente la latencia y satura la memoria al instanciar grafos enteros de objetos cuando solo se querían mostrar datos en pantalla. | Proyectar consultas de lectura directamente desde la base de datos a DTOs de salida optimizados (CQRS Ligero). | `OUT-04` |
| **Doble Escritura (Dual-Write) sin Outbox** | Si la base de datos confirma el guardado pero la red falla al notificar al broker, el sistema entra en inconsistencia irrecuperable. | Transactional Outbox Pattern: guardar en base de datos local y publicar de forma desacoplada en segundo plano. | `TRX-03`, `TRX-04` |
| **Consumo Asíncrono sin Idempotencia** | La red y los brokers garantizan entrega al menos una vez (*at-least-once*); procesar duplicados corrompe saldos y estados. | Idempotency Gate en adaptadores worker: verificar `eventId` antes de ejecutar mutaciones de estado. | `TRX-05` |
| **Agregado Anémico con Setters** | El agregado pierde el control de sus invariantes y cualquier servicio externo corrompe el estado interno. | Mutaciones mediante métodos de negocio semánticos y validación estricta en constructores o Value Objects. | `DOM-03`, `DOM-04` |
| **Entidad de Persistencia usada como Negocio** | El ORM o motor invade el dominio, forzando constructores vacíos y acoplando la lógica de negocio a tablas SQL. | Separar Agregado de Dominio de la entidad de persistencia mediante un mapper explícito bidireccional. | `DOM-01`, `OUT-01` |
| **Fuga del Contexto de Seguridad al Dominio** | Pasar objetos de sesión/token al dominio contamina las invariantes puras con detalles perimetrales de transporte. | La capa de aplicación valida el contexto y pasa al dominio únicamente identificadores planos primitivos. | `SHR-03`, `DOM-01` |
| **Lógica de Negocio en el Caso de Uso** | El caso de uso se vuelve un procedimiento monolítico y el modelo de dominio queda huérfano de lógica. | Mover las reglas de cálculo, transiciones e invariantes al interior de los métodos del Agregado. | `APP-02` |

---

## 9. Checklist de Pull Request y Gobernanza de Decisiones (ADRs)

### 9.1 Checklist de Verificación en Pull Request

Antes de aprobar la integración de código a ramas principales, el revisor debe certificar:

* [ ] ¿Las fronteras del módulo se respetan sin invadir paquetes internos (`domain` o `infrastructure`) de otros módulos? (`ARC-02`)
* [ ] ¿El paquete `application` expone su contrato formalmente como API pública hacia el exterior? (`ARC-01`)
* [ ] ¿El contexto de ejecución (`ExecutionContext`) se purifica en la aplicación sin filtrarse hacia el dominio? (`SHR-03`, `DOM-01`)
* [ ] ¿El Agregado protege sus invariantes sin exponer métodos mutadores genéricos (`setters`)? (`DOM-04`)
* [ ] ¿El caso de uso orquesta el flujo sin absorber reglas de cálculo que corresponden al Agregado? (`APP-02`)
* [ ] ¿Cada implementación en `application/service` atiende exactamente un `*UseCase` y termina en `Service`? (`APP-01`)
* [ ] ¿Las entidades de base de datos/ORM están estrictamente confinadas al adaptador de persistencia? (`OUT-01`)
* [ ] ¿Las consultas de lectura y listados van directas a proyecciones DTO sin hidratar Agregados? (`OUT-04`)
* [ ] ¿Toda mutación de negocio encola su correspondiente `DomainEvent` para el outbox transaccional? (`DOM-05`, `TRX-03`)
* [ ] ¿Todo consumidor de eventos (Worker) cuenta con compuerta de idempotencia antes de procesar lógica? (`TRX-05`)
* [ ] ¿Los errores se traducen a través del manejador global al contrato estándar `ErrorResponse`? (`INP-03`)

### 9.2 Índice de Registro de Decisiones de Arquitectura (ADR Base)

Toda decisión estructural adoptada en este Preset debe estar formalizada en el directorio `/adr` del repositorio:

| Identificador | Título de la Decisión | Estado | Decisión y Justificación Técnica | Enlace al Documento |
| --- | --- | --- | --- | --- |
| **`ADR-01`** | Persistencia Híbrida y Desacoplamiento de Modelos | Aceptado | Separación total de Agregados y entidades persistentes; uso de mapeos explícitos. | [`adr/ADR-001.md`](adr/ADR-001.md) |
| **`ADR-02`** | Publicación Asíncrona mediante Transactional Outbox | Aceptado | Persistencia local ACID de eventos para garantizar consistencia eventual libre de caídas de red. | [`adr/ADR-002.md`](adr/ADR-002.md) |
| **`ADR-03`** | Estandarización de Errores y Excepciones de Coste Cero | Aceptado | Contrato lean `ErrorResponse` y desactivación de captura de trazas en fallos de negocio. | [`adr/ADR-003.md`](adr/ADR-003.md) |
| **`ADR-04`** | Gobernanza Modular y Verificación en Compilación | Aceptado | Bounded Contexts directos con API pública en `application` y guardianes CI. | [`adr/ADR-004.md`](adr/ADR-004.md) |
| **`ADR-05`** | Validación Híbrida en Comandos (Web vs. Multicanal) | Aceptado | Comandos web como estructuras planas; comandos multicanal con validación fail-fast. | [`adr/ADR-005.md`](adr/ADR-005.md) |