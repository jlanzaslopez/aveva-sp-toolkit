# Elegir el modo de recuperación

Fuente: *AVEVA Historian Retrieval Guide*, diciembre 2020, pp. 27–90 y 125–127. Páginas internas. Todas las opciones se escriben como `AND wwRetrievalMode = 'nombre'` salvo SLR, que es un filtro. No asumir valores por defecto entre interfaces o tablas.

## Selección por resultado

| Necesidad | Modo documentado | Semántica y tipo | Páginas |
|---|---|---|---|
| Muestreo regular rápido | `Cyclic` | Último valor antes/en frontera, no interpolación lineal. Todos los tipos. Puede perder picos/huecos entre muestras | 27–30 |
| Cambios de valor | `Delta` | Suprime duplicados; devuelve cambios y, si se incluye inicio, valor inicial. Todos los tipos | 30–35 |
| Todos los puntos almacenados | `Full` | Conserva valores/calidades repetidos y timestamps; puede añadir inicial. No recupera lo eliminado por filtros de **almacenamiento** | 35–37 |
| Curva muestreada interpolada | `Interpolated` | Lineal o StairStep según configuración; interpolación solo analógicos, otros como Cyclic | 37–41 |
| Tendencia conservando extremos | `BestFit` | Por ciclo: primero, último, mínimo, máximo y primera excepción no-Good, sin duplicar un punto que cumple varios papeles. Analógicos/analog summaries; otros Delta | 41–46 |
| Promedio temporal | `Average` (`Avg` usado en ejemplos) | Ponderado por duración, analog/analog summaries; otros Cyclic | 46–50, 171–173 |
| Extremo real por ciclo | `Minimum`/`Min`, `Maximum`/`Max` | Mínimo/máximo con timestamp real; empate →primero. Apartados dicen analog; AUTO de analog summaries también documentado. Otros tipos Delta | 50–59, 126 |
| Cantidad de un caudal | `Integral` | Área con interpolación y divisor de unidad; analog/analog summaries. Otros tipos Cyclic según apartado general, revisar excepción summary abajo | 59–62, 127 |
| Velocidad de cambio | `Slope` | (V2−V1)/(T2−T1 en segundos); analog; otros Delta. AUTO de analog summary usa Last | 62–65, 127 |
| Incremento de un totalizador | `Counter` | Diferencia por ciclo con resets/rollover; analog y discrete, no String/summary | 65–69, 127 |
| Tiempo por estado | `ValueState` | Integer, Discrete, String, state summary; no real analog/analog summary. Una fila por estado/ciclo, NULL es estado | 70–75 |
| Tiempo entre comienzos consecutivos de estado | `RoundTrip` | Integer analog, Discrete, String; no real/analog summary/state summary. History y StateWideHistory | 75–78 |
| Punto almacenado a un lado del instante | `StartBound`, `EndBound` | StartBound: primero al buscar hacia atrás, en/antes del instante; EndBound: primero posterior. Sin instante usa ahora | 87–89 |
| Predicción de tendencia | **`wwFilter = 'SLR()'`** | Regresión lineal simple; no inventar `wwRetrievalMode='Predictive'`. No es medición almacenada | 86–87 |

Defaults de SQL: Delta en History y tablas Discrete/String; Cyclic en tablas Analog y resúmenes (pp. 28, 30). Especificar modo de forma explícita para que la consulta sea interpretable.

## Puntos de frontera y NULL

### Cyclic, Delta y Full (pp. 28–37)

Cyclic devuelve valores de frontera aunque no exista punto físico a esa hora. Delta/Full con inicio inclusivo pueden devolver el último valor previo estampado en el inicio, con `Quality=133`; si no hay valor previo/inicial, NULL con QualityDetail=65536, OPCQuality=0 y Quality=1. Con `>` no generar esa fila inicial. Delta no crea un valor final especial; Full sigue las reglas de inicio de Delta.

Delta devuelve el primer NULL después de un valor no-NULL, suprime NULL consecutivos y devuelve el primer no-NULL tras un hueco aunque coincida con el anterior al hueco. Full conserva repeticiones. El supuesto «Delta solo devuelve filas físicas» del texto introductorio debe leerse con su excepción inicial explícita.

### Interpolated y BestFit (pp. 37–46)

Lineal interpola entre anterior y posterior; si el anterior es NULL, se devuelve NULL. Si el anterior es válido pero el siguiente NULL, mantiene el anterior StairStep. Los límites pueden estar interpolados: no llamarlos puntos físicos.

BestFit es reducción para tendencia, no promedio ni lista completa de eventos. Puede devolver varios/ningún punto por ciclo; los valores no-NULL de ciclo parcial/con huecos reciben OR de 4096 en QualityDetail. Fuera de los puntos de frontera devuelve puntos delta seleccionados. Su número de filas no es igual a wwCycleCount.

### Average e Integral (pp. 46–50, 59–62, 111–118)

Average integra el área según interpolación y divide por el **tiempo válido**, no necesariamente duración total del ciclo. Linear pondera el promedio entre puntos; StairStep conserva el valor anterior. Los huecos no contribuyen con Good/Extended. PercentGood/TimeGood son necesarios para no confundir un promedio de poca cobertura con uno completo.

