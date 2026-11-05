# Límites de interfaz: REST/OData y SSRS

Fuente: *AVEVA Historian Retrieval Guide*, 3-dic-2020, capítulo 5, pp. 183–243. Esta referencia orienta la elección de interfaz y conserva diferencias relevantes para SQL. **No es un SDK REST completo ni verificación de endpoints/autenticación actuales**. Para implementar una llamada concreta consultar `$metadata`/documentación de la instalación, no extrapolar sintaxis SQL.

## REST no es INSQL (pp. 183–190)

La guía documenta Historian Data REST API v1/v2 para Insight y Historian Insight. Son versiones del API, no versiones OData ni del producto. En la fuente 2020 v2 es la recomendada; no afirmar por ello que sea la versión vigente de cualquier instalación 2023.

Recuperación usual por GET; upload por POST es otra operación. El endpoint debe obtenerse de la solución/instalación autorizada. Las URLs `online.wonderware.com/s/<solution_ID>/apis/historian/<api_version>/...` son ejemplos históricos; no reutilizar el identificador de ejemplo ni inventar un endpoint de planta.

En 2020: on-premises iHistory usa Windows Negotiate y grupos aaAdministrators/aaPowerUsers/aaUsers para datos; cloud describe OpenID Connect/bearer y Basic/retrieval tokens. Son descripciones de la fuente, no recomendación de habilitar métodos ni garantía de disponibilidad actual. Nunca incluir secretos/tokens en la skill o en URLs compartidas.

| Diferencia | Contrato de la fuente |
|---|---|
| Timestamps v1 | `datetimeoffset'2016-05-14T00:00:00.000-07:00'` en ejemplos |
| Timestamps v2 | No usar prefijo literal datetimeoffset; normalmente sin comillas, excepto Events, que usa comillas simples (p. 184) |
| Campos temporales | ProcessValues/Raw usan DateTime; resúmenes StartDateTime y EndDateTime |
| Tipos abiertos v2 | TagProperty/Events admiten propiedades dinámicas; inspeccionar `$metadata` |
| Summary combinado | Solo v1; en v2 usar AnalogSummary/StateSummary |
| Nombres de propiedades | Case-sensitive; el propio manual alterna OPCQuality/OpcQuality en tablas/JSON: tomar nombre exacto del metadata real |
| Paginación | Hasta 5000 registros por página; seguir el enlace devuelto, no asumir que primera página es el resultado completo |

OData: `$filter`, `$select`, `$skip`, `$skiptoken`, `$top`; operadores eq/ne/gt/ge/lt/le/and/or/not. Codificar URI correctamente (`%20` espacio, `%27` comilla); no pegar el `+` de ejemplos del visor como regla universal de codificación. La guía limita a una cláusula temporal combinada con una de filtrado mediante and; la cláusula de filtro puede ser compleja con paréntesis (p. 189).

REST usa **RetrievalMode** y **Resolution** (ms), sin prefijo ww. Los modos enumerados para Raw/ProcessValues son Average, Cyclic, Integral, Minimum, BestFit, Delta, Interpolated, Slope, Counter, Full, Maximum (pp. 186–187). No añadir ValueState/RoundTrip a ese enum por aparecer en SQL; StateSummary es un recurso diferente. Las entidades summary documentan sus propios modos Cyclic/Full y default Cyclic.

## Descubrimiento y recursos del capítulo 5

| Recurso GET | Uso y campos de identidad/tiempo de la guía | Páginas |
|---|---|---|
| `/ProcessValues` | VTQ; FQN `datasource.tagname`, DateTime, Value/Text y calidad; ejemplo v2 con fechas UTC ISO/Z | 191–192 |
| `/AnalogSummary` | FQN, StartDateTime/EndDateTime, estadísticas First/Last/Minimum/Maximum/Average/StdDev/Integral/Count, calidad/cobertura; Resolution y SliceBy/SliceByValue | 193–198 |
| `/StateSummary` | FQN y período; duración Average/Minimum/Maximum/Total y variantes Contained, porcentajes y estados según metadata | 199–201 |
| `/Events` | ID de evento, EventTime UTC, Type, Priority, Severity, EventTimeUTCOffsetMins, ReceivedTime, IsAlarm, Comment y propiedades específicas | 202–207 |
| `/Tags` | FQN, Source, Description, EngUnit, límites EU, InterpolationType, IntegralDivisor y propiedades extendidas en v2 | 208–213 |
| `/TagProperties` | Catálogo de propiedades base/extendidas, Name/Type/ReadOnly en respuesta | 214 |
| `/TagPropertyValues` | Name/Value y navegación Tags; la tabla imprime `/TagpropertyValues` y ejemplo otra capitalización: verificar metadata | 215 |
| `/TagGroups` | Fuentes/grupos: GroupID, GroupName, TypeID, ParentID, Scope y navegación | 216–217 |
| `/TagSuggest` | Sugerencias/contenido: q y Type como opciones; ranking/recuento y nombres | 218–219 |
| `/TagSearch` | q, kn, kv; FQN/Source/TagName/DisplayText/SearchRanking; algunas implementaciones pueden devolver vacío | 220 |
| `/TagExtendedProperties` | FQN, PropertyName, Value/Text/Type; tipos Edm y geo según tabla | 221 |
| `/Summary` | Resumen combinado, solo v1, Resolution y período | 222 |
| `/Daily` | Solo v1, resumen 24 h, default última semana; admite TimeZone propio del API | 223–228 |
| `/Hourly` | Solo v1, default últimas 24 h | 229–230 |
| `/Minutely` | Solo v1, default última hora | 231 |
| `/SystemParameters` | Solo on-premises; nombres/valores de parámetros | 232 |

