---
name: aveva-sp-object-viewer
description: Generar y editar ficheros XML de watchlists de AVEVA Object Viewer con pestañas, referencias ordenadas y arrays; ayudar con la apertura del modo avanzado.
---

# Object Viewer: watchlists

Formato definido por ejemplos aportados por el usuario para System Platform 2023, sin manual PDF. La importación real sigue pendiente de comprobar.

## Entregar el fichero

Cuando se solicite una watchlist, generar un fichero `.xml` UTF-8 y entregar su enlace descargable o local, listo para abrir desde Object Viewer. No limitar la entrega a un bloque de código. Si el entorno no permite crear archivos, explicarlo y facilitar el XML completo para guardarlo como `.xml`.

Usar una raíz `VisualLMXTestTool`, un `Watch` por pestaña con atributo `Tab`, y un `ReferenceString` por referencia completa. Conservar exactamente nombres, mayúsculas, orden de pestañas y orden de referencias. No ordenar ni eliminar duplicados sin petición. Si se proporcionan objetos y atributos compartidos, combinarlos conforme a la agrupación solicitada; preguntar solo si faltan datos necesarios.

No inventar atributos, sufijos, índices ni referencias de diagnóstico. Admitir referencias con varios puntos, como `317_S02_CNT001.scBalanceControl.ExecutionCnt`, sin reescribirlas.

## Arrays

- Array entero: `111_S09._Attributes[]`.
- Elemento concreto: `111_S09._TempCntsBySeverity[2]`.
- Pueden coexistir en la misma pestaña y en el orden solicitado. No expandir el array entero ni inferir su tamaño, límites o base de índices.

Ejemplo aportado por el usuario, formateado sin cambiar su contenido:

```xml
<VisualLMXTestTool>
  <Watch Tab="Watch List 1">
    <ReferenceString>111_S09._Attributes[]</ReferenceString>
    <ReferenceString>111_S09._TempCntsBySeverity[2]</ReferenceString>
  </Watch>
</VisualLMXTestTool>
```

Para varias pestañas, añadir otros elementos `Watch` hermanos dentro de la misma raíz; no generar varios documentos concatenados.

Antes de entregar, analizar el archivo con un parser XML, comprobar raíz, nombres de elementos, `Tab` y referencias no vacíos, y comparar los valores y su orden con lo pedido. Escapar caracteres XML mediante un serializador: el valor leído debe coincidir con el original. No añadir namespaces ni campos no documentados. Al editar un archivo existente, conservar contenido fuera del cambio solicitado; si contiene estructuras desconocidas, no descartarlas silenciosamente.

La validación XML no verifica la existencia de atributos en la Galaxy ni demuestra que Object Viewer importe el archivo. No afirmar que el doble clic abre Object Viewer, ni inventar argumentos para importar el XML: la asociación de archivos y la apertura deben comprobarse en la instalación de destino.

## Modo avanzado: comando y diagnóstico

Cuando se solicite abrir el modo avanzado, devolver el comando adecuado a la consola. El extracto de guía aportado indica `aaObjectViewer.exe /New` en `C:\Program Files (x86)\ArchestrA\Framework\Bin`. El usuario ha comunicado que no funciona en su entorno: conservar `/New` como indicación de esa fuente, sin presentarlo como probado ni sustituirlo por opciones inventadas.

CMD, si existe el ejecutable en esa ruta:

```cmd
cd /d "C:\Program Files (x86)\ArchestrA\Framework\Bin"
aaObjectViewer.exe /New
```

PowerShell, misma ruta y argumento, con sintaxis propia de la consola:

```powershell
& "C:\Program Files (x86)\ArchestrA\Framework\Bin\aaObjectViewer.exe" /New
```

Si falla, identificar consola, equipo de ejecución, ruta real, versión y error exacto; distinguir fallo al localizar/ejecutar el programa de apertura sin modo avanzado. En PowerShell, `cd /d` no es la sintaxis de CMD y ejecutar desde la carpeta requiere `./aaObjectViewer.exe` o una ruta con el operador `&`. Comprobar primero la existencia del archivo; no concluir que el argumento es incorrecto por un error de ruta o de consola. No cerrar instancias ni elevar permisos como supuesto remedio.
