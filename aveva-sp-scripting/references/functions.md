# Funciones: firmas, retornos y restricciones

Fuente: *AVEVA Application Server Scripting Guide*, 2-oct-2020, pp. 19–104. Páginas internas = páginas del PDF. Síntesis para SP2023; no validación de cada API en 2023.

Contenido: cliente gráfico/OMI; InTouch; matemática; calidad/logging/escrituras; strings/tiempo; sistema/DDE; inconsistencias.

## Convenciones

Las llamadas conservan nombre y orden de argumentos. Se omiten espacios tipográficos y variables receptoras como `Result =` cuando no son parte de la firma. Los tipos se indican únicamente donde el manual los da; una llamada sin tipos no constituye una firma .NET tipada. `...` representa la lista variable documentada; `[result =]` denota retorno opcional, no texto que pegar al script. Los nombres `SecsSince1-1-70` son metavariables del manual, no expresiones de resta que deban copiarse como argumento. Los retornos no definidos se señalan con «no especificado», sin inferir `void`.

Ámbitos: **cliente gráfico** = símbolo Industrial Graphics, sujeto al host; **OMI** = cliente/layout OMI; **InTouch** = funciones de esa familia y sus excepciones; **objeto local** = destino del objeto que aloja el script; **general** = sin prohibición específica en la entrada, sujeto a [execution.md](execution.md). No usar una API cliente desde servidor por aparecer en el browser.

## Cliente gráfico y OMI (pp. 19–39)

| Firma/call documentada | Retorno y restricciones | Páginas |
|---|---|---|
| `Int GetCPQuality(String name)` | Entero OPC 0..255, 192 Good. Solo custom property del símbolo, no tag HMI; constante siempre Good. Cliente gráfico; soporte depende del HMI | 19 |
| `DateTime GetCPTimeStamp(String name)` | Timestamp de valor actual de custom property; constante: hora de creación de su valor. No tags HMI | 20 |
| `HideContent(contentInfo);` | Parámetro `aaContent.ContentInfo`, retorno no especificado. OMI; no objeto. Oculta todos los elementos coincidentes en un nivel de layout | 20–24 |
| `HideGraphic(string identity);` | Retorno no especificado. Cliente gráfico; cierra popup o ventana HMI identificada; usar identidad de ShowGraphic | 24 |
| `HideSelf();` | Retorno no especificado. Llamar desde el símbolo mostrado; cierra gráfico/layout del contexto | 24–25 |
| `LogOff();` | Sin parámetros, retorno no especificado. Acción gráfica de ViewApp, Left-Click/Key/Touch Down; ejemplo escribe `Logoff()` | 25 |
| `ShowContent(contentInfo);` | `aaContent.ContentInfo`, retorno no especificado. Solo OMI; símbolo cliente/layout; no objeto. Carga un pane, no popup | 25–30 |
| `ShowGraphic(graphicInfo);` | `aaGraphic.GraphicInfo`, retorno no especificado. Cliente gráfico/layout OMI; popup OMI/InTouch y extensiones para ventana HMI/layout | 30–39 |
| `ShowLoginDialog();` | Sin parámetros, retorno no especificado. Acción gráfica ViewApp, Left-Click/Key/Touch Down | 39 |

### aaContent.ContentInfo (pp. 20–29)

Declaración documentada: `DIM contentInfo AS aaContent.ContentInfo;`. No inventar constructor obligatorio donde el ejemplo no lo utiliza.

| Miembro | Tipo y semántica |
|---|---|
| `Content` | String; obligatorio en ShowContent, opcional en HideContent. Nombre en Graphic Toolbox/asset: símbolo, layout o contenido externo; admite referencia relativa |
| `Name` | String opcional; identidad de instancia, única dentro del layout; puede repetirse en otros layouts anidados. En HideContent prevalece sobre Content |
| `ScreenName` | String opcional; nombre de Screen Profile. Si se especifica, búsqueda en layout superior |
| `PaneName` | String opcional; pane del Layout Editor |
| `ContentType` | String opcional; sobreescribe tipo que se usaría para buscar pane por Content, p. ej. Overview/Navigation/Faceplate |
| `SearchScope` | Enum opcional; por defecto `aaContent.SearchScope.Self` |
| `PropertyOverrideValues` | Array de `aaContent.PropertyOverrideValue`, utilizado por el ejemplo p. 26; solo ShowContent. Tabla p. 29 lo llama «Property Overrides»; no crear otro miembro por esa etiqueta |
| `OwningObject` | String opcional; propietario para referencias relativas en ShowContent. No reemplaza nombre absoluto del símbolo de Object Wizard |

