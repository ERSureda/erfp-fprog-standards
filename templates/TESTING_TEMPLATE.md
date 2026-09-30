# Manual de Estrategia de Pruebas y Verificación Automatizada — [<PATRÓN_O_LENGUAJE>]

> **Naturaleza del documento.** Especificación técnica normativa y Fuente Única de Verdad (SSOT) para el diseño, estructura, ejecución y automatización de pruebas en el Preset `[<NOMBRE_DEL_PRESET>]`.
>
> **Documento normativo asociado.** `ARCHITECTURE.md`. Implementa y desarrolla en detalle los mecanismos de verificación técnica de las reglas `[A]` y la fiabilidad de los flujos de operación.
>
> **Fuerza normativa.** Las reglas etiquetadas como `MUST` son obligaciones técnicas estrictas cuyo incumplimiento bloquea el pipeline de integración continua o la aprobación de Pull Requests. Las reglas `NEVER` son prohibiciones taxativas que solo admiten excepciones mediante un registro de decisión formal (`ADR`) con condición de salida escrita.

---

## 1. Metadatos de Gobernanza y Fuerza Normativa

* **Preset / Ecosistema:** `[<NOMBRE_DEL_PRESET> — Ej: Java Spring Modulith / Python FastAPI / Node.js Clean Architecture]`
* **Versión del Estándar:** `v1.0.0`
* **Estado:** `[Propuesta / Normativo / En Deprecación]`
* **Repositorio Semilla Asociado:** `[<URL_O_NOMBRE_DE_LA_SEMILLA_EJECUTABLE>]`
* **Alcance:** Aplicable a la totalidad de suites de pruebas unitarias, de integración, slices de transporte, pruebas de carga de eventos y guardianes de arquitectura en proyectos regidos por este Preset.

---

## 2. Filosofía, Pirámide y Presupuesto de Ejecución (*CI Budget*)

El sistema de pruebas garantiza confianza de despliegue a producción sin comprometer la velocidad de entrega (*Time-to-Market*):

### 2.1 Principios Rectores

1. **Determinismo Absoluto:** Todo test debe producir el mismo resultado independientemente del entorno de ejecución, orden de llamada o concurrencia de hilos. Queda prohibida la dependencia de redes externas no controladas.
2. **Verosimilitud en Persistencia:** Las pruebas de persistencia se ejecutan obligatoriamente contra el motor de base de datos real de producción mediante contenedores efímeros. Queda prohibido el uso de emuladores o bases en memoria (H2, SQLite) que falseen dialectos SQL, tipos nativos (JSONB) o políticas de seguridad.
3. **Cero Aserciones Triviales:** Queda prohibido escribir pruebas que verifiquen getters, setters o código autogenerado; cada test debe validar una invariante de dominio, una orquestación transaccional o un contrato de interfaz.

### 2.2 Proporción de la Pirámide de Pruebas

La distribución del esfuerzo de pruebas sigue una estructura piramidal estricta:

* **Base (70% — Unitarias Puras):** Pruebas de Dominio y Casos de Uso aislados. Ejecución instantánea en memoria sin arranque de frameworks ni contenedores (milisegundos).
* **Cuerpo (25% — Integración Confinada / Slices):** Pruebas de adaptadores de salida contra base de datos real y slices de controladores web.
* **Cúspide (5% — Flujo Crítico E2E / Humo):** Verificación restringida exclusivamente al Happy Path del flujo de conversión principal definido en la especificación de producto (`01-OPPORTUNITY.md`).

### 2.3 Presupuesto Máximo de Ejecución (*CI Time Budget*)

* **Tiempo total del pipeline de pruebas:** La suite completa de tests de un proyecto no debe superar los **3 a 5 minutos** en los ejecutores de CI.
* **Umbral unitario:** Cada prueba unitaria de dominio debe ejecutarse en menos de **10 milisegundos**.
* **Umbral de integración:** Los tests que interactúan con contenedores deben reutilizar instancias activas para evitar penalizaciones de arranque.

---

## 3. Catálogo Normativo de Reglas de Testing (`TST`)

### 3.1 Identificadores de Regla

La notación sigue la estructura unívoca: **`TST-nn · FUERZA [TIPO]`**.

* **`MUST`**: Obligación técnica estricta.
* **`NEVER`**: Prohibición técnica absoluta.
* **`[A]`**: Verificación automatizada obligatoria en el pipeline de CI.
* **`[R]`**: Verificación manual obligatoria en Pull Request mediante checklist.