Tags también documenta POST/DELETE. La presencia de esos métodos no autoriza modificar metadatos por una tarea de consulta. Parámetros que la tabla llama «required» a veces tienen defaults en los ejemplos (p. ej. fechas de Summary); una adaptación debe fijar límites explícitos para ser reproducible.

pp. 194–195 describen Average temporal, pero las fórmulas impresas de Integral repiten una división por tiempo propia del promedio. **No usar esa fórmula como prueba de unidades**: contrastar con la definición integral pp. 59–62 y la salida real. PercentGood de REST se describe allí como ratio de filas, mientras el capítulo SQL habla de cobertura temporal; no intercambiar ambos porcentajes como si tuviesen definición garantizada idéntica.

## TagFilter v2 (pp. 184, 186–188, 237–240)

Aplica a ProcessValues, AnalogSummary y StateSummary, en GET/POST de recuperación según servicio. Hasta 20 cláusulas AND o 20 OR, hasta 2 UDF; no mezclar and/or ni agrupación con precedencia. Operadores en minúsculas. No toupper/tolower; StartsWith/EndsWith documentados, con limitaciones para propiedades no String. InterpolationType/MessageOn/MessageOff tienen búsquedas sensibles a mayúsculas. EngUnitMax/EngUnitMin y algunas propiedades solo admiten eq. No confundir estas reglas con las más amplias de `$filter`.

## Herramientas cliente y SSRS (pp. 232–242)

La guía muestra browser, Postman, Excel y Power BI como clientes OData: obtener endpoint real, autenticar y seguir paginación. Son flujos de cliente, no nuevas tablas SQL. No reproducir capturas/versiones de menús de 2020 como instrucciones actuales verificadas.

SSRS: extensión **Historian OData Provider**, configurada mediante Configurator > Historian > Reporting. Connection string de ejemplo `Server=localhost:32569`; host/puerto deben verificarse. Acepta SQL-style, OData o SQL dentro del parámetro `sql=` de URL. Esa sintaxis no implica que todas las expresiones T-SQL/INSQL estén soportadas por el traductor OData. El ejemplo de p. 242 omite una letra de Site2 en una URL: no copiar literalmente un endpoint defectuoso.

## Errores REST (pp. 242–243)

| Estado de la fuente | Códigos documentados | Acción de diagnóstico |
|---|---|---|
| 304 | ConditionNotMet | Revisar condición de lectura/cache; no asumir respuesta con datos |
| 400 | MissingRequiredQueryParameter, UnsupportedQueryParameter, InvalidQueryParameterValue, OutOfRangeQueryParameterValue, RequestUrlFailedToParse, InvalidUri, InvalidHttpVerb, OutOfRangeInput, InvalidAuthenticationInfo, InvalidInput | Revisar versión, nombres/tipos, URI y headers; no repetir la misma petición inválida |
| 401 | Unauthorized | Verificar sesión/permiso sin exponer credenciales |
| 404 | ResourceNotFound | Verificar endpoint, recurso e identidad de tag |
| 500 | InternalError, OperationTimedOut | Registrar contexto; reducir ventana/volumen, reintento limitado solo de lectura |
| 503 | ServerBusy | Esperar/reintentar de forma limitada, no lanzar ráfagas |

La tabla es la de 2020, no un contrato exhaustivo del servidor actual. Una autenticación fallida o propiedad desconocida requiere comprobar servicio/metadata; no cambiar configuración de seguridad automáticamente.
