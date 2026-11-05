# Opciones, tiempo y calidad

Fuente: *AVEVA Historian Retrieval Guide*, 3-dic-2020. Páginas internas. Contenido: nombres y unidades; matriz de aplicación; ciclos y fronteras; zona horaria; calidad; filtros; erratas.

## Parámetros SQL y alcance

Se asignan como columnas virtuales en WHERE, por ejemplo `AND wwResolution = 60000`. Son entradas de recuperación, no columnas de configuración que actualizar. Proveedor stateless: repetir en cada consulta. Valores ww insensibles a mayúsculas; una sola asignación por opción, sin IN/OR para ramas de extensiones (pp. 18, 24–25, 89–91).

| Parámetro | Valores/contrato | Páginas |
|---|---|---|
| `wwRetrievalMode` | Cyclic, Delta, Full, Interpolated, BestFit, Average/Avg, Minimum/Min, Maximum/Max, Integral, Slope, Counter, ValueState, RoundTrip, StartBound, EndBound; compatibilidad por tipo en retrieval-modes.md | 27–89 |
| `wwCycleCount` | Número de fronteras/ciclos según modo y extremos. En Delta, si positivo limita **primeras N filas del conjunto completo**, no N por tag; omitido Delta sin límite según p. 142 | 91–93, 142 |
| `wwResolution` | Intervalo **ms** en modos cíclicos; 1000=1 s, 3600000=1 h, 86400000=24 h. En salida Delta/Full también expresa duración hasta siguiente punto, no intervalo impuesto | 93–97, 160–165 |
| `wwTimeDeadband` | **ms**, Delta, todos los tipos; se reinicia con último valor retornado. Los ejemplos requieren superar el umbral: con muestra cada segundo, 5000 produce separación 6 s, 4900 produce 5 s | 97–101 |
| `wwValueDeadband` | **% del rango EU completo**, Delta analógico; base último valor retornado; cambio de calidad fuerza devolución. No es % del valor actual ni unidades absolutas | 101–104 |
| `wwVersion` | Original / Latest; default Latest; no confundir con versión del producto. Todos los modos según texto general | 104–105 |
| `wwInterpolationType` | Linear / StairStep; consulta prevalece sobre tag y tag sobre default sistema. Su salida por fila puede ser LINEAR/STAIRSTEP | 105–107 |
| `wwTimeStampRule` | Start / End; default parámetro TimeStampRule del servidor; no asumir que solo cambia una etiqueta | 107–111 |
| `wwTimeZone` | Nombre de `Runtime.dbo.TimeZone.TimeZone`, poblado desde registro Windows del **servidor**; default zona de Historian, no cliente | 111 |
| `wwQualityRule` | Good / Extended / Optimistic; override del parámetro de sistema QualityRule | 111–119 |
| `wwStateCalc` | ValueState: Minimum, Maximum, Average, Total, Percent y MinContained, MaxContained, AvgContained, TotalContained, PercentContained. Ejemplos admiten Min/Max. RoundTrip: solo los cinco Contained | 119–120 |
| `wwFilter` | NoFilter por defecto; SigmaLimit(...), ToDiscrete(...), SnapTo(...); SLR() para predicción. Un solo filtro; calidad se aplica antes del filtro | 86–87, 120–125 |
| `wwValueSelector` | AUTO/FIRST/LAST/MIN/MINIMUM/MAX/MAXIMUM/AVG/AVERAGE/INTEGRAL/STDDEV/STANDARDDEVIATION, para analog summary tags | 125–127 |
| `wwEdgeDetection` | None / Leading / Trailing / Both; detecta transiciones del criterio de valor, no simplemente el signo de pendiente | 78–86 |
| `wwExpression` | Forma documentada `UOM(TagName, target unit)` dentro de string; ejemplos `UOM([Speed],cm)`, `UOM(Distance, m/s)`, `UOM(SysTimeSec,Minute)`; conversión licenciada, comprobar wwUnit devuelta | 61–62, 64–65, 69 |
| `SliceBy`, `SliceByValue` | En AnalogSummaryHistory: tag que define lotes al cambiar y filtro opcional por estado/lote | 134–138 |
| `wwOption` | Solo enumerado, sin contrato completo en esta guía: no inventar sus valores | 24 |
| `wwParameters`, `wwMaxStates` | Reservados; no utilizarlos como API implementada | 24, 90 |
| `wwRowCount` | Legacy/deprecated; usar wwCycleCount según modo | 24, 90 |