Valores de `aaContent.SearchScope`: `Self`, `AllScreens`, `SourceScreen`, `PrimaryScreen`. Self busca en el layout que hizo la llamada, incluido uno anidado; los demás buscan en layouts de nivel superior asociados al Screen Profile. AllScreens: origen, primario y restantes en orden alfabético. No confundir las abreviaturas de texto «All/Source/Primary» con nombres de enum.

Repetir ShowContent puede cerrar y reabrir contenido coincidente y actualizar overrides. Nombres Content duplicados no son comprobados por Application Server. Para seleccionar una instancia concreta usar Name. HideContent sin propiedades actúa como HideSelf sobre un layout anidado; no cierra layout superior. Aun sin propiedades hay que pasar `contentInfo`.

Override mostrado: `new aaContent.PropertyOverrideValue("CP1", "20", true)`. El tercer argumento indica constante; False trata el valor como referencia (ejemplo p. 26). Los arrays QuickScript empiezan en 1. Para Object Wizard y selección de Symbol Wizard, especificar nombre absoluto con propietario en Content; no copiar la mención inconsistente a GraphicName de la tabla como si ContentInfo tuviese ese miembro.

### aaGraphic.GraphicInfo (pp. 30–39)

Declaración: `DIM graphicInfo AS aaGraphic.GraphicInfo;`. Identity y GraphicName son obligatorios en el caso gráfico. Para ventana HMI el manual muestra Identity `"<HMIName>:Window1"` y `ShowGraphic(graphicInfo);` sin GraphicName.

| Miembro | Tipo; valor por defecto; límite/uso |
|---|---|
| `Identity` | String obligatorio; al menos una letra, hasta 329 caracteres; alfanuméricos y `$ # _` en caso gráfico; misma identidad para HideGraphic |
| `GraphicName` | String obligatorio en caso gráfico; hasta 329 caracteres y al menos una letra; acepta nombres Toolbox, instancia/hierarquía y relativos; validar en runtime |
| `OwningObject` | String; vacío; resuelve referencias, no sustituye nombre absoluto GraphicName en Object Wizard |
| `HasTitleBar` | Boolean; True |
| `WindowTitle` | String; vacío → Identity; hasta 1024 caracteres |
| `WindowType` | Enum; Modeless; `aaGraphic.WindowType.Modal=0`, `Modeless=1` |
| `HasCloseButton` | Boolean; True |
| `Resizable` | Boolean; False |
| `WindowLocation` | Enum; Center; valores detallados abajo |
| `WindowRelativePosition` | Enum; Desktop; valores abajo |
| `RelativeTo` | Enum; Graphic; valores abajo |
| `X`, `Y` | Integer; 0; −2147483648..2147483647; solo DesktopXY/WindowXY/ClientAreaXY; overflow registrado |
| `Width`, `Height` | Integer; 100; 0..10000; solo CustomizedWidthHeight; se limita valor fuera de rango y se registra; una dimensión permite calcular la otra por proporción |
| `TopMost` | Boolean; False; z-order superior. No abrir modal no-TopMost desde diálogo TopMost: puede quedar oculto y bloquear interacción |
| `ScalePercentage` | Integer; 100; 0..1000, limitado en runtime si se excede |
| `KeepOnMonitor` | Boolean; True; mantener popup dentro de límites de ventana de aplicación |
| `StretchGraphicToFitWindowSize` | Boolean; True; aplicable con ScalePercentage >100 |
| `StretchWindowToScreenWidth`, `StretchWindowToScreenHeight` | Boolean; False; solo Desktop/Window/ClientArea/ParentGraphic/ParentElement |
| `CustomProperties` | Array CustomPropertyValuePair; nombre, valor, IsConstant. True=constante, False=referencia; base 1 |

`aaGraphic.WindowLocation`: Center=0, Above=1, TopLeftCorner=2, Top=3, TopRightCorner=4, LeftOf=5, LeftSide=6, RightSide=7, RightOf=8, BottomLeftCorner=9, Bottom=10, BottomRightCorner=11, Below=12. Con posición relativa Desktop no son válidos Above/LeftOf/RightOf/Below.

`aaGraphic.WindowRelativePosition`: Desktop=0, Window=1, ClientArea=2, ParentGraphic=3, ParentElement=4, Mouse=5, DesktopXY=6, WindowXY=7, ClientAreaXY=8.

