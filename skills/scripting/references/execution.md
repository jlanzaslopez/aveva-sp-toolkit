# Ejecución y contexto

Fuente: *AVEVA Application Server Scripting Guide*, octubre 2020, páginas internas citadas. Objetivo SP2023; confirmar dependencias y host en la instalación. Contenido: matriz de contexto; ciclo de vida; triggers; binding; cliente/editor.

## Matriz de contexto

| Familia/API | Objeto Application Server | Símbolo Industrial Graphics | Layout/ViewApp OMI | Restricción y página |
|---|---|---|---|---|
| GetCPQuality, GetCPTimeStamp | No documentado como API de objeto | Sí, propiedades del símbolo | Dentro del contexto de símbolo | Solo custom properties; no tags HMI; soporte de GetCPQuality depende del HMI, pp. 19–20 |
| ShowContent, HideContent | No soportado, aunque aparezca en browser | Sí, alojado en OMI | Sí | OMI exclusivamente; pp. 20–29 |
| ShowGraphic | No usar como API de servidor | Sí | Sí | Popup; OMI e InTouch, pp. 25, 30 |
| HideGraphic, HideSelf | No documentado como API de objeto | Sí | HideSelf cierra gráfico/layout mostrado | Identidad/contexto local; pp. 24–25 |
| Logoff, ShowLoginDialog | No | Acción gráfica de ViewApp | ViewApp | Left-Click/Key/Touch Down; pp. 25, 39 |
| Familia InTouch | No funciona y puede no avisar | Sí, host InTouch y excepciones por función | No suponer compatibilidad OMI | Regla general p. 40; `LogonCurrentUser` solo scripting InTouch, no ArchestrA client scripting, p. 53 |
| LogDataChangeEvent | Sí | No operativo, warning en runtime | No extrapolar | Solo objetos; pp. 69–70 |
| SignedWrite, SignedAlarmAck | No soportado | Cliente | Cliente según host | No confundir retorno 0 con efecto final; pp. 77–84 |
| SetAttributeVT/VT2 y SetBad/Good/Initializing/Uncertain | Destino dentro del mismo objeto | No extrapolar a cliente | No extrapolar | pp. 74–77 |
| Matemática, strings, logging, Now | Generales, salvo restricción específica | Según capacidades del host | Según capacidades del host | «General» no certifica todos los HMI; pp. 19, 58–104 |
| StartApp/automatización GUI | Lanzar aplicaciones GUI no soportado desde servidor | Cliente con dependencias | Verificar host | pp. 19, 56; COM no elimina esta restricción |

La pertenencia a una carpeta del Function Browser no garantiza un ámbito compatible. Consultar [functions.md](functions.md) para excepciones individuales.

## Ciclo de vida del objeto (pp. 8–9)

| Método | Momento | Uso y límite |
|---|---|---|
| Startup | Carga en memoria: deployment, arranque de plataforma/engine | Instanciar COM/.NET; las asignaciones a atributos pueden fallar por carga; atributos off-object no disponibles |
| OnScan | Primera ejecución tras pasar a OnScan | Inicializar atributos locales y crear objetos; atributos off-engine no disponibles |
| Execute | Evaluado por scans mientras objeto OnScan | Lógica de runtime y referencias dinámicas; trigger selecciona ejecuciones |
| OffScan | Objeto sale de scan, directamente o por engine | Limpieza y preparación para dejar de ejecutar |
| Shutdown | Antes de retirar objeto de memoria, usualmente al parar AppEngine | Destruir/liberar objetos y recursos |

«Deployment Scripts» de p. 9 analiza Startup/OnScan durante despliegue; no define un sexto evento que se deba programar con ese nombre. Los scripts deben completar dentro del timeout de despliegue. Startup/OnScan se ordenan por nombre alfanumérico dentro del Area, no por execution order. Evitar allí bases de datos, archivos CSV/XML/TXT, referencias off-object y referencias dinámicas. Un ejemplo posterior de SQL en Startup (p. 122) o BindTo en Startup (p. 126) no anula esta recomendación expresa.

Al sacar de scan un objeto se solicita terminar sus scripts asíncronos activos mediante un atributo booleano de shutdown. Comprobarlo antes/después de operaciones largas; la p. 9 no proporciona su nombre concreto: obtenerlo del objeto/IDE. Tras 30 s sin finalizar se registra warning, pero no hay terminación forzada; la espera en el hilo principal puede bloquear el engine y el script podría agotar timeout antes de completar su lógica.

## Triggers Execute (pp. 8–9)

