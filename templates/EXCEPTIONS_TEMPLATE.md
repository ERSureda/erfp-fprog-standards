# Manual de Excepciones y Errores — [<PATRÓN_O_LENGUAJE>]

> **Naturaleza del documento.** Especificación técnica normativa y Fuente Única de Verdad (SSOT) para el diseño, jerarquía, tipado, captura y serialización de excepciones en el Preset `[<NOMBRE_DEL_PRESET>]`.
>
> **Documento normativo asociado.** `ARCHITECTURE.md`. Implementa y desarrolla en detalle las reglas de arquitectura `INP-03` y `SHR-04`.
>
> **Fuerza normativa.** Las reglas etiquetadas como `MUST` son obligaciones técnicas estrictas cuyo incumplimiento bloquea el pipeline de integración continua o la aprobación de Pull Requests. Las reglas `NEVER` son prohibiciones taxativas que solo admiten excepciones mediante un registro de decisión formal (`ADR`) con condición de salida escrita.

---

## 1. Metadatos de Gobernanza y Fuerza Normativa

* **Preset / Ecosistema:** `[<NOMBRE_DEL_PRESET> — Ej: Java Spring Modulith / Python FastAPI / Node.js Clean Architecture]`
* **Versión del Estándar:** `v1.0.0`
* **Estado:** `[Propuesta / Normativo / En Deprecación]`
* **Repositorio Semilla Asociado:** `[<URL_O_NOMBRE_DE_LA_SEMILLA_EJECUTABLE>]`
* **Alcance:** Aplicable a todos los servicios, capas de aplicación, controladores web, brokers y workers asíncronos desarrollados bajo este stack técnico.

---

## 2. Filosofía y Principios de Gestión de Errores

El subsistema de errores y excepciones se rige por tres pilares de ingeniería inquebrantables:

### 2.1 Excepciones de Coste Cero o Ultrabajo (*Zero / Low-Overhead*)

En la mayoría de entornos de ejecución y máquinas virtuales, entre el 90% y el 95% del coste computacional al instanciar una excepción proviene de capturar, inspeccionar y rellenar la traza de pila de ejecución (*fillInStackTrace*).

* Las **excepciones de negocio, conflicto y validación sintáctica** son flujos de control alternativos normales y esperados, **no anomalías del sistema**.
* Toda excepción de negocio debe heredar de una abstracción base (`<BaseException>`) configurada para **desactivar la generación de trazas de pila en el runtime** cuando el error sea de naturaleza funcional. Esto evita la saturación de CPU y memoria en rutas críticas de alto tráfico (*hot-paths*).

### 2.2 Tratamiento Diferenciado: Negocio vs. Técnico

* **Excepciones de Negocio / Dominio:** Se registran con nivel `WARN` en los logs del servidor sin traza forense, informando únicamente del código semántico y el motivo funcional.
* **Excepciones Técnicas / Infraestructura (`INTERNAL`):** Representan fallos imprevistos del sistema o fallos de red (pérdida de conexión a base de datos, caídas de almacenamiento, timeouts de socket). Estas **sí capturan la traza forense completa** y se registran con nivel `ERROR` para posibilitar el diagnóstico posterior.

### 2.3 Seguridad por Defecto (*Safe-by-Default*)

* Ningún detalle interno de implementación (nombres de tablas, fragmentos de sentencias SQL, IPs, puertos o volcados de memoria) se expone jamás al exterior en errores técnicos.
* Todo fallo técnico no controlado es interceptado en el perímetro perimetral y enmascarado hacia el cliente con el mensaje genérico:

```text
"An unexpected error occurred"
```

---

## 3. Catálogo Normativo de Reglas de Error (`ERR`)

### 3.1 Identificadores de Regla

La notación sigue la estructura unívoca: **`ERR-nn · FUERZA [TIPO]`**.

* **`MUST`**: Obligación técnica estricta.
* **`NEVER`**: Prohibición técnica absoluta.
* **`[A]`**: Verificación automatizada obligatoria en el pipeline de CI (linters o pruebas estáticas).
* **`[R]`**: Verificación manual obligatoria en Pull Request mediante checklist.

### 3.2 Reglas del Sistema de Errores

* **`ERR-01 · MUST` [A]** Toda excepción lanzada en la capa de dominio o aplicación debe heredar directa o indirectamente de la clase base abstracta del Preset (`<BaseException>`).
* **`ERR-02 · MUST` [A]** Las excepciones funcionales y de validación deben desactivar la inspección de trazas de pila en el runtime (`capturesDiagnostics = false`).
* **`ERR-03 · MUST` [A]** Todo error de negocio debe asociarse obligatoriamente a un código alfanumérico inmutable y tipado mediante el contrato `ErrorCode`.
* **`ERR-04 · MUST` [A]** Toda respuesta de error en la API pública debe serializarse bajo el modelo estándar `ErrorResponse`, omitiendo el campo `errors` cuando no contenga violaciones de campo.
* **`ERR-05 · NEVER` [A]** Se capturarán excepciones de forma genérica para silenciarlas (`catch {}` vacío) ni se lanzarán tipos genéricos no tipados del lenguaje (ej. `RuntimeException`, `Exception`).
* **`ERR-06 · MUST` [R]** Todo fallo técnico en adaptadores secundarios (`InfrastructureException`, `ExternalServiceException`) debe encadenar obligatoriamente la causa técnica original (`cause`) para preservar la cadena forense.

