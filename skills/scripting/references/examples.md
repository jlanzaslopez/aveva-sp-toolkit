# Ejemplos cotejados y patrones de adaptación

Fuente: *AVEVA Application Server Scripting Guide*, octubre de 2020, páginas internas citadas. **Verificación documental**: llamadas, orden de parámetros, contexto y reglas cotejados con el PDF. Ningún fragmento se ha compilado o ejecutado en SP2023. «Adaptado» identifica código reorganizado o corregido para esta síntesis; no es una transcripción certificada por AVEVA.

Contenido: calidad/cálculo; arrays/control; binding; referencias I/O; gráficos; .NET/archivos/SQL; cobertura de todos los ejemplos del capítulo 3; erratas y comprobaciones.

## Calidad y cálculo en Execute

Adaptado de pp. 58, 67–69, 108, 112–113. Objeto: atributos Double `Me.Input1`, `Me.Input2`, `Me.Average`. Execute Periodic con período del proceso, Quality según configuración. No se introduce un DIM para el dato, para conservar la calidad propagada.

```text
IF IsGood(Me.Input1, Me.Input2) THEN
    Me.Average = (Me.Input1 + Me.Input2) / 2;
ELSE
    SetBad(Me.Average);
ENDIF;
```

El resultado se actualiza solo con ambos Good; si cualquiera no es Good, se marca Bad el resultado local. Es una política conservadora **de esta adaptación**, no una obligación del manual. Si el requisito permite Uncertain, decidir explícitamente si se usa IsUsable; no cambiarlo sin revisar el uso posterior.

Comprobaciones matemáticas de referencia, sin atributos de planta (pp. 61, 63, 65, 108):

```text
DIM rounded AS Integer;
rounded = 32 / 60;
LogMessage(rounded);
LogMessage(Int(-4.7));
LogMessage(Trunc(-4.7));
LogMessage(Round(-4.5, 1));
```

Resultados documentados esperados: 1, −5, −4, −4. Ejecutar en IDE/runtime de prueba para verificar la conversión y formato de logs del host.

## Array y control

Adaptación de pp. 113–114, 120. Cuerpo de script: tres elementos String y FOR indexado, para no depender de la contradicción FOR EACH/OLE.

```text
DIM words[3] AS String;
DIM i AS Integer;
words[1] = "uno";
words[2] = "dos";
words[3] = "tres";
FOR i = 1 TO 3 STEP 1;
    LogMessage(words[i]);
NEXT;
```

Debe emitir tres entradas en ese orden. Un array bidimensional se declara `DIM grid[2,3] AS Integer;` y su último elemento es `grid[2,3]`, como p. 120. No usar `[0]` ni ReDim. Los arrays DIM difieren de indexadores .NET como SqlDataReader.GetString(0).

## Binding off-engine por scans

Adaptado de pp. 123–126. Objeto; atributo de salida Double `Me.RemoteValue`; referencia real remota sustituye `RemoteObject.PV`. Requiere que el destino permita lectura y no dependa del patrón ActiveOnDemand desaconsejado. El ejemplo no escribe el destino remoto.

**Declarations**:

```text
DIM remote AS Indirect;
DIM bindState AS Integer;
DIM waitScans AS Integer;
```

**OnScan**, solo inicialización local:

```text
bindState = 0;
waitScans = 0;
SetBad(Me.RemoteValue);
```

**Execute**, Periodic, período 0, con evaluación cada scan:

```text
IF bindState == 0 THEN
    remote.BindTo("RemoteObject.PV");
    bindState = 1;
ELSEIF bindState == 1 THEN
    IF IsGood(remote) THEN
        bindState = 2;
    ELSE
        waitScans = waitScans + 1;
        IF waitScans >= 100 THEN
            bindState = -1;
            SetBad(Me.RemoteValue);
            LogError("La referencia remota no ha alcanzado calidad Good");
        ENDIF;
    ENDIF;
ELSEIF bindState == 2 THEN
    IF IsGood(remote) THEN
        Me.RemoteValue = remote;
    ELSE
        SetBad(Me.RemoteValue);
        waitScans = 0;
        bindState = 1;
    ENDIF;
ENDIF;
```

La salida se marca Bad al entrar OnScan y la cadena IF/ELSEIF garantiza no leer en el scan que enlaza. El estado 2 empieza a consumir en un scan posterior al que detecta Good. El límite de **100 scans** es política ilustrativa añadida, no 100 segundos ni garantía AVEVA; ajustarlo al requisito. −1 no reintenta automáticamente; volver OnScan reinicia el intento. Antes de generar una solución real acordar timeout, señal de estado y recuperación requeridos. No tomar `WriteStatus(remote)==MxStatusOk` como prueba de binding, ni hacer espera activa WHILE dentro del scan.