| Trigger | Condición |
|---|---|
| Periodic | Primera ejecución en el siguiente scan programado al entrar OnScan; luego según período transcurrido |
| Data Change | Cambio de valor **o calidad** entre scans |
| OnTrue | False en un scan → True en el siguiente |
| OnFalse | True en un scan → False en el siguiente |
| WhileTrue | Expresión True al inicio del scan; sujeto al período configurado |
| WhileFalse | Expresión False al inicio del scan; sujeto al período configurado |

El manual introduce la lista en relación con la casilla Quality marcada (p. 8); no define exhaustivamente su comportamiento desmarcada. Indicar esa configuración al reproducir el caso. Los cambios False→True→False que ocurren entre muestras pueden no detectarse: se evalúa el valor al inicio, no todos los flancos físicos.

Período 0 significa ejecutar cada scan. WhileTrue/WhileFalse/Periodic calculan tiempo desde la marca temporal anterior, no desde un contador monotónico. Ajustes del reloj pueden alargar/acortar el intervalo real: el ejemplo 11:13→12:13 con retraso del reloj de 3 min tarda 63 min reales (p. 9).

## Referencias dinámicas (pp. 14–16, 123–126)

Para configurar InputSource/OutputDest, el manual usa Execute, WhileTrue con expresión `NOT Me.Ref_Done`, período 0 y una comprobación `Me.Set_Refs.ExecutionCnt == 2`. Al completar pone Ref_Done=True. Los valores de referencia del ejemplo son checkpointed y no requieren reconstrucción en cada arranque/failover. Esa espera de dos ejecuciones es un patrón de configuración concreto; **no** garantiza que cualquier referencia remota esté preparada en dos scans.

`Indirect` permite leer/escribir la referencia enlazada con `x.BindTo(s);`, donde `s` es una cadena ArchestrA. Escribir `x = valor;` escribe al destino enlazado. Sin binding no devuelve datos. `WriteStatus(x)` consulta el último write, no confirma binding ni actualidad.

Para off-engine:

1. Declarar `Indirect` y estado en **Declarations**, para conservarlos entre scans.
2. En Execute realizar `BindTo`, pasar a estado de espera y salir de esa rama.
3. En un **scan posterior** comprobar `IsGood` de todas las referencias requeridas; solo entonces consumirlas.
4. Mantener espera o timeout explícito si no resuelven. Definir política de recuperación/rebinding según el caso; el timeout es opcional en el manual, pero no prometer éxito ilimitado.

La referencia same-engine es inmediata según p. 124; off-engine puede necesitar varios scans. La calidad Good es la recomendación de pp. 124–125. `IsUsable` permite además Uncertain, por lo que no sustituye ese requisito.

Con Advanced Communication Management no se recomienda Indirect sobre referencias de scan groups DIObject **ActiveOnDemand**: activación y ejecución no están sincronizadas, y la comprobación puede permanecer False. Reutilizar el mismo Indirect para varios DI items puede cambiar de comportamiento al habilitar ACM. La grafía `IsUseable()` de la explicación de p. 124 es inconsistente: la función documentada es `IsUsable()` (pp. 68–69).

## Cliente InTouch con Advanced Communication Management (pp. 16–17)

| Acción de ventana WindowViewer | Efecto en símbolos embebidos |
|---|---|
| Abrir | Registrar/advisar referencias si hace falta; ejecutar OnShow y scripts con trigger satisfecho |
| Cerrar | Ejecutar OnHide, parar scripts; unadvise y unregister si ninguna otra ventana usa referencias |
| Minimizar | Parar scripts; unadvise si no hay otros consumidores; **no** ejecutar OnHide |
| Maximizar/restaurar | Advisar referencias si hace falta y ejecutar scripts con trigger satisfecho |

While Showing no ejecuta mientras la ventana está minimizada. No basar una tarea continua de servidor en la visibilidad de un símbolo. Estas reglas se documentan para WindowViewer con ACM; no extrapolarlas a todas las ventanas OMI.

## Editor y diagnóstico (pp. 10–18)

Ctrl+Space ofrece autocompletado, referencias, parámetros y tipos; `me.` navega atributos y `InTouch:` tags del ViewApp seleccionado. Es ayuda de edición, no comprobación de soporte en runtime. Los errores aparecen subrayados tras unos 2,5 s de inactividad; tooltip y Validate Script muestran el mensaje. Validar **y guardar el objeto** para detectar también declaraciones pendientes (p. 109).

Undo/redo se pierde al cerrar editor, cambiar script o pestaña de evento. Go To ayuda a localizar líneas; la vista sin zoom admite cuatro dígitos y puede recortar líneas >9999. Para diagnóstico usar logging; LogCustom/LogTrace requieren flags habilitados en SMC (p. 18). No se reproducen iconos ni colores del editor porque no afectan al código generado.
