# Consultas SQL cotejadas

Fuente: *AVEVA Historian Retrieval Guide*, 3-dic-2020. Ejemplos **adaptados y cotejados documentalmente**, no ejecutados contra Historian 2023. Usan vistas en la base Runtime y linked server INSQL cuando corresponde. Sustituir nombres de tags de ejemplo por nombres existentes. No se incluyen credenciales ni endpoints de planta.

Contenido: comprobaciones previas; muestras; WideHistory; promedio/integral; contador; estados; lotes; calidad; límites; mapa del capítulo 3.

## Antes de consultar

Los ejemplos siguientes usan zona `UTC` solo si aparece en la tabla TimeZone de la instalación. Comprobación de lectura basada en p. 111:

```sql
USE Runtime;
SELECT TimeZone
FROM dbo.TimeZone
WHERE TimeZone = 'UTC';
```

Si no aparece, escoger el nombre Windows equivalente que exista y adaptar **también ambos límites**. `USE Runtime` solo selecciona contexto de sesión; no crea bases ni cambia configuración del servidor. Fechas de ejemplo son intervalos UTC fijos; no implican que existan datos de esos días.

## Cambios y puntos almacenados (pp. 30–37, 143, 149–153)

History normalizado permite varios tipos; vValue conserva String/Discrete/Analog en un resultado. Para reconstruir el estado al inicio se usa `>=`, que puede crear una fila inicial con Quality=133:

```sql
SELECT DateTime, TagName, vValue, Quality, OPCQuality, QualityDetail
FROM dbo.History
WHERE TagName IN ('FlowRate', 'PumpStatus', 'BatchNumber')
  AND DateTime >= '2023-11-01 00:00:00.000'
  AND DateTime <  '2023-11-01 01:00:00.000'
  AND wwRetrievalMode = 'Delta'
  AND wwVersion = 'Latest'
  AND wwTimeZone = 'UTC'
ORDER BY DateTime, TagName;
```

Para todos los puntos almacenados cambiar **solo** modo a Full; no añadir Resolution esperando muestreo. Full no recupera muestras descartadas al almacenar. Para excluir cualquier inicial generado usar `>` y documentar que también excluye cambios físicos exactamente en el inicio.

Al concatenar extracciones `[inicio,fin)`, una fila inicial sintética de Delta/Full no debe contarse como un evento físico nuevo. Identificarla con Quality y mantener contexto de frontera según el cálculo.

## WideHistory con intervalo regular (pp. 17–19, 38, 130–132)

```sql
SELECT DateTime, [ReactTemp], [ReactLevel]
FROM OPENQUERY(INSQL, '
SELECT DateTime, [ReactTemp], [ReactLevel]
FROM WideHistory
WHERE DateTime >= "2023-11-01 00:00:00.000"
  AND DateTime <  "2023-11-01 01:00:00.000"
  AND wwRetrievalMode = "Cyclic"
  AND wwResolution = 60000
  AND wwTimeStampRule = "End"
  AND wwVersion = "Latest"
  AND wwTimeZone = "UTC"
')
ORDER BY DateTime;
```

La consulta solicita un punto por minuto con último valor conocido; no es promedio de ese minuto. ORDER BY es exterior. Si hace falta calidad por tag, usar History normalizado con sus columnas de calidad o verificar el esquema wide en Database Reference; no inventar `[Tag.Quality]`.

Con modo Delta la fila se genera cuando cambia cualquiera de los tags; los demás conservan su valor previo. Para nombres con guion, barra, puntos u otros signos usar identificadores delimitados documentados, por ejemplo `[ReactTemp-2]` (p. 138). Validar nombres reales y escapar `]` si se construye SQL dinámico; este ejemplo usa una lista fija.

## Promedio diario ponderado por tiempo (pp. 46–50, 95–97, 171–173)

Tag FlowRate analógico. Start con `[inicio,fin)` para que cada timestamp identifique el día que empieza, sin agregar el día previo. El mes de ejemplo tiene 30 días UTC:

```sql
SELECT StartDateTime, DateTime, TagName, Value,
       OPCQuality, QualityDetail, PercentGood, TimeGood
FROM dbo.History
WHERE TagName = 'FlowRate'
  AND DateTime >= '2023-11-01 00:00:00.000'
  AND DateTime <  '2023-12-01 00:00:00.000'
  AND wwRetrievalMode = 'Average'
  AND wwResolution = 86400000
  AND wwTimeStampRule = 'Start'
  AND wwInterpolationType = 'Linear'
  AND wwQualityRule = 'Good'
  AND wwVersion = 'Latest'
  AND wwTimeZone = 'UTC'
ORDER BY DateTime;
```

Linear es una **elección explícita de este ejemplo**, no una obligación para todo caudal. Si el sensor representa escalones/setpoints, elegir StairStep. El resultado esperado estructural es una fila por día para el tag compatible; comprobar los períodos y cobertura reales, sin prometer que todas sean numéricas o Good.

Average excluye tiempos inválidos con Good; un promedio de un día con huecos puede ser numérico. No convertirlo directamente a volumen multiplicándolo por 24 h. Para combinar días, ponderar por su duración válida, no `AVG(Value)` de los promedios; si son mediciones por unidad producida, la media aritmética de muestras **sí** puede ser la pregunta adecuada.

## Volumen de caudal (pp. 59–62, 111–118)

Consultar previamente las unidades y el divisor sin modificar configuración (join documentado pp. 20/60):

```sql
SELECT t.TagName, e.Unit, e.IntegralDivisor
FROM dbo.AnalogTag AS t
JOIN dbo.EngineeringUnit AS e ON t.EUKey = e.EUKey
WHERE t.TagName = 'FlowRate';
```

Integral diario; la unidad del resultado depende del tag/divisor, **no** del alias elegido:

```sql
SELECT StartDateTime, DateTime, TagName, Value, wwUnit,
       OPCQuality, QualityDetail, PercentGood, TimeGood
FROM dbo.History
WHERE TagName = 'FlowRate'
  AND DateTime >= '2023-11-01 00:00:00.000'
  AND DateTime <  '2023-12-01 00:00:00.000'
  AND wwRetrievalMode = 'Integral'
  AND wwResolution = 86400000
  AND wwTimeStampRule = 'Start'
  AND wwInterpolationType = 'Linear'
  AND wwQualityRule = 'Good'
  AND wwVersion = 'Latest'
  AND wwTimeZone = 'UTC'
ORDER BY DateTime;
```

Sumar solo ciclos del intervalo y con unidad comparable, conservando el informe de cobertura. Con huecos, la suma con Good representa la contribución conocida, no necesariamente volumen total real del mes; no imputar cero a la pérdida ni extrapolar sin autorización metodológica. Para un solo agregado mensual usar CycleCount=1 con los mismos límites/Start, retirando Resolution; comprobar StartDateTime.

Comprobación matemática sin servidor: si el caudal es constante 3.5 L/s, 60 s válidos dan 210 L; 30 s válidos dan 105 L conocidos. Un Average de 3.5 sobre solo esos 30 s no justifica declarar 210 L en un minuto con huecos.

## Totalizador con rollover (pp. 65–69)

```sql
SELECT StartDateTime, DateTime, TagName, Value, wwUnit,
       OPCQuality, QualityDetail, PercentGood
FROM dbo.History
WHERE TagName = 'ProductionCounter'
  AND DateTime >  '2023-11-01 00:00:00.000'
  AND DateTime <= '2023-11-01 01:00:00.000'
  AND wwRetrievalMode = 'Counter'
  AND wwCycleCount = 1
  AND wwTimeStampRule = 'End'
  AND wwQualityRule = 'Good'
  AND wwVersion = 'Latest'
  AND wwTimeZone = 'UTC';
```