No extrapolar una opción SQL `ww...` a un parámetro REST del mismo nombre. Tampoco inventar funciones SQL a partir de filtros escritos como strings.

## Matriz de opciones (pp. 89–90, revisada visualmente)

V=wwVersion; Q=wwQualityRule. La tabla original marca opciones ignoradas cuando no aplican. **Delta cycle count** se añade abajo por documentación específica p. 142 aunque la matriz no lo marque.

| Modo | CycleCount/Resolution | Deadbands | InterpolationType | TimeStampRule | StateCalc | Filtro analógico | ValueSelector | SLR |
|---|---|---|---|---|---|---|---|---|
| Cyclic | Sí | No | No | Sí | No | Sí | Sí | No |
| Delta | Resolution no; CycleCount límite p. 142 | Tiempo/valor | No | No | No | Sí | Sí | Sí |
| Full | No | No | No | No | No | Sí | Sí | Sí |
| Interpolated | Sí | No | Sí | Sí | No | Sí | Sí | No |
| BestFit | Sí | No | Sí | No | No | Sí | Sí | Sí |
| Average | Sí | No | Sí | Sí | No | Sí | Sí | No |
| Minimum/Maximum | Sí | No | No | No | No | Sí | Sí | No |
| Integral | Sí | No | Sí | Sí | No | Sí | Sí | No |
| Slope | No | No | No | No | No | Sí | Sí | No |
| Counter | Sí | No | No | Sí | No | Sí | No | No |
| ValueState/RoundTrip | Sí | No | No | Sí | Sí | Sí | No | No |
| StartBound/EndBound | Tabla sin marcas; no inferir opciones adicionales de las casillas vacías | | | | | | | |

La matriz marca V/Q para todos los modos anteriores salvo Bounding, mientras los textos de wwVersion/wwQualityRule afirman aplicación a todos. Para Bounding usar solo el contrato específico pp. 87–89 y comprobar diferencias de versión. «Filtro analógico Sí» no significa soportado para tags no analógicos o analog summary (p. 120).

## CycleCount, resolución y límites (pp. 91–97, 142–159)

Elegir **uno**. Conflicto de fuente: pp. 92/142 dicen que Resolution domina e ignora CycleCount; p. 143 dice que ambos juntos devuelven cero filas. No hacer depender una consulta de esa combinación.

Defaults documentados para Cyclic: ninguno →100; CycleCount negativo →100; Resolution/Count ambos 0 o Count=0 sin resolución →100000 según pp. 92/142. No usar 0 como sinónimo portable de «sin límite». Delta: Resolution ignorada; CycleCount=0 limita a 100000 total; positivo limita N total; omitido devuelve todos los cambios del intervalo (p. 142).

En Cyclic con n puntos y duración D:

| Extremos | Resolución calculada | Timestamps |
|---|---|---|
| `>= inicio`, `<= fin` | D/(n−1) | Incluye ambos |
| Solo uno inclusivo | D/n | Excluye el extremo abierto dejando un paso |
| Ambos abiertos | D/(n+1) | Deja un paso en cada extremo |

La fórmula general de p. 91 D/(n−1) no cubre todos los casos; pp. 154–156 los distinguen. No aplicarla con n=1 sin revisar modo y fronteras. History normalizado devuelve filas por tag en Cyclic; WideHistory devuelve filas alineadas totales. BestFit/ValueState pueden devolver varias/ninguna por ciclo; Delta cuenta filas totales y puede cortar tags posteriores.

Con Resolution explícita, la rejilla se ancla al inicio; `>` salta un intervalo. El fin solo aparece si cae en rejilla y se incluye; `<` lo excluye siempre (pp. 157–159). La explicación general de p. 143 dice que agregados pueden tener ciclo final parcial; no obliga a Cyclic a añadir un punto fuera de rejilla.

### Ciclos fantasma

Una frontera de timestamp no identifica por sí sola el período de cálculo. Con End, la fila al inicio puede ser el agregado del período previo del mismo tamaño; con Start, una fila en el fin puede agregar el período siguiente. Revisar **StartDateTime y DateTime** en History; StartDateTime no es criterio fiable en WHERE de History, pero sí sirve para inspeccionar resultados.

