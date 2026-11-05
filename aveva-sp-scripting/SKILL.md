---
name: aveva-sp-scripting
description: Escribir, revisar y diagnosticar QuickScript .NET de AVEVA System Platform, incluidos scripts de objetos Application Server y clientes Industrial Graphics, OMI e InTouch. Usar para sintaxis, triggers, calidad, referencias Indirect y funciones de scripting; no para SQL de Historian ni scripting de otros productos AVEVA.
---

# AVEVA System Platform scripting

Base documental: *AVEVA Application Server Scripting Guide*, 2 de octubre de 2020, 130 páginas. Objetivo de uso: System Platform 2023 por decisión del usuario. La compatibilidad de cada API con 2023 no está certificada por esta síntesis. Las páginas citadas son las internas del manual y coinciden con la posición física del PDF suministrado. El PDF no forma parte de la skill.

## Trabajar con esta skill

1. Identificar el contexto: objeto Application Server, símbolo Industrial Graphics alojado en InTouch u OMI, o layout OMI; además, evento/trigger, sincronía y ubicación del atributo (local, otro objeto, otro engine). Preguntar solo por datos que cambien la solución y no consten en el proyecto.
2. Consultar la referencia de la rama necesaria:
   - [syntax.md](references/syntax.md): declaraciones, arrays, operadores, calidad en expresiones, control y excepciones.
   - [execution.md](references/execution.md): ciclo de vida, triggers, restricciones por contexto, comunicación y enlace entre engines.
   - [functions.md](references/functions.md): firmas y restricciones de todas las funciones con entrada propia en el manual. Buscar por nombre antes de generar una llamada.
   - [examples.md](references/examples.md): patrones cotejados, dependencias y erratas de los ejemplos originales.
3. Generar QuickScript .NET, manteniendo nombres, orden de argumentos, puntuación y unidades documentados. Las firmas del manual que no declaran tipos no autorizan a inventarlos. Métodos .NET, librerías importadas y APIs fuera del manual necesitan su propia documentación o verificación en el IDE instalado.
4. Entregar código con su ubicación (Declarations/cuerpo y evento/trigger), atributos y tipos requeridos, dependencias y tratamiento del fallo. Distinguir «cotejado con el manual», «adaptado» y «compilado/ejecutado en SP2023». Solo afirmar las pruebas realmente realizadas.

## Invariantes que cambian la solución

- Una función visible en IntelliSense no demuestra que esté soportada en ese contexto. Resolver el ámbito con la matriz de ejecución y las restricciones de su entrada.
- El trabajo de ejecución y las referencias dinámicas van en Execute. Startup/OnScan tienen limitaciones de disponibilidad y despliegue; no trasladar allí acceso a archivos o bases de datos siguiendo un ejemplo antiguo aislado.
- Un enlace off-engine necesita estado persistente entre scans y comprobación de calidad en un scan posterior. Un `WHILE` dentro del cuerpo no sustituye a un trigger WhileTrue.
- Conservar calidad y seguridad de los atributos: pasar por un DIM puede perder la calidad de origen; retorno 0 en SignedWrite/SignedAlarmAck no confirma el efecto final.
- En las referencias, «general» significa que la entrada no establece una prohibición específica; no certifica todos los HMI. Aplicar las restricciones del host.
- Las erratas se identifican junto a la API o ejemplo. No convertir `IsUseable` o `SStringCompareNoCase` en funciones nuevas.

## Límite documental

Esta es una síntesis técnica original, no una copia del manual. Cubre sus capítulos operativos y catálogo, omitiendo capturas, iconos del editor, repeticiones e índice alfabético. No implica permiso para desplegar, escribir en planta, reconocer alarmas o enviar mensajes. Para APIs o diferencias de versión no resueltas, consultar la ayuda de la instalación SP2023 y declarar la incertidumbre.