### 3.2 Reglas del Sistema de Pruebas

* **`TST-01 · NEVER` [A]** Se utilizarán dobles de prueba (*mocks*, *stubs*, *spies*) en pruebas de la capa de Dominio; los agregados, entidades y Value Objects se prueban exclusivamente mediante instanciación real.
* **`TST-02 · MUST` [A]** En pruebas de Casos de Uso (`application/service`), los dobles de prueba están confinados estrictamente a las interfaces secundarias de salida (`port/out`), prohibiéndose el mocking de clases de dominio o estructuras internas de aplicación.
* **`TST-03 · MUST` [A]** Toda prueba de adaptadores de persistencia debe ejecutarse contra una instancia del motor de base de datos idéntico a producción mediante contenedores efímeros aislados.
* **`TST-04 · MUST` [A]** Las pruebas de adaptadores web deben ejecutarse como slices de transporte, validando exclusivamente mapeo sintáctico, autorizaciones de cabecera (`ExecutionContext`), status HTTP y contrato `ErrorResponse`, mockeando la interfaz del caso de uso (`port/in`).
* **`TST-05 · MUST` [A]** Todo consumidor asíncrono o worker debe contar con una prueba de idempotencia obligatoria que certifique la deduplicación ante la recepción consecutiva del mismo evento (`eventId`).
* **`TST-06 · NEVER` [A]** Se utilizarán pausas ciegas de tiempo (`Thread.sleep()`, `time.sleep()`) en pruebas asíncronas; toda sincronización debe efectuarse mediante mecanismos reactivos o sondeo activo condicional (*polling*) con tiempo límite (*timeout*).
* **`TST-07 · MUST` [A]** La suite de pruebas debe incorporar la ejecución obligatoria de los guardianes de arquitectura (análisis estático de dependencias) que verifiquen las reglas `ARC-*`, `DOM-*` y `OUT-*`, rompiendo el build ante infracciones.
* **`TST-08 · MUST` [R]** Toda prueba debe estructurarse obligatoriamente bajo el patrón **Arrange-Act-Assert (AAA)** (o *Given-When-Then*) y utilizar una convención de nomenclatura semántica que describa la intención y el escenario evaluado.

---

## 4. Topología y Delimitación de Pruebas por Capa