Patrones documentados que evitan incluir el ciclo ajeno en Average/Integral: Start con `>= inicio AND < fin`; o End con `> inicio AND <= fin` y resolución adecuada. No trasladarlos automáticamente a Minimum/Maximum/ValueState: revisar su contrato y verificar el período calculado. Para ValueState Total, el ejemplo p. 73 usa End con `(inicio, fin]` y count=1. Para Minimum/Maximum el requisito general es fin inclusivo y el punto inicial puede ser extremo anterior.

Además, p. 92 advierte para ValueState que excluir el inicio del ciclo impide devolver estados del primer ciclo, y excluir el final impide devolver estados del último. Conservar esta advertencia junto al ejemplo específico Total/End de p. 73: no generalizar sus extremos a múltiples ciclos/otros StateCalc sin comprobar qué ciclos y estados se devuelven.

StartDateTime conserva el mismo offset que DateTime incluso si cruzan un cambio DST (p. 96). Un día civil no siempre son 86400000 ms: para reporting local calcular los límites del día/mes en su zona y convertir de forma coherente. No llamar «día local» a una partición de 24 h UTC sin explicarlo.

### wwResolution como salida (pp. 160–165)

Delta/Full: duración ms hasta próximo punto, recortada al intervalo cuando existe punto siguiente; si ya no existe ningún punto posterior, puede ser NULL; punto exactamente en fin →0. Para duraciones >2147000000 ms (~25 días) devuelve −1 según manual. No sumar −1/NULL como tiempo válido ni sustituirlos por cero silenciosamente. En WideHistory es tiempo hasta cambio de cualquier columna. SummaryHistory no comparte este comportamiento Delta/Full.

## Zona horaria (pp. 11, 111, 144–145, 175–176)

Almacenamiento UTC, interfaz SQL normalmente hora local de Historian; precisión interna FILETIME 100 ns. SQL datetime puede redondear a ~3.3 ms frente a datetime2. No usar ese redondeo para corregir duplicados o fronteras.

`wwTimeZone` toma un nombre **Windows** de TimeZone; obtener los valores admitidos de esa tabla. Para UTC, confirmar que `UTC` existe antes de usarlo; no asumir nombres IANA como Europe/Madrid. Mantener límites en la zona escogida y especificar ambos. En Events, EventTimeUtc siempre UTC y wwTimeZone no lo cambia; EventTime sí refleja zona.

SQL Server evalúa GETDATE/DATEADD primero en su hora local; después INSQL interpreta los literales con wwTimeZone. Cambiar solo wwTimeZone puede convertir «últimos 30 min» en varias horas. Usar límites UTC ya calculados si se consulta con UTC, o límites locales consistentes; no aplicar dos conversiones. La guía no documenta exhaustivamente funciones modernas de conversión T-SQL: verificar fuera de ella si se necesitan.

## Tres codificaciones distintas de calidad (pp. 34, 46, 104–119, 168–170)

| Campo | Interpretación |
|---|---|
| Quality | Código legacy de recuperación: 0 normal, 133 inicial sintético; 1 aparece para ciertos NULL; 256 marca hueco en ejemplos Classic Storage. No es OPCQuality |
| OPCQuality | Calidad OPC; 192 Good y 64 Uncertain/Doubtful son ejemplos base, 0 Bad. Hay subestados y bits de proveedor |
| QualityDetail | Detalle/flags; 65536 ausencia de dato válido en ciertos cálculos, 4096 ciclo parcial (OR sobre calidad existente), 0x2000 valor alterado por filtro, 202 punto revisado Good, 212 rollover Counter, 17 pendiente infinita según ejemplos |

No filtrar universalmente `QualityDetail=192` ni `Quality=192`. Un punto bueno revisado o con flags legítimos tendría otro detalle. Para join con OPCQualityMap, usar **`(h.OPCQuality & 255) = o.OPCQuality`**, pues el byte superior puede ser específico de servidor (p. 168). El mapa describe códigos; no garantiza cobertura temporal.