## Configuración dinámica InputSource/OutputDest

Patrón reducido de pp. 14–16. Script de objeto llamado **Set_Refs**, Execute WhileTrue, expresión `NOT Me.Ref_Done`, período 0. Requiere Boolean `Ref_Done` inicialmente False y atributo I/O `IOItem1` con InputSource/OutputDest. Los nombres de DIObject/topic/item deben existir.

```text
IF Me.Set_Refs.ExecutionCnt == 2 THEN
    Me.IOItem1.InputSource = "DIObject.Topic." + Me.Tagname + ".Item1";
    Me.IOItem1.OutputDest = "DIObject.Topic." + Me.Tagname + ".Item1";
    Me.Ref_Done = True;
ENDIF;
```

El original configura dos I/O items. Esta adaptación conserva uno para mostrar el mecanismo. El manual dice que estas referencias se checkpointan y no deben reconstruirse tras cada failover. Si cambian los destinos, definir cómo volver a solicitar configuración; no añadir reset de Ref_Done en cada arranque. La condición ExecutionCnt==2 no verifica por sí sola calidad/comunicación remota.

## Popup Industrial Graphics y contenido OMI

Popup desde símbolo cliente, adaptado de pp. 38–39. Requiere símbolo `ValveSymbol` con CP String/numérica `CP1` compatible con valor 20; identidad `ValvePopup1` única para esta instancia.

```text
DIM graphicInfo AS aaGraphic.GraphicInfo;
DIM cpValues[1] AS aaGraphic.CustomPropertyValuePair;
cpValues[1] = new aaGraphic.CustomPropertyValuePair("CP1", 20, true);
graphicInfo.Identity = "ValvePopup1";
graphicInfo.GraphicName = "ValveSymbol";
graphicInfo.CustomProperties = cpValues;
ShowGraphic(graphicInfo);
```

En otra acción del mismo cliente: `HideGraphic("ValvePopup1");`. Desde el símbolo popup: `HideSelf();`. Para CP por referencia, el tercer argumento es False y el segundo una referencia String real. No emplear la variante de cuatro argumentos del ejemplo defectuoso de p. 38.

Contenido de pane **solo OMI**, adaptado de pp. 26–29; acción de símbolo o script de layout, nunca objeto de servidor. Requiere Content `ValveSymbol` y pane configurado `DetailPane` en el layout invocante.

```text
DIM contentInfo AS aaContent.ContentInfo;
contentInfo.Content = "ValveSymbol";
contentInfo.Name = "ValveDetail1";
contentInfo.PaneName = "DetailPane";
contentInfo.SearchScope = aaContent.SearchScope.Self;
ShowContent(contentInfo);
```

Para ocultar una instancia identificada, otra acción:

```text
DIM contentInfo AS aaContent.ContentInfo;
contentInfo.Name = "ValveDetail1";
HideContent(contentInfo);
```

Self busca en el layout invocante, también si es anidado. No sustituirlo por AllScreens esperando búsqueda recursiva. Para Object Wizard usar el nombre absoluto de símbolo con propietario.

## .NET: Hashtable, formato y XML

Adaptado de pp. 118–121; no requiere servicios públicos. Cuerpo de script con tipos .NET disponibles en la instalación:

```text
DIM lookup AS System.Collections.Hashtable;
DIM currentTime AS Time;
DIM doc AS System.Xml.XmlDocument;
DIM root AS System.Xml.XmlElement;
lookup = new System.Collections.Hashtable;
lookup["Valve1"] = 20;
LogMessage(lookup["Valve1"]);
currentTime = Now();
LogMessage(currentTime.ToString("hh:mm:ss"));
doc = new System.Xml.XmlDocument;
root = doc.CreateElement("catalog");
doc.AppendChild(root);
root.SetAttribute("asset", "Valve1");
```

Resultado de lookup esperado 20; formato de tiempo de 12 h (`hh`) tal como el ejemplo. Para persistir XML, el original usa `doc.Save(ruta);`; para cargar `doc.Load(ruta);` y consultar `SelectSingleNode(xpath)`/`SelectNodes(xpath)`. Comprobar nodo distinto de null antes de `.InnerText` en adaptaciones. Archivo/ruta/permisos son requisitos independientes; no poner el acceso en Startup/OnScan (p. 9).

## Archivos y excepciones

Adaptación de pp. 114–115, 121, 123. Execute autorizado para lectura; ruta de ejemplo sustituible, archivo existente y permisos de cuenta runtime. No son las funciones FileReadMessage de InTouch.