```text
┌────────────────────────────────────────────────────────────────────────┐
│ 4.1 DOMINIO: Unitarias Puras (Invariantes, Value Objects, Factorías)   │ ──► Cero mocks, 100% en memoria
├────────────────────────────────────────────────────────────────────────┤
│ 4.2 APLICACIÓN: Orquestación (UseCases con mocks en port/out)          │ ──► Mocks solo en dependencias I/O
├────────────────────────────────────────────────────────────────────────┤
│ 4.3 SALIDA / PERSISTENCIA: Integración Real (Contenedor Efímero)       │ ──► Motor SQL real, mapeo, RLS, Outbox
├────────────────────────────────────────────────────────────────────────┤
│ 4.4 ENTRADA / WEB: Slices de Transporte (MockMvc / HTTP Test Client)   │ ──► Validaciones, Headers, HTTP Status
├────────────────────────────────────────────────────────────────────────┤
│ 4.5 WORKERS / ASÍNCRONO: Idempotencia y Políticas de Fallo / DLQ       │ ──► Replay de eventos, deduplicación
└────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Pruebas de Dominio (Unitarias Puras)

* **Qué se prueba:** Invariantes de AggregateRoot, factorías estáticas (`create`, `reconstruct`), Value Objects inmutables, transiciones de estado legales y encolado de eventos de dominio (`DomainEvent`).
* **Qué está prohibido:** Levantar contextos de frameworks, abrir transacciones, simular métodos con mocks o interactuar con el sistema de archivos.

### 4.2 Pruebas de Casos de Uso (Aplicación)

* **Qué se prueba:** La orquestación del flujo de negocio: recuperación de la entidad vía repositorio, llamada al método semántico del agregado, persistencia del nuevo estado y retorno del DTO (`*Result`). Verificación de propagación de excepciones tipadas (`BaseException`).
* **Aislamiento:** Los puertos secundarios (`port/out`: repositorios, clientes externos, publicadores) se sustituyen mediante mocks o fakes controlados.

### 4.3 Pruebas de Adaptadores de Salida (Persistencia Real)

* **Qué se prueba:** Mapeo explícito bidireccional (Agregado ➔ Entidad de BD ➔ Agregado), restricciones de clave foránea (`FOREIGN KEY`), constraints de unicidad (`UNIQUE`), comprobaciones `CHECK`, control de concurrencia optimista (`version = version + 1`) y consultas nativas optimizadas.
* **Infraestructura:** Instancia real de base de datos administrada por contenedor.

### 4.4 Pruebas de Adaptadores de Entrada (Web Slices)

* **Qué se prueba:** Validación sintáctica de la petición (`@Valid`), extracción de cabeceras perimetrales (`ApiHeaders`) para reconstruir el `ExecutionContext`, códigos de estado HTTP correctos (200, 201, 204) y serialización estructurada de `ErrorResponse` ante fallos.
* **Aislamiento:** El caso de uso (`port/in`) se sustituye mediante un mock para evitar tocar capas inferiores.

### 4.5 Pruebas de Consumidores Asíncronos (Workers)

* **Qué se prueba:** Deserialización de payloads JSON desde la tabla de Outbox o broker, propagación del contexto de correlación, ejecución del caso de uso de destino y comportamiento ante fallos transitorios frente a permanentes.

---

## 5. Pruebas Transversales Críticas: Multi-Tenancy e Idempotencia

### 5.1 Test Mandatorio de Fuga entre Inquilinos (*Tenant Leakage Test*)

En arquitecturas multi-tenant basadas en Row-Level Security (RLS) o discriminante lógico, es obligatorio implementar una prueba negativa de aislamiento:

1. **Configuración:** Persistir un registro perteneciente al `Tenant A` en la base de datos.
2. **Ejecución:** Inicializar el contexto de ejecución bajo el `Tenant B` e intentar leer, actualizar o borrar dicho registro a través de los adaptadores correspondientes.
3. **Aserción:** La consulta debe retornar resultado vacío (0 registros) o lanzar `ResourceNotFoundException` / `ForbiddenException`. Queda estrictamente prohibido que los datos del `Tenant A` sean visibles para el `Tenant B`.

### 5.2 Test Mandatorio de Atomicidad del Transactional Outbox

1. **Configuración:** Ejecutar un comando de negocio que mute estado y registre un evento de dominio.
2. **Escenario de Fallo:** Forzar un error o excepción de base de datos antes del cierre transaccional.
3. **Aserción:** Se certifica que la transacción hizo rollback completo: ni la entidad ni la tupla en `outbox_events` deben existir en la base de datos.

### 5.3 Test Mandatorio de Idempotencia Asíncrona

1. **Configuración:** Enviar un mensaje con un `eventId` único al worker de entrada.
2. **Primera Ejecución:** El worker procesa la lógica, persiste el resultado y marca el evento como procesado.
3. **Segunda Ejecución (Replay):** Reenviar el mismo mensaje con idéntico `eventId`.
4. **Aserción:** El worker detecta el registro previo en la compuerta de idempotencia, cortocircuita la ejecución sin volver a mutar el estado y emite confirmación de éxito (*ACK*).

---

## 6. Gestión de Infraestructura de Test y Control Temporal

### 6.1 Contenedor Compartido Singleton (*Shared Singleton Container*)

Para garantizar que la suite de integración cumpla el presupuesto de tiempo de CI:

* Se prohíbe terminantemente iniciar y detener contenedores de base de datos por cada clase de prueba.
* La suite de tests debe configurar una instancia única de contenedor compartida a lo largo de todo el ciclo de ejecución del build.
* **Aislamiento de Estado:** La limpieza de datos entre tests se gestiona mediante reversión transaccional declarativa (`@Transactional` en pruebas) o mediante scripts de truncado rápido de tablas entre ejecuciones, sin reiniciar el contenedor.

### 6.2 Factorías de Datos (Patrón Object Mother / Builders)

* Se prohíbe instanciar manualmente entidades complejas o construir JSONs gigantes repetitivos dentro del cuerpo de los tests.
* Los datos de prueba se generan mediante factorías especializadas (*Object Mother* o *Builders*) ubicadas en el paquete de utilidades de test, las cuales proporcionan instancias válidas preconfiguradas con sobreescritura selectiva de campos.

### 6.3 Control Temporal Determinista

* Toda lógica de negocio que dependa de marcas temporales, expiraciones o cálculos de antigüedad debe consumir la abstracción de reloj del sistema (`UtcClockPort`).
* En las pruebas unitarias y de integración, se inyecta una implementación determinista fija (*Fake/Fixed Clock*), permitiendo simular el paso del tiempo o fijar instantes inmutables sin depender del reloj del hardware.

---

## 7. Guardianes de Arquitectura (*Fitness Functions*)

La suite de pruebas debe contener pruebas estáticas ejecutables automatizadas que certifiquen el cumplimiento de `ARCHITECTURE.md`. Si una regla se viola, el build falla inmediatamente:

| Guardián de Arquitectura | Regla Verificada | Condición de Fallo en Test |
| --- | --- | --- |
| **`Fronteras Modulares`** | `ARC-01`, `ARC-02` | Módulo A importa clases privadas (`domain`, `infrastructure`) de Módulo B. |
| **`Aislamiento de Dominio`** | `DOM-01` | Paquete `domain` depende de librerías web, frameworks o anotaciones ORM. |
| **`Inmutabilidad de Agregados`** | `DOM-04` | Clases en `domain.model` exponen métodos públicos que comiencen por `set*`. |
| **`Estructura de UseCases`** | `APP-01` | Clases en `application.service` no implementan exactamente un `*UseCase`. |
| **`Confinamiento de Persistencia`** | `OUT-01` | Entidades de base de datos residen fuera del adaptador secundario correspondiente. |
| **`Delimitación Transaccional`** | `TRX-01` | Anotaciones o demarcaciones transaccionales presentes fuera de la capa de aplicación. |

---

## 8. Matriz Pedagógica de Anti-Patrones y Checklist de Pull Request

### 8.1 Matriz de Anti-Patrones Comunes en Testing

| ❌ Anti-Patrón | 💥 Por qué falla | ✅ Solución Normativa | Regla Asociada |
| --- | --- | --- | --- |
| **Mockear el ORM / Persistencia** | La prueba pasa en verde pero oculta fallos de sintaxis SQL, violaciones de tipos o bloqueos en motor. | Probar la persistencia contra el motor real en contenedor efímero. | `TST-03` |
| **Pausas ciegas (`sleep`) en Asíncrono** | Provoca lentitud extrema y fallos intermitentes (*flaky tests*) por contención de CPU en CI. | Utilizar herramientas de sondeo condicional con timeout (*Awaitility*). | `TST-06` |
| **Tests con Dependencia de Orden** | Un test pasa únicamente si se ejecuta después de otro, ocultando polución de base de datos. | Limpieza atómica de datos entre tests; cada prueba es independiente. | `TST-08` |
| **Revalidar Dominio en el Test Web** | Multiplica el mantenimiento: si una regla de validación cambia, fallan 5 tests en capas distintas. | El test web solo valida códigos HTTP, serialización y headers. | `TST-04` |
| **Ignorar Pruebas de Multi-Tenancy** | Deja expuesta la mayor vulnerabilidad de seguridad: fuga accidental de datos entre clientes. | Implementar obligatoriamente el Test de Fuga de Inquilino (*Tenant Leakage*). | `TST-08` |
| **Reiniciar Contenedores por Clase** | Multiplica por 10 el tiempo de compilación, destruyendo la agilidad del CI. | Utilizar el patrón Singleton para compartir el contenedor en toda la suite. | `TST-03` |

### 8.2 Checklist de Verificación para Pull Requests

* [ ] ¿Las pruebas de Dominio están completamente libres de mocks y frameworks? (`TST-01`)
* [ ] ¿En los tests de Casos de Uso solo se falsean o mockean interfaces de salida `port/out`? (`TST-02`)
* [ ] ¿Toda nueva consulta de persistencia se prueba contra el motor de base de datos real en contenedor? (`TST-03`)
* [ ] ¿Los tests de controladores web verifican status HTTP, DTOs y errores sin duplicar lógica de negocio? (`TST-04`)
* [ ] ¿Se ha incluido la prueba de idempotencia en los nuevos consumidores de eventos o workers? (`TST-05`)
* [ ] ¿La suite está libre de llamadas a `sleep()`, usando sincronización condicional? (`TST-06`)
* [ ] ¿Se ejecutan y pasan en verde los guardianes automatizados de arquitectura en el build? (`TST-07`)
* [ ] ¿Las entidades o tablas nuevas con discriminante `tenant_id` cuentan con su respectivo Test de Fuga entre Inquilinos? (`TST-08`)
* [ ] ¿El tiempo total de ejecución de la suite de pruebas se mantiene dentro del presupuesto de CI (< 5 min)? (`TST-08`)