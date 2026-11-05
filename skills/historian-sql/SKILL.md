---
name: aveva-sp-historian-sql
description: Escribir, revisar y diagnosticar consultas SQL de recuperación de AVEVA Historian mediante INSQL, History, WideHistory, resúmenes y Events. Usar para modos de recuperación, ciclos, calidad, zona horaria, caudales, contadores y tiempos por estado. No sustituye la skill QuickScript .NET ni documenta esquemas ajenos a Historian.
---

# AVEVA Historian SQL

Fuente: *AVEVA Historian Retrieval Guide*, 3 de diciembre de 2020, 243 páginas. Objetivo: uso con System Platform/Historian 2023, sin afirmar compatibilidad individual probada. Las citas indican páginas internas, coincidentes con las posiciones físicas del PDF suministrado. No se redistribuye el PDF.

## Resolver la consulta

1. Identificar interfaz (SQL Server/INSQL o REST), versión real de Historian, servidor/base/linked server, tags y tipos, intervalo, zona horaria y resultado requerido: muestras, tendencia, agregado temporal, cantidad integrada, contador, estados o alarmas. Obtener nombres y metadatos del proyecto o herramientas conectadas cuando estén disponibles; preguntar solo por datos que afecten a la solución.
2. Leer la referencia relevante antes de fijar semántica o sintaxis:
   - [querying.md](references/querying.md): tablas, columnas usadas, vistas, OPENQUERY, restricciones del proveedor y diagnóstico.
   - [retrieval-modes.md](references/retrieval-modes.md): selección de modo y compatibilidad por tipo de tag.
   - [options-quality-time.md](references/options-quality-time.md): parámetros exactos, matriz de opciones, ciclos fantasma, límites, unidades y calidad.
   - [examples.md](references/examples.md): consultas adaptadas con contexto, requisitos y resultados verificables.
   - [alarms-events.md](references/alarms-events.md): esquema observado de Events, frecuencia y duración de alarmas.
   - [interfaces.md](references/interfaces.md): orientación REST/OData/SSRS del capítulo 5; consultar solo si la tarea usa esa interfaz.
3. Entregar SQL con contexto de conexión, nombres que haya que sustituir, modo explícito, zona/intervalo y política de calidad. Separar el SQL que procesa SQL Server del texto que procesa INSQL. Identificar ejemplos adaptados y la fuente; solo llamar «ejecutado» a lo que realmente se haya probado.

## Decisiones que no se pueden omitir

- `History` es una tabla virtual sobre history blocks, no una tabla SQL ordinaria. `WideHistory` requiere OPENQUERY; no generar su SELECT directo como si fuera una vista local.
- `AVG(Value)` promedia filas; `Average` pondera por tiempo. Para volumen de un caudal usar `Integral` y comprobar `EngineeringUnit.IntegralDivisor`/unidades. Para un totalizador con rollover usar `Counter`, no integrar ni restar extremos sin revisar su configuración.
- Elegir **wwResolution o wwCycleCount**, nunca ambos: el manual es contradictorio sobre su combinación. Las filas iniciales agregadas pueden corresponder al ciclo anterior. Mostrar `StartDateTime` para interpretar el período; en History no usarlo como sustituto del filtro DateTime.
- Los límites se eligen por modo. `[inicio, fin)` evita duplicar muestras en extracciones por tramos, pero no es regla universal para agregados: Minimum/Maximum/ValueState piden fin inclusivo y los ejemplos tienen excepciones señaladas.
- Conservar `OPCQuality`, `QualityDetail`, `PercentGood` y/o tiempos válidos cuando cambien la interpretación. `Quality=0` y `OPCQuality=192` pertenecen a codificaciones distintas; `Quality=133` identifica un valor inicial, no equivale por sí solo a Bad.
- La zona de recuperación no es la zona del ordenador cliente. No combinar GETDATE local con wwTimeZone distinta sin normalizar ambos límites. Descripción/unidades consultadas hoy no prueban metadatos históricos del intervalo.
- Retrieval Guide no es Database Reference: usar solo tablas/columnas documentadas aquí o verificadas en la instalación. No inventar procedimientos, valores de enums ni equivalencias SQL/REST.

## Alcance y verificación

La operación por defecto es lectura. Una petición de análisis no autoriza INSERT/UPDATE/DELETE, anotaciones, ALTER de configuración, creación de linked servers o escritura de históricos. Los ejemplos de escritura del manual se describen sin ejecutarlos.

Para validar en destino: probar primero una ventana pequeña y tags conocidos, verificar unidad, número de ciclos, límites y calidad, y comparar un caso de referencia. No ejecutar automáticamente un Full largo en producción. Una consulta vacía puede deberse a opciones incompatibles, tipos o filtros mal delegados; no demuestra ausencia de datos.
