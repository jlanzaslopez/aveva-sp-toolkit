# Sintaxis QuickScript .NET

Fuente: *AVEVA Application Server Scripting Guide*, octubre 2020. Citas `p.`/`pp.` = páginas internas; objetivo SP2023 sin prueba de runtime. Contenido: sentencias; variables/arrays; operadores; calidad; control; .NET.

## Sentencias y literales (pp. 7, 108–111)

- Terminar sentencias con `;`. Asignar con `=`, comparar con `==` o `<>`. Solo una asignación por sentencia y un destino a su izquierda.
- Comentario de línea: apóstrofo `'`. Comentario multilínea: `{ ... }`. El manual también usa `//` en ejemplos, pero la definición formal de p. 7 especifica las dos formas anteriores.
- Strings entre comillas dobles; escapar una comilla duplicándola: `"Dijo ""hola"""`. Concatenar con `+`.
- Entero decimal de 32 bits: −2147483648 a 2147483647; sin ceros iniciales salvo `0`. Hexadecimal: `0x`/`0X`, signo opcional, hasta ocho dígitos hexadecimales.
- Reales con punto decimal y exponente `e`/`E`; pueden asignarse a Float/Real/Double, con error de validación por desbordamiento. Un literal que comienza por punto necesita al menos un dígito detrás.
- Si una referencia parece un literal exponencial, usar el calificador mostrado por el manual: `Attribute("5E3")` (p. 110).

## Declaraciones, ámbitos y arrays (pp. 108–110, 120)

Una variable por DIM, antes de su primer uso:

```text
DIM count AS Integer;
DIM label AS String;
DIM samples[3] AS Double;
DIM grid[2,3] AS Integer;
```

La gramática impresa de p. 108 muestra paréntesis para dimensiones; los ejemplos de p. 120 usan corchetes como arriba. No sustituir esta notación por declaraciones C#.

| Propiedad | Regla documental |
|---|---|
| DIM en cuerpo | Valor local a esa ejecución; no accesible por otros scripts |
| DIM en Declarations | Conserva el valor durante la vida del objeto asociado; no implica checkpoint ni retención tras descargarlo |
| Sin AS | Integer por defecto |
| Tipos enumerados | Boolean, Discrete, Integer, ElapsedTime, Float, Real, Double, String, Message, Time, Object |
| Otros tipos | Tipos .NET o de librerías importadas; `Indirect` aparece específicamente en pp. 123–125 |
| Nombre | Empieza por letra, continúa con letras/dígitos/`_`; hasta 255 caracteres Unicode; sin puntos; nombre y tipo sin distinguir mayúsculas |
| Colisión | La variable prevalece sobre atributos/alias/otros nombres; el alias ignorado genera advertencia |
| Array DIM | De 1 a 3 dimensiones, límite inferior 1; límite superior 1..2147483647, fijo; sin ReDim |
| Array completo | `localArray[] = tag.attr[];`; comparación `==` de arrays completos no soportada: comparar elementos |

No declarar `DIM a, b AS Integer;`. La comprobación de una referencia antes del DIM puede producirse al guardar el objeto, y no al pulsar Validate Script (p. 109).

La base 1 se refiere a arrays QuickScript declarados con DIM. Los índices de APIs .NET son los de esas APIs: el propio manual utiliza `reader.GetString(0)` (p. 115). No imponer base 1 a todo indexador .NET.

`null` representa ausencia de objeto. Comprobar antes de acceder a miembros; desreferenciarlo termina el script y registra error. Asignarlo permite la recolección cuando ya no queden referencias; no sustituye cerrar archivos/conexiones (p. 110).

## Operadores y precedencia (pp. 104–108)

De mayor a menor precedencia, preservando la tabla original:

| Nivel | Operadores |
|---|---|
| 1 | `( )` |
| 2 | `-` unario, `NOT`, `~` |
| 3 | `**` |
| 4 | `*`, `/`, `MOD` |
| 5 | `+`, `-` binario |
| 6 | `SHL`, `SHR` |
| 7 | `<`, `>`, `<=`, `>=` |
| 8 | `==`, `<>` |
| 9 | `&` |
| 10 | `^` |
| 11 | `|` |
| 12 | `=` |
| 13 | `AND` |
| 14 | `OR` |