`aaGraphic.RelativeTo`: Graphic=0, DeskTop=1, CustomizedWidthHeight=2 (se conserva la grafía `DeskTop` del manual).

Constructor mostrado: `new aaGraphic.CustomPropertyValuePair("CP1", 20, true)`. No confundir este tipo/miembro con `aaContent.PropertyOverrideValue`/`PropertyOverrideValues`. Propiedades con default son opcionales; no inventar valores de retorno de ShowGraphic.

## Familia InTouch (pp. 40–58)

Regla común p. 40: utilizables en scripts de símbolos del Industrial Graphic Editor; **no funcionan en scripts de objetos ArchestrA aunque no aparezcan errores ni warnings**. Excepción explícita: LogonCurrentUser solo InTouch scripting, no ArchestrA client scripting. La conversión a Industrial Graphic puede anteponer erróneamente `InTouch:` al nombre de función: quitar ese prefijo de la función; no quitarlo indiscriminadamente de referencias de tags.

### Seguridad

| Call documentada | Retorno / parámetros / límite | Páginas |
|---|---|---|
| `AddPermission("Domain", "Group", AccessLevel);` | Boolean; asigna nivel a grupo existente para **logons posteriores**. Solo seguridad OS. No afecta usuarios conectados | 40 |
| `AttemptInvisibleLogon("UserId", "Password", "Domain");` | Boolean: True autenticado, actualiza `$OperatorDomain`, `$OperatorName`, `$AccessLevel`, `$Operator`; False conserva usuario actual. Domain solo OS; en ArchestrA sobre OS usar usuario cualificado | 40–41 |
| `[Result=]ChangePassword();` | Integer 0 Cancel, 1 OK; diálogo de contraseña | 41–42 |
| `GetAccountStatus(Domain, UserID);` | Días hasta expiración; −1 expiró, −2 nunca expira, −3 bloqueada, −4 deshabilitada, −5 fallo. Seguridad OS; no usar como función ArchestrA, aunque consulta directa al dominio pueda funcionar si coincide configuración OS del GR | 48 |
| `InvisibleVerifyCredentials("UserId", "Password", "Domain");` | Nivel de acceso Integer o −1. **Síncrona**; asíncrona siempre −1. No hace logon; Domain solo OS; usuario cualificado en ArchestrA sobre OS | 52 |
| `IsAssignedRole("RoleName");` | Boolean de rol del usuario actual; solo ArchestrA security | 52–53 |
| `LogonCurrentUser();` | `IntegerResult`; −1 fallo, sin cambiar tags de operador. Usa usuario Windows; requisitos OS/ArchestrA OS user/group; smart card soportada bajo condiciones descritas. Solo InTouch scripting | 53–54 |
| `PostLogonDialog();` | Boolean True; muestra diálogo InTouch | 54 |
| `QueryGroupMembership("Domain", "Group");` | Boolean de pertenencia del usuario actual; OS security o ArchestrA configurado sobre OS | 55–56 |

Las comillas de las firmas indican parámetros string, no credenciales reales que deban incluirse en repositorio. No registrar contraseñas.

### Archivos

| Call documentada | Resultado y contrato | Páginas |
|---|---|---|
| `FileCopy(sourcefile, destfile, progresstag)` | Inicio: 1 aceptado, 0 otro copy activo, −1 error. Tag Integer de progreso **por nombre**: 0 en curso, 1 completado, −1 error. Rutas string/UNC; origen admite `* ?`. Destino ausente puede esperar 10 s. No QuickFunctions asíncronas | 42–43 |
| `FileDelete(filename)` | 1 borrado, 0 fallo. Archivo individual/UNC, sin comodines ni directorios; no QuickFunctions asíncronas | 43–44 |
| `FileMove(sourcefile, destfile, progresstag)` | Igual distinción inicio/progreso 1/0/−1; origen admite comodines/UNC; sirve para renombrar. No QuickFunctions asíncronas | 44–45 |
| `[result =] FileReadFields(filename, offset, starttag, numberoffields)` | Nuevo offset en bytes opcional; CSV solo coma; **solo síncrona**. starttag string entre comillas con sufijo numérico, p. ej. `"RecipeTag1"`; incrementa sufijo para campos sucesivos | 45–46 |
| `[result =] FileReadMessage(filename, offset, messagetag, charstoread)` | Nuevo offset en bytes; charstoread=0 lee hasta LF. En Industrial Graphic Editor pasar nombre de tag entre comillas | 46 |
| `[result =] FileWriteFields(filename, offset, starttag, numberoffields)` | Nuevo offset en bytes; crea archivo si falta; offset=−1 append; CSV coma y tags de sufijo numérico correlativo | 46–47 |
| `[result =] FileWriteMessage(filename, offset, messagetag, linefeed)` | Nuevo offset en bytes; offset=−1 append; linefeed 1 añade LF, 0 no. No confundir longitud de caracteres Unicode con offset en bytes | 47–48 |

