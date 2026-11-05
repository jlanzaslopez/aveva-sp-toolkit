# SQL Server, INSQL y tablas

Fuente: *AVEVA Historian Retrieval Guide*, 3-dic-2020. Citas internas; objetivo 2023 sin pruebas en servidor. Contenido: arquitectura, tablas/columnas, sintaxis, restricciones, metadatos y diagnóstico.

## Arquitectura y acceso (pp. 11–18)

Los valores históricos están en **history blocks**, fuera de los MDF de SQL Server. El proveedor OLE DB INSQL solicita datos al subsistema de recuperación y expone rowsets/tablas virtuales. Metadatos/configuración y algunos datos de eventos clásicos sí están en tablas SQL; no confundirlos con Events del capítulo 4.

INSQL no se usa en modo standalone: el cliente conecta a SQL Server. La instalación crea el linked server INSQL y, para joins legacy analógico/discreto, el alias INSQLD. Religar con sp_addlinkedserver/sp_serveroption/sp_addlinkedsrvlogin es administración, no una corrección que ejecutar por una petición de consulta (p. 14).

| Acceso | Forma documentada | Cuándo |
|---|---|---|
| Cuatro partes | `INSQL.Runtime.dbo.History` | Preferido cuando sea posible; Runtime=catalog y dbo=schema |
| Vista local | `History` o `dbo.History` en Runtime | Vistas con nombre de la tabla del proveedor; legacy suele usar `v_...` |
| Pass-through | `OPENQUERY(INSQL, 'SELECT ... FROM History WHERE ...')` | Texto enviado a INSQL sin reformulación de SQL Server; **necesario para WideHistory** |
| Ad hoc | `OPENROWSET('INSQL',' ', 'SELECT ...')` | Documentado, puede estar deshabilitado por seguridad; no activar automáticamente |

Toda tabla de extensión salvo HistoryBlock requiere WHERE. El proveedor no conserva opciones entre consultas de una misma conexión ni genera un stream continuo: repetir los parámetros ww requeridos en cada consulta.

## Entidades documentadas (pp. 13, 129–138, 143, 171–175)

| Entidad | Uso |
|---|---|
| `History` | Formato normalizado: DateTime, TagName y valor por fila; acepta mezcla de tipos con vValue |
| `Live` | Último valor **streamed** disponible. No necesariamente último histórico: store/forward/CSV, revisiones y reinicio sin nuevos valores pueden no actualizar Live (p. 130) |
| `WideHistory` | Esquema dinámico, tags como columnas; hasta 1024 columnas; solo OPENQUERY |
| `StateWideHistory` | Referido para consultas de estado/RoundTrip y filtros (pp. 75, 120); comprobar columnas concretas en Database Reference |
| `AnalogSummaryHistory` | Vista/tabla de resúmenes analógicos; múltiples estadísticas por tag y período. Sus nombres StartDateTime/EndDateTime sí son criterios documentados |
| `StateSummaryHistory` | Estadísticas por estado; una fila por estado/período |
| `Events` | Alarmas/eventos de Application Server en history blocks; no confundir con Classic Event subsystem |
| `HistoryBlock` | Información de bloques; excepción al WHERE obligatorio |
| `Tag`, `AnalogTag`, `DiscreteTag`, `StringTag`, `EngineeringUnit`, `TimeZone` | Metadatos consultados por los ejemplos; esta guía no define su esquema completo |
| `QualityMap`, `OPCQualityMap` | Mapas de calidad; usar codificación correcta de cada columna |
| `Annotation`, `UserDetail` | Anotaciones legacy de p. 149; no son el Comment de Events |
| `EventHistory`, `SnapshotTag`, `v_SummaryData`/`SummaryData` | Subsistema clásico y ejemplos pp. 20–23, 167–173; no asumir que describen las alarmas actuales |

Legacy proceso: AnalogHistory/DiscreteHistory/StringHistory y AnalogLive/DiscreteLive/StringLive. Legacy eventos: v_AlarmEventHistoryInternal2→LegacyAlarmEventHistory; v_AlarmHistory→LegacyAlarmHistory; v_AlarmHistory2→LegacyAlarmHistory2; v_EventHistory→LegacyEventHistory; v_AlarmEventHistory2 equivale a v_AlarmEventHistoryInternal2 (p. 13). El manual identifica History y las tres *History legacy de proceso como actualizables; las otras extensiones son lectura. Esa capacidad no autoriza escribir.

### Columnas empleadas por los ejemplos, no esquema exhaustivo

| Contexto | Columnas documentadas |
|---|---|
| History | DateTime, TagName, Value, vValue, Quality, OPCQuality, QualityDetail, StartDateTime, StateTime, PercentGood, TimeGood, wwResolution y otros parámetros ww según modo |
| AnalogSummaryHistory | TagName, StartDateTime, EndDateTime, First, Last, Minimum, Maximum, Average, Integral, OPCQuality, PercentGood; SliceBy/SliceByValue en pp. 134–138 |
| StateSummaryHistory | TagName, Value, StartDateTime, EndDateTime, OPCQuality, StateCount, StateTimeTotal, StateTimePercent, StateTimeMin, StateTimeMinContained (p. 133) |
| AnalogTag/EngineeringUnit | EUKey; EngineeringUnit.Unit e IntegralDivisor en ejemplos y cálculo integral (pp. 20, 60); AnalogTag.InterpolationType (p. 39) |