- `~`: complemento a uno de entero de 32 bits; `&`, `^`, `|`: AND, XOR y OR bit a bit. `^` no es potencia.
- `SHL`/`SHR`: enteros de 32 bits, desplazamiento sin signo, relleno con ceros; se pierden bits salientes.
- `MOD`: resto de división de cantidades enteras.
- `**`: inválidos base cero con exponente negativo y base negativa con exponente fraccionario; el manual dice que operandos inválidos devuelven cero (pp. 106–107). No usar ese cero como validación del cálculo.
- `AND`, `OR`, `NOT`: booleanos/discretos; numéricos cero se convierten a False, otros a True. El manual no garantiza cortocircuito: para proteger un acceso a un objeto nulo, separar comprobaciones en IF anidados.
- Cálculo matemático interno en Double. Al asignar el resultado a Integer se redondea, no se trunca: `32/60` asignado a Integer da 1. Usar `Trunc()` o una operación explícita para el redondeo deseado (pp. 58, 108).
- Trigonometría QuickScript en grados; no intercambiar directamente con equivalentes .NET (pp. 59–64). `Round(Number, Precision)` usa un paso de precisión, no cantidad de decimales.

## Calidad en asignaciones y condiciones (pp. 67–69, 109–113)

- `me.Out = me.In;` y `me.Out.value = me.In.value;` copian valor **y calidad** entre atributos del mismo tipo.
- Una variable local en el lado derecho se considera Good para propagación. Pasar por DIM puede ocultar una calidad Bad del origen; no equivale a sanear una medida.
- Una comparación con calidad Bad puede impedir ejecutar el cuerpo del IF. Comprobar explícitamente con `IsGood`/`IsBad` según el requisito antes de usar el dato.
- `IsUsable` acepta Good **o Uncertain** y excluye NaN en Float/Double; si se requiere Good usar `IsGood`. Una calidad aceptable no prueba actualidad del timestamp.
- La p. 113 presenta `== TRUE` y `.value` como alternativas en la discusión de calidad, pero no establecen por sí solas que el dato sea Good. Para código nuevo usar la comprobación explícita documentada.

## Control (pp. 111–116)

```text
IF count == 0 THEN
    label = "cero";
ELSEIF count > 0 THEN
    label = "positivo";
ELSE
    label = "negativo";
ENDIF;

FOR count = 1 TO 3 STEP 1;
    samples[count] = count;
NEXT;

WHILE count > 0
    count = count - 1;
    IF count == 1 THEN
        EXIT WHILE;
    ENDIF;
ENDWHILE;
```

Estos fragmentos presuponen los DIM anteriores. IF acepta Boolean/Discrete y numéricos (cero False); String/Message, Time y ElapsedTime no pueden ser condición. Object se convierte en runtime y puede lanzar excepción (pp. 111–112).

`FOR variable = inicio TO fin [STEP incremento]; ... NEXT;` admite Integer/Float/Real/Double. Extremo inclusivo; STEP positivo o negativo y `EXIT FOR;`. La p. 113 establece por defecto +1 ascendente y −1 descendente; especificar STEP explícitamente cuando importe. Los bucles transcurren dentro de **una** ejecución, no liberan scans ni esperan enlaces remotos.

`FOR EACH variable IN colección ... NEXT;`, con variable declarada y `EXIT FOR;`, se describe como limitado a colecciones OLE Automation en p. 114. Sin embargo, p. 120 lo usa con arrays DIM y pp. 120–121 con colecciones .NET: contradicción documental. Para arrays preferir FOR indexado en código nuevo; comprobar en el IDE el caso FOR EACH requerido.

```text
TRY
    LogMessage(Now().ToString());
CATCH
    LogError(ERROR.Message);
ENDTRY;
```

`ERROR` es `System.Exception`, disponible dentro de CATCH; admite `Message` y `GetType()`. Al lanzar una excepción se omite el resto del TRY, se ejecuta CATCH y se continúa después. El manual no documenta `FINALLY`, `using`, catch tipados múltiples ni construcciones C#; no introducirlos por analogía (pp. 114–115). Cerrar recursos también en rutas de error.

## Interoperabilidad (pp. 19, 101, 110, 117–123)

`NEW` instancia tipos .NET; `CreateObject(ProgID)` crea COM. Importar librerías mediante Galaxy > Import > Script Function Library cuando sea necesario. La p. 19 indica soporte de librerías CLR 2.0.50727 pero no de nuevas características del lenguaje .NET 2.0, como genéricos; no extrapolar esto a una matriz de runtimes de SP2023.

Para pasar atributos a métodos de objetos de sistema, p. 110 exige variable intermediaria o conversión explícita adecuada. Los ejemplos muestran `.ToString(formato)`, `System.Collections.Hashtable` e indexadores; son APIs .NET, no nuevas funciones integradas de QuickScript.