Requiere rollover, rango, signed/unsigned y reset/deadband correctos en el tag. Con rollover=10000 y extremos 9900→100, incremento=200 si hubo un rollover y cobertura suficiente. No verificar un contador solo por esa resta si puede haber huecos o varios rollovers. Rollover=0 y reset manual contestan otra interpretación; no cambiar configuración desde la consulta.

## Duración por estado, conservando desconocidos (pp. 70–75, 119–120)

PumpStatus debe ser tipo compatible (Integer/Discrete/String/state summary). End con `(inicio,fin]` sigue el ejemplo Total de p. 73 y evita sumar el phantom inicial:

```sql
SELECT StartDateTime, DateTime, TagName, vValue,
       StateTime, OPCQuality, QualityDetail
FROM dbo.History
WHERE TagName = 'PumpStatus'
  AND DateTime >  '2023-11-01 00:00:00.000'
  AND DateTime <= '2023-11-02 00:00:00.000'
  AND wwRetrievalMode = 'ValueState'
  AND wwStateCalc = 'Total'
  AND wwCycleCount = 1
  AND wwTimeStampRule = 'End'
  AND wwQualityRule = 'Good'
  AND wwVersion = 'Latest'
  AND wwTimeZone = 'UTC';
```

StateTime está en ms; dividir por 3600000.0 para horas en la presentación. Mantener fila NULL como tiempo desconocido en vez de sumarlo a Off. No filtrar inicialmente por un único estado si se necesita auditar que la suma temporal cierre. El texto «On/Off» puede depender de configuración: no asumir que vValue String y Value numérico son intercambiables.

Para episodios completos usar Contained; para tiempo entre arranques usar RoundTrip y AvgContained. No cambiar Total por AvgContained pretendiendo la misma disponibilidad. En summary tags verificar que sus períodos coincidan con los ciclos solicitados.

## Lotes definidos por otro tag (pp. 134–138)

```sql
SELECT SliceByValue, TagName, StartDateTime, EndDateTime,
       Average, Integral, OPCQuality, PercentGood
FROM dbo.AnalogSummaryHistory
WHERE TagName = 'M31.FlowIn'
  AND SliceBy = 'M31.Batch'
  AND StartDateTime >= '2023-11-01 00:00:00.000'
  AND EndDateTime <=   '2023-11-02 00:00:00.000'
  AND wwTimeZone = 'UTC';
```

Incluye solo lotes que comienzan y terminan dentro. Para lotes que **solapan** el intervalo, los ejemplos invierten criterios a `EndDateTime >= inicio AND StartDateTime <= fin`; eso puede devolver períodos completos fuera del reporte, no los recorta automáticamente. Definir si el KPI es por lote completo o solo por tramo del día. Lote aún abierto/latencia puede devolver OPCQuality=64 y estadísticas provisionales.

Promedio global por estado/lote no es AVG de promedios: con duraciones distintas usar ponderación; con huecos usar el tiempo válido de cada agregado. Integral se puede sumar únicamente para períodos no solapados, unidades consistentes y cobertura conocida.

## Metadatos de calidad y anotaciones (pp. 149, 168)

Join al mapa OPC en SQL Server; por claridad, sobre rowset de History ya acotado o cuatro partes compatible:

```sql
SELECT h.DateTime, h.TagName, h.Value, h.OPCQuality,
       h.QualityDetail, o.Description
FROM dbo.History AS h
JOIN dbo.OPCQualityMap AS o
  ON (h.OPCQuality & 255) = o.OPCQuality
WHERE h.TagName = 'FlowRate'
  AND h.DateTime >= '2023-11-01 00:00:00.000'
  AND h.DateTime <  '2023-11-01 01:00:00.000'
  AND h.wwRetrievalMode = 'Full'
  AND h.wwTimeZone = 'UTC';
```