Inicio aceptado de FileCopy/FileMove no significa operación terminada. `progresstag` recibe nombre del tag, no el valor de un DIM. No sustituir estas APIs por funciones de archivo válidas en servidor sin cambiar explícitamente de familia/contexto.

### Sistema, ventanas y Terminal Services

| Call/comando documentado | Retorno / semántica | Páginas |
|---|---|---|
| `EnableDisableKeys(AltKey, EscKey, WinKey);` | Retorno no especificado; cada Integer: 1 habilita filtro/deshabilita tecla, 0 permite tecla. Filtrar Alt también deshabilita Win+L según manual; Esc afecta todas sus acciones | 42 |
| `GetNodeName(messagetag, nodenum);` | Escribe nombre en tag Message; nodenum 0..131 caracteres; en editor gráfico nombre de tag entre comillas | 48–49 |
| `InfoAppTitle(appname)` | Texto de título/task de aplicación en ejecución; appname sin `.exe` | 49 |
| `InfoDisk(drive, infotype, trigger);` | 1 tamaño bytes, 2 libre bytes, 3 tamaño KB, 4 libre KB; drive usa primera letra. Trigger recalcula en animation display link; en script puede ser literal numérico | 49–50 |
| `InfoFile(filename, infotype, trigger)` | 1 existencia: 0 ausente/1 archivo/2 directorio; 2 tamaño bytes; 3 creación en segundos desde 1970; 4 número de coincidencias. Rutas/comodines; trigger como InfoDisk | 50–51 |
| `InfoInTouchAppDir();` | String directorio aplicación actual; puede truncarse en tag Message/animation por límite 131 | 51 |
| `InTouchVersion(infotype);` | 0 versión completa, 1 major, 2 minor, 3 patch, 4 build; no deducir offsets a partir del ejemplo histórico | 51 |
| `LaunchTagViewer()` | Retorno no especificado. Requiere WindowViewer activo, Tag Viewer habilitado en WindowMaker y permisos. No application OnStartup/OnShutdown | 53 |
| `Playsound(soundname, flag)` | Retorno no especificado; WAV/nombre [Sounds] en Win.ini; flag 0 una vez síncrono, 1 una vez asíncrono, 9 continuo hasta otra llamada | 54 |
| `PrintScreen(ScreenOption, PrintOption)` | Retorno no especificado; ScreenOption 1 cliente sin menús, 2 ventana completa; PrintOption 1 best fit, 2 vertical, 3 horizontal, 4 stretch. Inválido/0 → best fit. Recorta popups fuera de área WindowViewer | 54–55 |
| `ShowHome();` | Abre Home Windows configuradas y cierra las demás; retorno no especificado | 56 |
| `StartApp appname;` | **Comando**, no `StartApp(...)`; ruta/nombre string, PATH permite omitir ruta. GUI desde servidor no soportada (p. 19) | 56 |
| `SwitchDisplayLanguage(LocaleID);` | Retorno no especificado; cambia textos estáticos/campos de alarma con traducciones disponibles; ejemplo 1031 alemán | 56–57 |
| `TseGetClientId();` | String ID/IP si cliente Terminal Server; si no, vacío | 57 |
| `TseGetClientNodeName();` | String nombre identificable Windows del cliente Terminal Server; si no, vacío | 57 |
| `TseQueryRunningOnClient();` | Integer no cero en cliente Terminal Services, cero en otro caso | 57 |
| `TseQueryRunningOnConsole();` | Integer no cero en consola Terminal Services, cero en otro caso | 57–58 |

## Matemática (pp. 58–65)

Ámbito general. Number = número o atributo numérico salvo restricción. Las operaciones se calculan en Double y se redondea al destino; `System.Math.Floor` es la alternativa .NET mostrada para cociente truncado hacia abajo (p. 58).

