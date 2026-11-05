# AVEVA SP Toolkit

Skills para trabajar con AVEVA System Platform. Cada skill contiene un `SKILL.md` como punto de entrada y referencias técnicas que se consultan según la tarea.

## Skills

| Skill | Estado | Contenido |
|---|---|---|
| [Scripting](skills/scripting/SKILL.md) | Primera versión | QuickScript .NET: sintaxis, ejecución, funciones y ejemplos para objetos Application Server, Industrial Graphics, OMI e InTouch |
| Historian SQL | Pendiente | Se incorporará en `skills/historian-sql/` cuando se prepare su contenido |

## Estructura actual

```text
aveva-sp-toolkit/
├── skills/
│   └── scripting/
│       ├── SKILL.md
│       └── references/
│           ├── syntax.md
│           ├── execution.md
│           ├── functions.md
│           └── examples.md
└── README.md
```

La estructura prevista para Historian SQL es `skills/historian-sql/SKILL.md` con su carpeta `references/`. Todavía no se incluye una skill vacía.

## Uso

Abrir el `SKILL.md` de la especialidad y seguir sus enlaces a las referencias. La carpeta `skills/scripting/` contiene la skill cuyo identificador es `aveva-sp-scripting`; sus enlaces son relativos y se conserva como unidad al copiarla o instalarla.

## Fuentes y validación

Scripting se basa en el *AVEVA Application Server Scripting Guide*, publicado el 2 de octubre de 2020. Las referencias citan páginas internas del manual y distinguen sus erratas de las adaptaciones propuestas. El PDF original no se redistribuye.

El objetivo de uso es System Platform 2023. Los ejemplos se han cotejado documentalmente; todavía no se han compilado ni ejecutado en esa versión. Antes de usarlos en una instalación, comprobar contexto, dependencias, calidad de datos y permisos. El repositorio no instala ni despliega cambios automáticamente.