---

## 4. Jerarquía Canónica de Excepciones

Ubicación transversal: `<shared-kernel>/domain/exception/`.

### 4.1 Árbol de Herencia

```text
[Tipo de Error Base del Runtime]
 └── <BaseException> (abstracta: porta ErrorCode y ErrorCategory)
      ├── ResourceNotFoundException       [404 NOT_FOUND]
      ├── ConflictException               [409 CONFLICT]
      ├── ValidationException             [400 VALIDATION]
      ├── ForbiddenException              [403 FORBIDDEN]
      ├── UnauthenticatedException        [401 UNAUTHENTICATED]
      ├── InfrastructureException         [500 INTERNAL]
      └── ExternalServiceException        [500 INTERNAL]
```

### 4.2 Catálogo de Excepciones Base

| Excepción | Categoría | Transporte HTTP | Comportamiento en Workers / Colas | Traza | Cuándo Utilizarla |
| --- | --- | --- | --- | --- | --- |
| **`ResourceNotFoundException`** | `NOT_FOUND` | **404** | Descarte / DLQ (No reintentable) | ❌ No | Una entidad o recurso solicitado no existe por su identificador o criterio. |
| **`ConflictException`** | `CONFLICT` | **409** | Descarte / DLQ (No reintentable) | ❌ No | Violación de reglas de unicidad o colisión con el estado del agregado. |
| **`ValidationException`** | `VALIDATION` | **400** | Descarte / DLQ (No reintentable) | ❌ No | Infracción de invariantes; encapsula lista inmutable de `FieldViolation`. |
| **`ForbiddenException`** | `FORBIDDEN` | **403** | Descarte / DLQ (No reintentable) | ❌ No | Actor autenticado sin permisos o falta de contexto de inquilino (`tenantId`). |
| **`UnauthenticatedException`** | `UNAUTHENTICATED` | **401** | Descarte / DLQ (No reintentable) | ❌ No | Se requiere una identidad válida y la petición llegó de forma anónima. |
| **`InfrastructureException`** | `INTERNAL` | **500** | Reintento con backoff ➔ DLQ | ✅ Sí | Fallo técnico imprevisto en persistencia, disco o base de datos. |
| **`ExternalServiceException`** | `INTERNAL` | **500** | Reintento con backoff ➔ DLQ | ✅ Sí | Fallo de red, timeout o respuesta corrupta de servicios downstream. |

---

## 5. Clasificación Semántica (`ErrorCategory`)

El enum o estructura cerrada `<ErrorCategory>` define la semántica del fallo y gobierna tanto el código de transporte como el comportamiento en flujos asíncronos y la captura de diagnóstico:

```text
ErrorCategory:
  VALIDATION       -> HTTP 400 | Async: No reintentable (DLQ) | Captura Traza: false
  UNAUTHENTICATED  -> HTTP 401 | Async: No reintentable (DLQ) | Captura Traza: false
  FORBIDDEN        -> HTTP 403 | Async: No reintentable (DLQ) | Captura Traza: false
  NOT_FOUND        -> HTTP 404 | Async: No reintentable (DLQ) | Captura Traza: false
  CONFLICT         -> HTTP 409 | Async: No reintentable (DLQ) | Captura Traza: false
  INTERNAL         -> HTTP 500 | Async: REINTENTABLE          | Captura Traza: TRUE
```

---

## 6. Contratos de Identificación y Salida Web

### 6.1 Contrato `ErrorCode`

Interfaz o contrato funcional que obliga a exponer un código alfanumérico estable (`code() -> String`):

* **Errores Comunes (`CommonError`):** Provistos por `shared` (`RESOURCE_NOT_FOUND`, `CONFLICT`, `VALIDATION_ERROR`, `FORBIDDEN`, `UNAUTHENTICATED`, `INTERNAL_ERROR`).
* **Errores de Módulo (`<Subdominio>Error`):** Enums o estructuras cerradas tipadas por Bounded Context (ej. `ORDER_ALREADY_SHIPPED`, `INSUFFICIENT_STOCK`) para que clientes frontend y consumidores programáticos identifiquen la causa exacta sin parsear texto.

### 6.2 Contrato Universal de Salida JSON (`ErrorResponse`)

Toda respuesta de error en la API pública emite estrictamente este formato ultraligero:

```json
{
  "status": 400,
  "code": "VALIDATION_ERROR",
  "detail": "Validation failed for one or more fields",
  "errors": [
    {
      "field": "email",
      "message": "must be a well-formed email address"
    }
  ]
}
```