| Call exacta | Retorno / dominio / unidad | Páginas |
|---|---|---|
| `Abs(Number);` | Valor absoluto | 59 |
| `ArcCos(Number);` | Ángulo 0..180 **grados**; entrada −1..1 | 59 |
| `ArcSin(Number);` | Ángulo −90..90 grados; entrada −1..1 | 59 |
| `ArcTan(Number);` | Ángulo −90..90 grados | 60 |
| `Cos(Number);` | Coseno, entrada grados | 60 |
| `Exp(Number);` | e elevado a Number | 60–61 |
| `Int(Number);` | Integer menor o igual: Int(−4.7)=−5 | 61 |
| `Log(Number);` | Real logaritmo natural; cero indefinido; validar dominio positivo | 61 |
| `Log10(Number);` | Logaritmo base 10 | 61–62 |
| `LogN(Number, Base);` | Logaritmo base Base; Base entero/atributo Integer; base 1 indefinida. Validar dominio matemático | 62 |
| `Pi();` | Real, pi | 62 |
| `Round(Number, Precision);` | Real redondeado al paso Precision; ejemplos: (106,5)→105, (4.5,1)→5, (−4.5,1)→−4. No es número de decimales | 63 |
| `Sgn(Number);` | Integer: −1 negativo, 0 cero, 1 positivo | 63 |
| `Sin(Number);` | Seno, entrada grados | 63–64 |
| `Sqrt(Number);` | Real raíz cuadrada; validar entrada no negativa | 64 |
| `Tan(Number);` | Tangente, entrada grados | 64 |
| `Trunc(Number);` | Numérico sin parte fraccionaria hacia cero; Trunc(−4.3)=−4 | 65 |

Los controles de dominio positivo/no negativo son precauciones matemáticas para generar código; no afirmaciones de un código de error del runtime. La equivalencia textual de Trunc con asignar Float a Integer en p. 65 contradice la regla de redondeo pp. 58/108: utilizar Trunc explícito para truncar.

## Calidad, logging y otras funciones generales (pp. 65–77, 84–85)

| Call documentada | Retorno / contrato | Páginas |
|---|---|---|
| `ActivateApp(TaskName);` | Retorno no especificado; usa texto exacto del Task Bar/Task Manager para activar aplicación. GUI: contexto cliente compatible | 65 |
| `DateTimeGMT();` | Número de **días y fracciones de día** desde 1-ene-1970 UTC; multiplicar por 86400.0 para segundos | 66 |
| `IsBad(Attribute1, Attribute2, ...);` | Boolean True si **alguno** Bad | 67 |
| `IsGood(Attribute1, Attribute2, ...);` | Boolean True si **todos** Good | 67 |
| `IsInitializing(Attribute1, Attribute2, ...);` | Boolean True si **alguno** Initializing | 67–68 |
| `IsUncertain(Attribute1, Attribute2, ...);` | Boolean True si **todos** Uncertain, según Return Value impreso | 68 |
| `IsUsable(Attribute1, Attribute2, ...);` | Boolean True si todos Good o Uncertain; además Float/Double no NaN. No exige Good exclusivamente | 68–69 |
| `LogCustom(CustomFlag, msg);` | Retorno no especificado; strings; primer uso crea flag custom, deshabilitado por defecto; habilitar Log Custom en SMC | 18, 69 |
| `LogDataChangeEvent(AttributeName, Description, OldValue, NewValue, TimeStamp);` | Retorno no especificado; **solo objetos**. AttributeName como tag, Description texto, valores viejo/nuevo; timestamp opcional UTC/local, default Now(). Símbolo puede compilar, pero función inoperante con warning | 69–70 |
| `LogError(msg);` | Retorno no especificado; string; rojo, flag habilitado por defecto | 70–71 |
| `LogMessage(msg);` | Retorno no especificado; string; Log Viewer añade fecha/hora y objeto.script | 71 |
| `LogTrace(msg);` | Retorno no especificado; string; flag deshabilitado por defecto, habilitar Log Trace en SMC | 18, 71–72 |
| `LogWarning(msg);` | Retorno no especificado; string; amarillo, manual indica flag deshabilitado por defecto | 72 |
| `SendKeys(KeySequence);` | Retorno no especificado; string/atributo; teclado para aplicación. Limitado en algunos equipos por Windows HAL; GUI cliente | 72–74 |
| `SetAttributeVT(Attribute, Value, TimeStamp);` | Retorno no especificado; **objeto local**. Asigna valor y timestamp, calidad Good; timestamp variable/referencia/string local o UTC, conversión interna a UTC. Solo último cálculo buffered historizado | 74 |
| `SetAttributeVT2(Attribute, Value, TimeStamp);` | Mismo contrato/localidad; permite historizar valores buffered intermedios una vez por ciclo de scan | 74–75 |
| `SetBad(Attribute);` | Retorno no especificado; calidad Bad, atributo del mismo objeto | 75 |
| `SetGood(Attribute);` | Retorno no especificado; calidad Good, atributo del mismo objeto | 75–76 |
| `SetInitializing(Attribute);` | Retorno no especificado; calidad Initializing, atributo del mismo objeto | 76 |
| `SetUncertain(Attribute);` | Retorno no especificado; calidad Uncertain, atributo del mismo objeto | 76–77 |
| `WriteStatus(Attribute);` | Enum del último write; nunca escrito devuelve MxStatusOk; atributos sin calidad calculada siempre MxStatusOk. No prueba que un write solicitado haya tenido efecto | 84 |
| `WWControl(AppTitle, ControlType);` | Retorno no especificado; strings; ControlType `"Restore"`, `"Minimize"`, `"Maximize"`, `"Close"`; GUI cliente | 85 |

