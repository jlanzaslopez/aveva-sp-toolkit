# AVEVA SP Toolkit

Complemento de skills para trabajar con AVEVA System Platform. Cada skill contiene un `SKILL.md` como punto de entrada y referencias técnicas que se consultan según la tarea.

## Skills

| Skill | Estado | Contenido |
|---|---|---|
| [Scripting](skills/scripting/SKILL.md) | Primera versión | QuickScript .NET: sintaxis, ejecución, funciones y ejemplos para objetos Application Server, Industrial Graphics, OMI e InTouch |
| Historian SQL | Pendiente | Se incorporará en `skills/historian-sql/` cuando se prepare su contenido |

## Estructura actual

```text
aveva-sp-toolkit/
├── .agents/
│   └── plugins/
│       └── marketplace.json
├── plugin.json
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

## Añadir el marketplace en ChatGPT

El repositorio contiene un complemento Agent Plugins 1.0 (`plugin.json`) y un catálogo Codex (`.agents/plugins/marketplace.json`). El catálogo publica un único complemento, **AVEVA SP Toolkit**, desde la raíz del repositorio; sus skills se descubren dentro de `skills/`.

En un workspace con importación desde GitHub disponible, un administrador puede ir a **Admin > Plugins > Add > Import marketplace** e indicar:

- **Source:** `https://github.com/jlanzaslopez/aveva-sp-toolkit`
- **Path:** raíz del repositorio (dejar vacío cuando el formulario lo permita).
- **Branch, tag, or commit:** `main`.

Revisar el informe de importación y habilitar el complemento para los roles deseados. Para incorporar cambios posteriores del repositorio, usar **Sync now** y revisar su informe. La disponibilidad de esta opción depende de la cuenta y de los permisos del workspace.

En clientes locales compatibles, el catálogo también permite descubrir el complemento desde este repositorio. Las rutas del catálogo se resuelven desde la raíz del repositorio, no desde `.agents/plugins/`. El complemento solo proporciona instrucciones y referencias: no necesita servidor MCP ni conexión a una Galaxy.

Referencias oficiales: [formato del complemento y marketplace](https://developers.openai.com/plugins/build/plugins) e [importación y sincronización desde GitHub](https://learn.chatgpt.com/docs/enterprise/plugin-management). La sintaxis y las rutas se comprueban antes de publicar; la importación real en ChatGPT todavía no se ha probado.

## Fuentes y validación

Scripting se basa en el *AVEVA Application Server Scripting Guide*, publicado el 2 de octubre de 2020. Las referencias citan páginas internas del manual y distinguen sus erratas de las adaptaciones propuestas. El PDF original no se redistribuye.

El objetivo de uso es System Platform 2023. Los ejemplos se han cotejado documentalmente; todavía no se han compilado ni ejecutado en esa versión. Antes de usarlos en una instalación, comprobar contexto, dependencias, calidad de datos y permisos. El repositorio no instala ni despliega cambios automáticamente.