```text
DIM reader AS System.IO.StreamReader;
reader = null;
TRY
    reader = System.IO.File.OpenText("C:\Data\sample.txt");
    WHILE reader.Peek() > -1
        LogMessage(reader.ReadLine());
    ENDWHILE;
CATCH
    LogError(ERROR.Message);
ENDTRY;
IF reader <> null THEN
    TRY
        reader.Close();
    CATCH
        LogError(ERROR.Message);
    ENDTRY;
ENDIF;
```

La limpieza después de CATCH es una adaptación porque el manual no documenta FINALLY. Lee todo el archivo dentro de una ejecución: usar solo con archivo pequeño acotado; para trabajo largo elegir ejecución asíncrona y comprobar solicitud de shutdown según host. El ejemplo de escritura original usa `System.IO.File.CreateText(ruta)`/`WriteLine`/`Close` y **sobrescribe** un archivo existente; no reutilizarlo para append ni ejecutar sin autorización sobre ese destino.

## SQL parametrizado: núcleo documental

Patrón de pp. 119–121, con nombres del esquema Northwind del ejemplo. Es **núcleo de operación**, no script completo desplegable: presupone `connection` abierta de tipo `System.Data.SqlClient.SqlConnection`, autenticación/configuración revisadas, permiso INSERT y esquema/clave sin conflicto. No incorporar `uid=sa` sin contraseña del original como configuración recomendada.

```text
DIM command AS System.Data.SqlClient.SqlCommand;
DIM regionId AS System.Data.SqlClient.SqlParameter;
DIM regionDesc AS System.Data.SqlClient.SqlParameter;
DIM commandText AS String;
commandText = "INSERT INTO Region (RegionID, RegionDescription) VALUES (@id, @desc)";
command = new System.Data.SqlClient.SqlCommand(commandText, connection);
regionId = command.Parameters.Add("@id", System.Data.SqlDbType.Int, 4);
regionDesc = command.Parameters.Add("@desc", System.Data.SqlDbType.NChar, 50);
command.Prepare();
regionId.Value = 5;
regionDesc.Value = "Europe";
command.ExecuteNonQuery();
```

No ejecutarlo periódicamente sin un trigger de petición y control de duplicados: la clave 5 puede existir. La envoltura real debe gestionar TRY/CATCH y cierre de reader/conexión tanto en éxito como en error, conforme al patrón anterior. Para SELECT el manual usa SqlCommand/ExecuteReader, `WHILE reader.Read()` y cierre reader→connection. No convertir concatenación de datos de usuario en SQL; el patrón documentado de parámetros evita esa necesidad. Estas son APIs .NET, no funciones QuickScript integradas.

## Mapa completo del capítulo 3 (pp. 117–126)

Se sintetizan todos los casos sin redistribuir sus listados extensos ni presentar dependencias antiguas como vigentes.