* **Optimización de Ancho de Banda:** Cuando la lista de violaciones por campo está vacía (comportamiento estándar en errores 404, 409, 500, etc.), el campo `errors` **no se emite** en el payload serializado.

---

## 7. Responsabilidades del Manejador Centralizado

La captura y traducción de errores reside en adaptadores perimetrales dedicados:

### 7.1 Perímetro Síncrono (Web / API)

1. Intercepta cualquier subclase de `<BaseException>`.
2. Mapea la categoría semántica (`ErrorCategory`) a su respectivo código de transporte HTTP.
3. Si la categoría es `INTERNAL`: registra un log de nivel `ERROR` con la traza completa forense y enmascara el campo `detail` a `"An unexpected error occurred"`.
4. Si la categoría es de negocio: registra un log de nivel `WARN` sin traza y devuelve el mensaje descriptivo funcional.
5. Captura fallos de validación sintáctica de entrada (Bean Validation, esquemas de transporte) transformándolos en instancias de `ErrorResponse` con su lista de `FieldViolation`.
6. Captura cualquier excepción no controlada (`Throwable` / `Exception`) devolviendo un error 500 seguro y enmascarado.

### 7.2 Perímetro Asíncrono (Workers / Consumidores de Eventos)

1. Si la excepción es de negocio (`INTERNAL == false`): descarta el mensaje o lo envía directamente a la Dead-Letter Queue (DLQ) para evitar bucles infinitos de reintento sobre datos que jamás serán válidos.
2. Si la excepción es técnica (`INTERNAL == true`): programa el reintento con backoff exponencial incrementando el contador de intentos; superado el límite máximo configurado, deriva el mensaje a la DLQ con registro de la causa técnica original.

---

## 8. Matriz Pedagógica de Anti-Patrones y Checklist de Pull Request

### 8.1 Matriz de Anti-Patrones Comunes

| ❌ Anti-Patrón | 💥 Por qué falla | ✅ Solución Normativa | Regla Asociada |
| --- | --- | --- | --- |
| **Silenciar Excepciones (`catch {}` vacío)** | Provoca inconsistencia transaccional al impedir el rollback y oculta errores críticos en producción. | Dejar propagar las excepciones hacia el manejador central o relanzar excepciones tipadas. | `ERR-05`, `TRX-01` |
| **Manejadores Locales por Controlador** | Fragmenta el contrato de salida y rompe la estandarización del esquema `ErrorResponse`. | Centralizar toda captura y serialización de errores en el componente perimetral global. | `ERR-04`, `INP-03` |
| **Lanzar Excepciones Genéricas (`RuntimeException`)** | Oculta la naturaleza del fallo retornando HTTP 500 para errores funcionales y degrada métricas operativas. | Lanzar exclusivamente subclases tipadas derivadas de `<BaseException>` (`NotFound`, `Conflict`, etc.). | `ERR-01`, `ERR-03` |
| **Omitir Causa Original en Fallos Técnicos** | Destruye la cadena de error original, imposibilitando el diagnóstico forense en los logs del servidor. | Encadenar siempre el error original (`cause`) en constructores de excepciones de infraestructura. | `ERR-06` |
| **Reintentar Errores de Negocio en Colas** | Reintentar un payload con datos inválidos satura la cola, bloquea el procesamiento y desperdicia CPU. | Desviar directamente a DLQ o emitir ACK si el fallo es de categoría no técnica (`VALIDATION`, `CONFLICT`). | `ERR-02`, `TRX-05` |
| **Filtrar Datos Sensibles en Mensajes** | Los errores de negocio exponen su `detail` al cliente; emitir tokens, credenciales o secretos viola normativas de seguridad. | Restringir `detail` a explicaciones funcionales libres de datos personales o confidenciales. | `ERR-04`, `INP-03` |
| **Hardcodear Textos sin Código Semántico** | Obliga a clientes frontend y APIs consumidoras a parsear cadenas de texto frágiles en lugar de validar códigos estables. | Declarar enums o constantes que implementen `ErrorCode` por subdominio, separando el código del texto. | `ERR-03`, `SHR-04` |

### 8.2 Checklist de Verificación para Pull Requests

* [ ] ¿Toda nueva excepción introducida hereda directa o indirectamente de `<BaseException>`? (`ERR-01`)
* [ ] ¿Las excepciones funcionales o de validación desactivan la captura de traza en el runtime? (`ERR-02`)
* [ ] ¿Cada error funcional se asocia a un código alfanumérico tipado que implementa `ErrorCode`? (`ERR-03`)
* [ ] ¿Los fallos técnicos enlazan obligatoriamente el error original como causa (`cause`)? (`ERR-06`)
* [ ] ¿Los mensajes de detalle funcional (`detail`) están completamente libres de secretos, contraseñas o tokens? (`ERR-04`)
* [ ] ¿La respuesta JSON final en la API cumple estrictamente la estructura de `ErrorResponse` omitiendo `errors` cuando está vacío? (`ERR-04`)