SetAttributeVT/VT2 no detectan en compilación si el atributo admite timestamp: puede no emitirse aviso. Conservar el orden de letras **VT** de ambas firmas.

WriteStatus enum (p. 84): `MxStatusOk`, `MxStatusPending`, `MxStatusWarning`, `MxStatusCommunicationError`, `MxStatusConfigurationError`, `MxStatusOperationalError`, `MxStatusSecurityError`, `MxStatusSoftwareError`, `MxStatusOtherError`. No inventar valores numéricos.

Loggers usan parámetro `msg`, no `Message` (palabra reservada). LogCustom y LogTrace incluyen `ObjectName.ScriptName` como componente; activar flags para ver salida, no interpretar silencio como éxito.

SendKeys: caracteres ordinarios concatenados; especiales `{BACKSPACE}`/`{BS}`, `{BREAK}`, `{CAPSLOCK}`, `{DELETE}`/`{DEL}`, `{DOWN}`, `{END}`, `{ENTER}`/`~`, `{ESCAPE}`/`{ESC}`, `{F1}`..`{F12}`, `{HOME}`, `{INSERT}`, `{LEFT}`, `{NUMLOCK}`, `{PGDN}`, `{PGUP}`, `{PRTSC}`, `{RIGHT}`, `{TAB}`, `{UP}`. Modificadores SHIFT=`+`, CTRL=`^`, ALT=`%`; agrupar con paréntesis, ejemplo `SendKeys("^(%(p))");` (pp. 73–74).

### Filtrado de eventos: alcance limitado de la fuente (pp. 65–66)

El manual incluye un apartado de event service: aplicar filtros **antes** de `StartRequestingEvent()`. Tipos: Integer, Float, String, Bool, DateTime, Double, Short, Array. Comparadores: `eq`; `beginswith` solo string; `lt/le/gt/ge` excluyen strings y arrays; `between` comprueba pares y admite numéricos/fecha. Negaciones impresas: `neg`, `nbegins`, `nlt`, `nle`, `ngt`, `nge`, `nbetween`. No inventar firma/receptor de StartRequestingEvent ni de registro de filtros: este manual no los define.

## Firmas con seguridad de operador (pp. 77–84)

### SignedAlarmAck

```text
int SignedAlarmAck(String Alarm_List,
    Boolean Signature_Reqd_for_Range,
    Integer Min_Priority,
    Integer Max_Priority,
    String Default_Ack_Comment,
    Boolean Ack_Comment_Is_Editable,
    String TitleBar_Caption,
    String Message_Caption);
```

**Solo cliente**, no objeto. Alarm_List string de alarmas de tags/atributos separadas por espacios o comas, máximo 1024 caracteres; ejemplo permite array de strings `arr[]` convertido a lista. Una alarma inválida invalida toda la operación. Firma opcional por rango, prioridades 1..999 y mínimo ≤ máximo. Default_Ack_Comment máximo 200, título 1024, mensaje 250 caracteres. False en Ack_Comment_Is_Editable bloquea edición; comentario vacío no muestra default. Título vacío usa título del sistema. Message_Caption no se propaga al registro de eventos.

Retornos (pp. 79–80): −1 usuario canceló; −2 ninguna alarma pendiente; 0 parámetros/credenciales válidos, sin cancelación y escritura a `.AckMsg`; 1 otro fallo; 2 coerción de tipo; 3 Alarm_List inválida; 4 prioridad fuera de rango; 5 mínimo > máximo. **0 no demuestra reconocimiento**, que puede fallar por permisos o porque ya se reconoció.

### SignedWrite

