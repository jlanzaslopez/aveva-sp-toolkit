# Alarmas y eventos

Fuente: *AVEVA Historian Retrieval Guide*, 3-dic-2020, pp. 175–182; contraste REST pp. 202–207. Citas internas. Consultas adaptadas y cotejadas con el manual, sin ejecución en Historian 2023.

## Entidad y columnas documentadas

Events expone historia de alarmas/eventos de Application Server en history blocks. No es EventHistory del Classic Event subsystem. La vista SQL no expone necesariamente todas las propiedades específicas de aplicación; REST/OData puede devolver más. Muchos campos pueden ser NULL.

Las consultas deben incluir al menos una columna datetime según p. 175; acotar temporalmente WHERE. `EventTimeUtc` es UTC fijo; `EventTime` es hora local/de recuperación y le afecta wwTimeZone, que **no** cambia EventTimeUtc (p. 176).

| Columnas usadas por el capítulo SQL | Uso observado |
|---|---|
| EventTime, EventTimeUtc | Instante del evento, según convención anterior |
| Alarm_ID | Correlación del ciclo de vida de alarma en ejemplos |
| Alarm_State | Ejemplos de activación filtran `UNACK_ALM` |
| Alarm_Acknowledged | Ejemplo selecciona valor 1 para reconocimientos |
| Type | Ejemplos SQL usan `Alarm.Acknowledged` y `Alarm.Clear` |
| Source_Area, Source_Object, Source_ConditionVariable | Localización e identidad de atributo/condición |
| Severity | Categoría de urgencia; no sustituye prioridad |
| User_Name | Usuario del evento |
| Alarm_UnAckDurationMs | Tiempo hasta reconocimiento en ms |
| Alarm_DurationMs | Duración de alarma hasta clear en ms |

`Comment` se describe para recurso REST Events (p. 202). No usar Annotation.Content como si fuese comentario de alarma. Confirmar en Database Reference/metadata si una propiedad adicional existe en la vista SQL antes de generarla.

## Listado UTC (pp. 175–176)

Base Runtime, nombres como en el manual:

```sql
SELECT EventTimeUtc, Alarm_ID, Type, Alarm_State,
       Source_Area, Source_Object, Source_ConditionVariable
FROM dbo.Events
WHERE EventTimeUtc >= '2023-11-01 00:00:00.000'
  AND EventTimeUtc <  '2023-11-02 00:00:00.000'
ORDER BY EventTimeUtc;
```

No añadir wwRetrievalMode de proceso por analogía: el capítulo 4 usa criterios propios de Events. La sintaxis de fechas REST con ISO/Z no se intercambia automáticamente con el SELECT SQL.

## Frecuencia por hora y condición (pp. 176–178)

Adaptación de los ejemplos que cuentan `Alarm_State='UNACK_ALM'`. Los campos reales de condición se mantienen separados, evitando concatenaciones ambiguas de nombres. Cada registro que cumpla el criterio cuenta; verificar si el origen repite publicaciones antes de interpretarlo como número de episodios únicos.

```sql
SELECT CAST(EventTimeUtc AS date) AS EventDateUtc,
       DATEPART(hour, EventTimeUtc) AS EventHourUtc,
       Source_Area, Source_Object, Source_ConditionVariable,
       COUNT(*) AS MatchingAlarmRecords
FROM dbo.Events
WHERE EventTimeUtc >= '2023-11-01 00:00:00.000'
  AND EventTimeUtc <  '2023-11-02 00:00:00.000'
  AND Alarm_State = 'UNACK_ALM'
GROUP BY CAST(EventTimeUtc AS date), DATEPART(hour, EventTimeUtc),
         Source_Area, Source_Object, Source_ConditionVariable
ORDER BY EventDateUtc, EventHourUtc, MatchingAlarmRecords DESC;
```

Los ejemplos originales usan tablas variables para materializar registros y luego agrupar. El SELECT directo es una simplificación que debe probarse contra el proveedor/versión; si la delegación falla, recuperar Events acotado y agrupar en SQL Server fuera de OPENQUERY o en cliente. No hacer GROUP BY dentro del texto de OPENQUERY.

Para «episodios únicos» definir deduplicación con Alarm_ID y transición/tipo verificados; no contar todos los estados (Set/Ack/Clear) como tres alarmas distintas. Para tasa media por hora incluir horas sin alarmas en el denominador: el AVG de grupos existentes del ejemplo pp. 176–177 **no incluye horas sin registros**.

## Reconocimiento y clear son métricas diferentes (pp. 178–182)

```sql
SELECT CAST(EventTimeUtc AS date) AS EventDateUtc,
       DATEPART(hour, EventTimeUtc) AS EventHourUtc,
       Severity,
       AVG(CAST(Alarm_UnAckDurationMs AS float)) AS MeanAckDurationMs,
       COUNT(Alarm_UnAckDurationMs) AS SamplesWithAckDuration
FROM dbo.Events
WHERE EventTimeUtc >= '2023-11-01 00:00:00.000'
  AND EventTimeUtc <  '2023-11-02 00:00:00.000'
  AND Type = 'Alarm.Acknowledged'
GROUP BY CAST(EventTimeUtc AS date), DATEPART(hour, EventTimeUtc), Severity;
```

Es una media de duración hasta **reconocimiento** para eventos de reconocimiento ocurridos en la ventana; no es tiempo hasta clear ni media de todas las alarmas iniciadas ese día. COUNT de la columna explicita cuántas duraciones no-NULL participan. El CAST a Float es una adaptación para no perder fracción al promediar enteros; no altera el dato almacenado.

Para clear, la fuente usa `Type='Alarm.Clear'` y `Alarm_DurationMs`. Para ciclo de vida, pp. 180–181 correlacionan activación/ack/clear por Alarm_ID. Ese ejemplo hace INNER JOIN con clear: **excluye alarmas aún abiertas**. Filtrar los tres estados por la misma ventana también pierde episodios que cruzan límites. Antes de generar un informe, decidir cohorte (iniciadas, cerradas, reconocidas o activas en intervalo) y política para episodios incompletos; usar intervalos de búsqueda suficientes y comprobar cardinalidad del join.

## Inconsistencias de nombres y categorías

- El encabezado pp. 178–179 dice «time to clearing» pero filtra Alarm.Acknowledged y promedia Alarm_UnAckDurationMs: es tiempo de reconocimiento. No etiquetarlo como clear.
- El capítulo SQL etiqueta Severity 1/2/3/4 como critical/high/medium/low; REST p. 202 lo etiqueta Critical/Major/Minor/Informational. Conservar valor numérico y verificar nomenclatura de la instalación/interfaz; no mezclar con Priority 1..999.
- SQL usa `Alarm.Acknowledged`; el catálogo REST imprime `Alarm.Acknowledge`. No corregir uno a partir del otro sin comprobar Type real.
- `Alarm_ID` no es automáticamente la misma columna que ID de evento REST. Verificar esquema y correlación; no inventar conversiones GUID ni claves.

Estas consultas son de lectura. Una petición de análisis de alarmas no autoriza reconocerlas, borrar registros o cambiar sus prioridades.