Value es apropiado para datos numéricos. vValue permite mezclar tipos mediante sql_variant; el proveedor lo entrega a SQL Server como string para ciertos filtros y requiere conversión explícita (pp. 143–144). No convertir tags String a Float por comodidad.

## Límites del SQL enviado al proveedor (pp. 17–25, 144–148)

| Construcción | Cuatro partes/vista | Dentro del texto OPENQUERY |
|---|---|---|
| ORDER BY | Sí | No; ordenar en SELECT exterior |
| GROUP BY | Sí | No; agrupar fuera |
| TagName IN/LIKE | Sí, con límites | Sí, con límites |
| Funciones fecha/hora | Sí | Tabla p. 18 indica sí; comprobar expresión concreta |
| MIN/MAX/AVG/SUM | Sí | Sí; STDEV no figura soportado dentro |
| COUNT(*) | Sí | No; contar fuera (p. 146) |
| Joins implícitos/explícitos | Solo joins compatibles con proveedor | No; joins fuera de OPENQUERY |
| Subselect tabla SQL + extensión | Con restricciones y coste | No |
| Subselect dos extensiones | No | No |
| CONVERT sobre vValue | En SQL Server | No; convertir/filtrar fuera |

OPENQUERY: texto máximo **8000 caracteres** según la guía; no admite variables directamente. Usar fechas `yyyy-mm-dd hh:mm:ss.fff` dentro del pass-through. Los ejemplos usan cadena SQL exterior con comilla simple y strings interiores con comilla doble. No trasladar esas comillas dobles a T-SQL normal sin revisar QUOTED_IDENTIFIER.

En SQL dinámico para WideHistory (pp. 168–169), validar listas de tags y escapar identificadores/comillas; fechas provenientes de variables tipadas se formatean antes de construir el texto. Un `sp_executesql` exterior no parametriza el string interno de OPENQUERY. No concatenar entradas sin validación. Preferir History parametrizado si no hace falta formato wide.

LIKE solo se documenta sobre TagName y Value, este último solo para strings. Los subselects con LIKE de AnalogTag/DiscreteTag/StringTag en IN tienen restricción de devolver vValue; suele ser más eficiente INNER REMOTE JOIN. El patrón recomendado es **tabla SQL a la izquierda**, extensión Historian a la derecha (pp. 19–23, 139).

No usar IN/OR para varios valores de una misma extensión ww ni para combinar ramas de opciones ww. Los valores de las columnas virtuales son insensibles a mayúsculas, aunque nombres SQL puedan depender de collation. Mantener espacios alrededor de operadores aritméticos dentro de consultas del proveedor (p. 147).

StartDateTime en **History** informa del inicio de ciclo, pero no sustituye DateTime en WHERE y puede ser ignorado como criterio (pp. 23–24). Esto **no** impide su uso documentado como criterio en las vistas Summary.

## WideHistory y agregación (pp. 130–132, 147–148, 160–165)

En Delta multitag se devuelve una fila cuando cambia cualquiera de los tags. Los demás muestran su valor anterior; no son muestras físicas simultáneas de todos los tags. La calidad puede ser inválida y debe considerarse al usarla.

SUM/AVG **dentro** de WideHistory Delta agregan cambios de cada columna, no todos sus valores repetidos de las filas alineadas. Un AVG exterior sobre el rowset wide puede por tanto dar otro resultado. Ninguno representa necesariamente promedio temporal; usar Average/Integral para esa intención.

La columna wwResolution en salida Delta/Full muestra tiempo hasta el siguiente punto, con tratamiento especial del último punto y NULL; en WideHistory mide hasta el siguiente cambio de **cualquier** tag. No confundir este resultado con imponer wwResolution como opción de entrada (véase options-quality-time.md).

## Diagnóstico y coste (pp. 16, 21–23, 142–143, 167–170)

- Error 7320/«no WHERE» pese a tenerlo: SQL Server puede eliminar/reformular criterios. Añadir filtro significativo (tag, intervalo, modo), revisar plan y delegación; OPENQUERY puede aislar lo que recibe INSQL.
- Join complejo: reducir a una consulta histórica simple; comprobar lado derecho de INNER REMOTE JOIN y límites de tags/tiempo antes de recuperar metadatos.
- Resultado vacío: comprobar tipos soportados por modo, modo/opción, intervalos, calidad y existencia de datos. La guía dice que conflictos de columnas virtuales pueden devolver cero filas.
- No usar TOP como prueba de cobertura completa; Delta con cycle count también puede truncar globalmente el resultado. Full largo puede ser muy voluminoso. Acotar por tags e intervalo desde el proveedor; dividir con una política explícita de extremos.
- Procedimientos almacenados de SQL Server pueden encapsular cuatro partes, OPENQUERY, parámetros y cursores; no eliminan las restricciones del parser INSQL (p. 168). El cursor de snapshot por evento pp. 167–168 es un patrón legacy, no la primera opción para consultas masivas.
- En Classic Storage, huecos entre bloques generan marcadores de fin/reinicio; al consultar antes del primer bloque o en el futuro pueden devolverse NULL (pp. 169–170). No convertir esos huecos en ceros.

El ejemplo p. 165 hace DROP TABLE y SELECT INTO; pp. 166–167 insertan en ManualAnalogHistory/History; p. 149 escribe Annotation. Son escrituras separadas de retrieval. Describirlas solo si son relevantes y ejecutarlas únicamente con autorización para el destino concreto.