```text
int SignedWrite(string Attribute,
    object Value,
    string ReasonDescription,
    Bool Comment_Is_Editable,
    Enum Comment_Enforcement,
    string[] Predefined_Comment_List);
```

**Solo cliente**, no objeto. Destino AutomationObject con clasificación Secured Write o Verified Write. Attribute string admite referencias bound/nested y CP entre comillas que se resuelve a referencia. Value debe coincidir con tipo del destino; NULL si no se introduce valor. ReasonDescription máximo 256 caracteres; el manual describe uso de descripción de Field Attribute si existe, o Short Description del ApplicationObject.

Comment_Enforcement: Optional=0, Mandatory=1, PredefinedOnly=2. No inventar namespace del enum: ejemplos usan números. Lista de hasta 20 comentarios, máximo 200 caracteres cada uno; puede ser array vacío o null. Con editable=True, PredefinedOnly permite solo selección; los demás permiten edición. Con editable=False no se muestra selector y no se edita texto. La descripción de cómo se aplica el comentario predefinido contiene matices: validar la configuración real del diálogo si es crítica.

Retornos (p. 83): 0 escritura puesta en cola; 1 otro fallo; 2 coerción de tipos; 3 Attribute inválido, inexistente o sin clasificación requerida; 4 enum inválido. **0 no prueba actualización**: cancelación posterior, permisos o coerción pueden impedirla. No automatizar una sustitución que eluda la firma requerida.

## Strings y tiempo (pp. 85–101)

Ámbito general, sujeto al host. Text/strings pueden ser literales o atributos String según entrada; nombres IntegerResult/StringResult/RealResult son variables receptoras de los ejemplos.

| Call documentada | Retorno, argumentos y límites | Páginas |
|---|---|---|
| `DText(Discrete, OnMsg, OffMsg);` | String: OnMsg si Boolean True, OffMsg si False | 85 |
| `StringASCII(Char);` | Integer ASCII del **primer** carácter | 86 |
| `StringChar(ASCII);` | String carácter del código Integer; 13 CR, 10 LF, 34 comilla | 86 |
| `StringCompare(Text1, Text2);` | −1, 0, 1 para menor/igual/mayor | 87 |
| `StringCompareNoCase(Text1, Text2);` | Igual comparación ignorando mayúsculas; cabecera/ejemplos así, pero línea Syntax imprime erróneamente `SStringCompareNoCase` | 87–88 |
| `StringFromGMTTimeToLocal(SecsSince1-1-70, StringType);` | String: segundos UTC desde epoch → presentación local; DST automático; manual lo equipara a StringFromTime | 88–89 |
| `StringFromIntg(Number, numberBase);` | String representación del entero en base; ejemplos bases 2/8/16 | 89 |
| `StringFromReal(Number, Precision, Type);` | String; Precision número de decimales Integer, Type `"f"`, `"e"`, `"E"` para fijo/exponencial | 89–90 |
| `StringFromTime(SecsSince1-1-70, StringType);` | String; segundos UTC → representación local según texto. Ejemplos p. 90 no declaran zona: no inferir UTC de la salida | 90–91 |
| `StringFromTimeLocal(SecsSince1-1-70, StringType);` | String local; DST automático. No asumir una conversión adicional no explicada por el manual | 91 |
| `StringInString(Text, SearchFor, StartPos, CaseSens);` | Integer primera posición encontrada desde StartPos, base 1; 0 no encontrado. CaseSens 0 ignora case, 1 sensible | 92 |
| `StringLeft(Text, Chars);` | String primeros Chars; **0 devuelve todo**, exceso también todo | 92–93 |
| `StringLen(Text);` | Integer longitud, incluye espacios y caracteres no visibles | 93 |
| `StringLower(Text);` | String en minúsculas, otros caracteres no cambian | 93–94 |
| `StringMid(Text, StartChar, Chars);` | String desde StartChar (base 1) y cantidad Chars; limita al final del texto | 94 |
| `StringReplace(Text, SearchFor, ReplaceWith, CaseSens, NumToReplace, MatchWholeWords);` | String; CaseSens/MatchWholeWords 0 no, 1 sí; NumToReplace=−1 todas. Caracteres especiales `@ # $ % & * ( )` se tratan como delimitadores según manual; no prometer reemplazo literal universal | 94–95 |
| `StringRight(Text, Chars);` | String últimos Chars; **0 devuelve todo**, exceso también todo | 95–96 |
| `StringSpace(NumSpaces);` | String de NumSpaces espacios | 96 |
| `StringTest(Text, TestType);` | Boolean del **primer carácter**; tipos de test debajo | 97 |
| `StringToIntg(Text);` | Integer, lee prefijo numérico, ignora blancos; primer carácter no numérico →0. `"22.2 is the Value"`→22; no valida toda la cadena | 97–98 |
| `StringToReal(Text);` | Real, prefijo numérico; no numérico inicial →0; `"22.261 is the value"`→22.261 | 98 |
| `StringTrim(Text, TrimType);` | String; 1 quita blancos iniciales, 2 finales, 3 quita extremos y colapsa interiores a un espacio; whitespaces ASCII 09..0D/20 | 99 |
| `StringUpper(Text);` | String en mayúsculas | 99–100 |
| `Text(Number, Format);` | String por formato; `Text(22.269,"#.00")`→`"22.27"`. No confundir con Precision de Round | 100 |
| `wwStringFromTime(SecsSince1-1-70, StringType);` | String: valor en hora local → UTC, DST automático; contrario al sentido de GMT→local | 100–101 |