| Ejemplo del manual | Patrón conservado y limitación | Páginas |
|---|---|---|
| Excel con librería importada | Importar Microsoft.Office.Interop.Excel.dll mediante Script Function Library; Workbooks.Open/ActiveSheet/Range/Close. Depende de Excel instalado; no lanzar GUI desde servidor; el ejemplo no guarda cambios | 117 |
| Excel con CreateObject | `CreateObject("Excel.Application")`, Workbooks.Add, Range.Value y Close(false); corregir puntos extra de `Range.(...)`; requiere COM/Office y revisión de cierre de proceso | 117–118 |
| Web service de temperatura | Generar wrapper WSDL, compilar DLL, importar, instanciar WeatherRetriever y GetTemperature(zipcode). Son tipos/métodos del servicio de ejemplo, no APIs integradas; URL histórica sin verificar | 118 |
| Web service para email | Wrapper importado SendMail.Message/SendSimpleMail, servicio histórico; no suponer disponibilidad ni enviar correo como prueba | 118 |
| Lookup | Hashtable no genérico, indexador por clave | 118 |
| Crear XML | XmlDocument/CreateElement/AppendChild/SetAttribute/InnerText/Save; permisos/ruta y errores | 119 |
| INSERT parametrizado | SqlConnection/SqlCommand/SqlParameter, tipos Int/NChar y Prepare/ExecuteNonQuery; evitar credenciales del ejemplo y duplicados | 119–120 |
| Array String | DIM base 1; original FOR EACH contradice restricción formal de p. 114; alternativa indexada arriba | 120 |
| Array Integer bidimensional | DIM x[2,3], índices 1..2 y 1..3; misma advertencia FOR EACH | 120 |
| Formato de número | Integer.ToString con imagen `"$#,###,###.00"`; formato/localización no implica cálculo monetario | 120 |
| Formato de tiempo | Time=Now(), ToString("hh:mm:ss"); `hh` de 12 h | 120 |
| Directorios | DirectoryInfo/GetDirectories/FullName; corregir construcción de instancia del listado y validar FOR EACH .NET, no afirmar soporte genérico a partir de él | 120 |
| Cargar/consultar XML | Load/SelectSingleNode y SelectNodes con XPath; gestionar nodo ausente y archivo inválido | 121 |
| Consultar SQL | ExecuteReader/Read/indexador CompanyName/Close; revisar tipos, nulos y cierre en error | 121 |
| Performance counter | Declaración persistente PerformanceCounter, inicializar CategoryName="Processor", CounterName="% Processor Time", InstanceName="0"; Execute NextValue. Nombres/permisos dependen del sistema | 121 |
| Leer texto | File.OpenText, Peek, ReadLine, Close; ejemplo adaptado arriba | 121 |
| Compartir objeto .NET/SQL | AppDomain.CurrentDomain.SetData/GetData, quitar con Null, sincronizar Monitor.Enter/Exit. Solo ámbito AppDomain, no intercambio entre engines; liberar lock incluso con error. El SQL Open en Startup contradice p. 9: no adoptarlo como práctica nueva | 122 |
| DDE con Excel | WWPoke/WWRequest/WWExecute, comillas dobles duplicadas. Legado; restricciones síncronas pp. 102–104, disponibilidad DDE/Excel no garantizada | 122 |
| Exchange/MAPI | CreateObject("MAPI.Session"), Logon/Outbox/Recipients/Send/Logoff. Manual excluye Office ≥2010 y solo soporta Office 32 bits; Office 2007 necesita CDO 1.2.1. No recomendar como solución SP2023 sin requisitos heredados explícitos | 122 |
| Screen scraping meteorología | WebRequest, StreamReader, Regex/Match/Groups, HttpUtility.UrlEncode. Depende de URL, HTML, librerías y cierre de streams; no usar URL/regex histórica sin verificar | 123 |
| SMTP | System.Web.Mail.SmtpMail.Send del entorno histórico; no es función integrada QuickScript ni garantía de soporte/autenticación actual | 123 |
| Escribir texto | File.CreateText/WriteLine/Close; riesgo de sobrescribir; permisos y limpieza | 123 |
| Indirect lectura/escritura | BindTo(string), leer variable/escribir variable, WriteStatus; off-engine requiere espera y calidad | 123–124 |
| Binding off-engine | Estado en Declarations, BindTo en Execute, comprobar Good en scans posteriores; variante WhileTrue es trigger, no bucle de espera | 124–126 |

## Erratas de ejemplos que requieren atención

- P. 69: LogCustom sin paréntesis de cierre. P. 72: ejemplo LogWarning sin `;`. Corregir puntuación siguiendo firmas.
- P. 30: ejemplo HMI mezcla graphicInfo/graphicInfo0. Declarar y usar el mismo nombre.
- P. 38: una construcción CustomPropertyValuePair tiene cuatro argumentos; las construcciones de tres argumentos pp. 38–39 son el patrón usado aquí.
- P. 100: el ejemplo dice que genera «One=1 Two=2» pero concatena signos `+`; no usar su salida textual como prueba del código.
- P. 113: `Me.AttrA2` aparece en el ejemplo cuando se discutía Attr2. La comparación `.value` o `== TRUE` no sustituye una comprobación explícita Good.
- P. 118: `ws.Range.(...)` contiene un punto extra. P. 120: construcción DirectoryInfo sin NEW en el listado, que debe verificarse/corregirse en una adaptación.
- P. 125: `System DateTime` carece de punto; nombres SimValue01/SimValue02 y SimValue1/SimValue2 varían; sustituir por atributos reales. El ejemplo evalúa estados con IF consecutivos y podría revisar calidad en el mismo scan: esta síntesis usa ELSEIF para imponer separación.
- P. 126: comentario permite BindTo en Startup, contrario a la recomendación explícita pp. 9/14. Usar Execute.

## Comprobación en instalación SP2023

Para convertir un fragmento cotejado en probado: validar script y guardar objeto/símbolo, comprobar dependencias y nombres en ese host; ejecutar en entorno de prueba con datos conocidos; revisar Logger, calidad y efecto final. Para binding probar engine remoto OffScan y recuperación; para recursos provocar archivo ausente o error SQL y confirmar cierre; para funciones cliente comprobar instancia/layout y cancelación de diálogos. Registrar versión/hotfix, contexto, trigger y resultado real. La verificación documental no reemplaza esas pruebas.