Los metadatos de tags y unidades describen la configuración consultada; no se han probado como versionados históricamente. `Annotation.Content` de p. 149 es una entidad legacy diferente de `Events.Comment` de alarmas; pedir/consultar la entidad correcta. No incluir el INSERT de anotación del original en una consulta de lectura.

## Pruebas de fronteras que debe superar una instalación (pp. 149–159)

Sobre un tag de referencia que cambia cada segundo y un minuto completo:

| Caso Cyclic con End | Parámetros/intervalo | Filas previstas por tag |
|---|---|---|
| Malla fija incluyendo ambos extremos | Resolution=1000; `>= 00:00:00`, `<= 00:01:00` | 61 |
| Malla fija excluyendo fin | Resolution=1000; `>= 00:00:00`, `< 00:01:00` | 60 |
| Count fijo incluyendo ambos | CycleCount=60; intervalo de 59 s cerrado | 60, separación 1 s |
| Count fijo con solo inicio inclusivo | CycleCount=60; intervalo de 60 s `[inicio,fin)` | 60, separación 1 s |

Son expectativas del modelo documentado, no pruebas realizadas. Añadir casos: sin valor previo, NULL intermedio, final sin punto posterior, tag string con modo incompatible, DST, summary desalineado y retorno vacío por opciones. No introducir Full sobre meses para probar un solo caso de frontera.

## Cobertura del capítulo 3 y erratas (pp. 129–173)

| Bloque | Tratamiento en esta skill |
|---|---|
| History/Live/WideHistory; Delta wide (129–132) | querying.md y ejemplos anteriores; Live no sustituye último histórico |
| Analog/StateSummaryHistory (132–134) | Esquema observado y períodos propios; no aplicar criterio History a Summary |
| SliceBy y GROUP BY ponderado (134–138) | Lotes contenidos vs solapados y ponderación |
| Tags no convencionales/REMOTE JOIN (138–139) | Identificadores y delegación en querying.md |
| Deadbands y conflicto Resolution/Count (139–143) | options-quality-time.md |
| Variantes, fechas, GROUP BY/COUNT y agregación wide (143–148) | Parser SQL exterior/interior y medias distintas |
| Anotaciones (149) | Distinguir escritura/lectura y entidad legacy |
| Comparaciones Delta/Cyclic (149–160) | Fronteras e iniciales; rejilla no es filtro ordinario |
| Duraciones wwResolution (160–165) | NULL/−1/0, siguiente punto y multitag |
| SELECT INTO, traslado a History (165–167) | Escrituras fuera del alcance de consulta; no ejecutar DROP/INSERT de ejemplo |
| Cursores/stored procedures (167–168) | Encapsulación no amplía SQL del proveedor |
| OPCQualityMap/variables Wide (168–169) | Máscara 255 y texto dinámico validado |
| Huecos Classic y tiempos no válidos (169–170) | Mantener NULL/marcadores, no completar con cero |
| Cuatro vías de agregación (171–173) | AVG muestral, Average temporal, summary replication, summary de eventos clásico; no equivalentes por nombre |
| Bitwise (173) | SQL Server: `CONVERT(BIT, CAST(Value AS INT) & 1)` para bit0, máscaras 2/4/8… para otros, hasta 32 bits con INT. Añadir límites y modo explícitos que el original omite |

Erratas: p. 88 Bounding omite FROM y operador de TagName. Una adaptación válida estructuralmente es `SELECT DateTime, TagName, Value FROM History WHERE TagName='BatchNumber' AND DateTime >= '2023-11-01 00:00:00.000' AND wwRetrievalMode='StartBound'`; añadir zona elegida y comprobar punto real en/antes del instante. EndBound cambia el modo y busca después, no un interpolado.

P. 165 dice que `Total>0` significa «más de un tag True» pero para dos booleanos significa **al menos uno**; para ambos sería Total=2. No copiar esa interpretación. P. 57 menciona MIN bajo Maximum. Varios timestamps/resultados de tablas no concuerdan con resolución: esta skill no los presenta como resultados ejecutados.