StringType compartido: 1 fecha según Windows; 2 hora según Windows; 3 fecha+hora de 24 caracteres; 4 día de semana abreviado; 5 día completo. Idioma, zona y DST forman parte del resultado. Para comparación/almacenamiento fijar explícitamente origen UTC/local y unidades; no ajustar DST manualmente otra vez.

StringTest TestType: 1 alfanumérico A–Z/a–z/0–9; 2 numérico 0–9; 3 alfabético; 4 mayúscula; 5 minúscula; 6 puntuación 0x21..0x2F según tabla; 7 ASCII 0x00..0x7F; 8 hexadecimal; 9 imprimible 0x20..0x7E; 10 control 0x00..0x1F o 0x7F; 11 whitespace 0x09..0x0D o 0x20. No confundir «primer carácter numérico» con validar un número completo.

## Sistema y DDE (pp. 101–104)

| Call documentada | Retorno y contrato | Páginas |
|---|---|---|
| `CreateObject(ProgID);` | Object COM/ActiveX por ProgID string; depende de registro/instalación/arquitectura. No autoriza lanzar GUI desde servidor | 101 |
| `Now();` | Hora actual, variable receptora TimeValue; se puede formatear con .NET | 101–102 |
| `WWExecute(Application, Topic, Command);` | Integer status DDE: 1 enviado, 0 ocupado, −1 error. Strings aplicación/topic/comando | 102 |
| `WWPoke(Application, Topic, Item, TextValue);` | Integer status DDE 1/0/−1. Strings; convertir número a texto. View→View indefinido/no soportado | 103 |
| `WWRequest(Application, Topic, Item, Attribute);` | Integer status DDE 1/0/−1; Attribute destino string por referencia/nombre entre comillas según descripción; recibe texto que luego puede convertirse | 103–104 |

DDE es legado. En síncronos no llamar repetidamente en bucle, ni varias llamadas seguidas en el mismo script, ni tareas largas de otra aplicación. El manual admite estas acciones en asíncronos. La p. 103 bajo WWPoke nombra WWRequest en su advertencia: conservar la cautela en ambas; no tratarlo como autorización implícita para ráfagas síncronas. Las menciones legacy `Execute/Poke/Request` no aportan nuevas firmas alternativas.

## Inconsistencias que no deben convertirse en APIs

| Fuente | Lectura para esta skill |
|---|---|
| Syntax p. 87 `SStringCompareNoCase` | Cabecera y ejemplos `StringCompareNoCase`; corrección editorial propuesta, comprobar en IDE |
| p. 124 `IsUseable` | Entrada formal `IsUsable`, pp. 68–69 |
| p. 69 ejemplo LogCustom carece de `)` | Firma `LogCustom(CustomFlag, msg);`; corregir puntuación al adaptar |
| p. 30 mezcla `graphicInfo` y `graphicInfo0` | Usar la misma variable declarada; no cambia firma |
| p. 38 ejemplo CustomPropertyValuePair con cuatro argumentos | Otros ejemplos pp. 38–39 usan tres (nombre, valor, IsConstant); no inventar overload de cuatro |
| p. 29 etiquetas PropertyOverrides/GraphicName en sección ContentInfo | Utilizar miembro mostrado `PropertyOverrideValues` p. 26 y `Content`; verificar ayuda instalada si difiere |

Una inconsistencia documental no demuestra por sí sola el comportamiento del compilador SP2023. Citarla cuando cambie la respuesta.