Good: excluye Uncertain de cálculos; Bad representa hueco. Extended: incluye Good+Uncertain; Bad sigue siendo hueco. Optimistic: permite cálculos sobre información incompleta y último valor conocido, con particularidades de Counter/Integral. **No elimina el riesgo por devolver un número**. PercentGood<100 exige advertir cobertura; no rellenar huecos con cero. El texto p. 112 alterna «good quality» y luego «uncertain» para el resultado Optimistic: conservar la cautela y comprobar calidad efectiva.

La nota de p. 119 dice que Cyclic, Full, Delta, Maximum, Minimum y BestFit no combinan calidades y que las reglas no se aplican a esos modos, pese al alcance general/matriz de pp. 89–90/112. Distinguir selección de puntos, cálculo del agregado y combinación de su calidad; no prometer que `wwQualityRule='Good'` filtre universalmente todas las filas devueltas o convierta su calidad a Good. Inspeccionar las columnas efectivas y validar la política del modo/versión en destino.

Promedio temporal sobre huecos divide por tiempo válido; no por duración total (p. 48). Al combinar resúmenes, ponderar por el tiempo que realmente sustentó cada resultado, no AVG de promedios; la fórmula por duración de p. 138 presupone cobertura apropiada. Los resúmenes desalineados pueden prorratear y devolver Uncertain aunque su valor sea numérico.

## Contratos exactos de filtros (pp. 78–87, 120–125)

Son strings asignados a wwFilter, no llamadas SQL independientes:

| Filtro | Firma y default | Restricción |
|---|---|---|
| SigmaLimit | `SigmaLimit()` / `SigmaLimit(n)`; n Double, default 2.0 | Media/desviación ponderadas de todo intervalo incluidos phantom; excluye NULL; elimina fuera de media±nσ. Si primer valor se filtra, lo reemplaza con media |
| ToDiscrete | `ToDiscrete()` / `ToDiscrete(CutoffValue)` / `ToDiscrete(CutoffValue, Operator)`; cutoff Double 0.0, operador `>` por defecto | Operadores impresos: `>`, `>=`, `<=`; no añadir `<` por simetría. Argumentos vacíos intermedios no admitidos. Preserva NULL y calidad; marca cambio 0x2000 |
| SnapTo | `SnapTo([tolerance[,base_value_1[,base_value_2]…]])`; default tolerance=0.01, base=0.0; hasta 100 bases Double | Rangos inclusivos base±tolerancia; si se solapan gana primera base. Sintaxis `SnapTo(3,)`, `SnapTo(,0)` inválida. Desbordamiento de rango Double indefinido; flag 0x2000 |
| SLR | `SLR()` | Predicción por regresión lineal simple; SQL de ejemplo p. 87 no fija modo; matriz admite Delta/Full/BestFit. No presentar predicción como recuperación de medición real |

Filtros SigmaLimit/ToDiscrete/SnapTo solo analog, no otros tipos ni analog summary; History/WideHistory/StateWideHistory. Solo uno por consulta y wwQualityRule antes de wwFilter.

wwEdgeDetection se aplica a detectores de valor analog/discrete. None devuelve criterio cumplido; Leading primera fila que pasa de no cumplir a cumplir; Trailing primera que deja de cumplir; Both ambos. El flanco se refiere a verdad del criterio, no siempre ascenso/descenso físico: con `Value <= umbral`, Leading puede ser descenso. Trailing puede devolver valores que **no** cumplen el filtro de valor por ser la frontera de salida (pp. 78–86).

## Inconsistencias documentales que deben conservarse visibles

- p. 143 contradice pp. 92/142 sobre Resolution+CycleCount: elegir uno.
- p. 96 escribe 36000000 en prosa sobre una hora; el ejemplo usa 3600000, que es la conversión correcta.
- pp. 50/55/72 exigen fin `<=` para Min/Max/ValueState; ejemplos pp. 96/73 usan `<`. No convertir esos ejemplos en prueba de equivalencia.
- p. 57 llama MIN aggregate a un ejemplo Maximum; debe interpretarse por el modo `Maximum`, no por esa frase.
- pp. 40–41 y otros listados usan resoluciones impresas que no concuerdan con minutos mostrados. No fabricar resultados esperados copiando esos timestamps sin verificar unidades.
- p. 88 ejemplos Bounding omiten FROM y operador/sintaxis TagName válidos; los ejemplos adaptados de esta skill corrigen esas omisiones y lo indican.