Integral calcula área y aplica EngineeringUnit.IntegralDivisor para convertir la base temporal a cantidad. Ejemplo: 3.5 L/s durante 60 s →210 L. Confirmar unidad/divisor configurados antes de prometer m³, litros, energía o distancia. `wwExpression='UOM([Speed],cm)'` transforma la unidad de resultado según ejemplo; exige licencia de conversión, sin la cual puede devolverse unidad original (pp. 61–62).

OPTIMISTIC puede rellenar/escalar sobre huecos con último valor válido; no equivale a datos completos. No calcular volumen mensual multiplicando un Average parcial por todas las horas del mes. Preferir integral con política explícita de cobertura; devolver valor y cobertura y decidir si el informe lo acepta.

Con timestamp End, la fila al inicio puede agregar el ciclo **previo**; con Start, la fila en el fin puede agregar un ciclo **posterior**. Véanse reglas exactas en options-quality-time.md. No sumar filas sin identificar sus períodos.

### Minimum/Maximum (pp. 50–59)

El apartado general exige `<=` para el fin. El inicial es el extremo del ciclo fantasma anterior, reestampado en el inicio; no se modifica su calidad por ese reestampado. Si un extremo del primer ciclo cae también en el inicio, pueden aparecer dos filas en ese timestamp.

Devuelve además el primer NULL del ciclo; no confundir «NULL aparece» con «todas las filas del ciclo son NULL». Extremos de ciclos con huecos/parciales añaden flag 4096. Un punto exactamente en fin puede devolverse con flag parcial. La p. 96 muestra una consulta Min con `<` pese a la exigencia general: contradicción que debe indicarse si cambia el resultado, no deducir una regla universal a partir de ella.

### Slope (pp. 62–65)

Se calcula por segundo con un punto y su anterior. Produce inicial y final según puntos que rodean el instante. Tras un NULL, pendiente del primer punto válido=0; NULL consecutivos se suprimen. Dos puntos en el mismo instante pueden dar NULL con QualityDetail=17 (pendiente infinita). No interpretar el cero después de un hueco como ausencia medida de cambio.

## Counter: configurar antes de interpretar (pp. 65–69)

- Un descenso se interpreta como rollover según configuración. El rollover es el **siguiente** valor después del máximo representable (0..9999 →10000), no siempre MaxEU.
- Para resets manuales, el manual prescribe rollover=0; ejemplo 123→0→3 devuelve 3, mientras rollover=200 devuelve 80.
- Counter deadband es porcentaje del rango, afecta clasificación reset/rollover y reversals; solo una inversión detectada antes del rollover. No inventar nombre de columna/configuración que esta guía no da.
- Tags CTU signed que saltan de máximo positivo a mínimo negativo necesitan revisar tipo unsigned/rango para ajustarse al modelo. No cambiar tipo o rango sin autorización.
- Valores negativos o ≥rollover positivo pueden devolver 0 con qdIO_FILTEREDPOINT; 0 no implica producción cero. Un rollover puede marcar QualityDetail=212.
- Ciclos que empiezan/terminan en hueco pueden devolver NULL/65536; un hueco interior puede ocultar varios rollovers y hacer impreciso el resultado. Revisar PercentGood incluso con resultado numérico.

## ValueState y RoundTrip (pp. 70–78, 119–120)

ValueState Total/Percent sirve para tiempo operativo; Minimum/Maximum/Average de episodios pueden sesgarse porque los episodios se cortan en frontera. Variantes `...Contained` excluyen episodios incompletos y contestan otra pregunta. StateTime se devuelve en **milisegundos** para duraciones; Percent/PercentContained son porcentajes (la frase «except Percent» de p. 119 omite aclarar explícitamente el segundo, pero p. 120 lo define como porcentaje).

RoundTrip mide entre flancos de entrada sucesivos al mismo estado; no mide directamente tiempo en ese estado. Solo admite MinContained, MaxContained, AvgContained (default), TotalContained, PercentContained. Un punto en la frontera final ayuda a cerrar el episodio previo pero se asigna al siguiente ciclo. Sin roundtrip, StateTime puede ser NULL; no equivale a cero.

State summary: ValueState solo es preciso si las fronteras de consulta coinciden con los períodos de resumen; si no, puede devolver Uncertain y tiempos mayores que el ciclo. No extrapolar el detalle del tag original a partir de resúmenes de menor resolución.

## Analog summary tags y wwValueSelector (pp. 125–127)

Un summary contiene múltiples estadísticas por período. History necesita seleccionar un valor VTQ; AnalogSummaryHistory permite pedir varias estadísticas juntas.

AUTO: Cyclic/Delta/Full/Slope usan Last y su timestamp real; Minimum/Maximum usan extremo y timestamp real. BestFit considera First/Last/Min/Max; una cobertura <100% cuenta como hueco. Average/Integral usan columnas Average/Integral, siempre StairStep para resúmenes, prorrateando períodos parciales y TimeGood; fronteras desalineadas o summary incierto →calidad 64. ValueState/Counter/RoundTrip sobre analog summary no devuelven valores.

Selectors explícitos: AUTO; FIRST; LAST; MIN/MINIMUM; MAX/MAXIMUM; AVG/AVERAGE; INTEGRAL; STDDEV/STANDARDDEVIATION. First/Last/Min/Max retornan timestamp real; Average/Integral/StdDev inicio del período summary. Que se seleccione STDDEV no crea un nuevo modo de recuperación. Si hubo datos no-Good en el período, el OPCQuality del valor elegido puede ser Uncertain aunque el Last original fuera Good.
