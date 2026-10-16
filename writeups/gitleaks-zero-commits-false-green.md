# Gitleaks en verde, cero commits analizados: cómo comprobar el alcance real del escaneo

Un control de secretos terminó en verde después de integrar cambios. Al revisar el log, el dato que importaba no era el número de hallazgos: el escáner había informado de cero commits procesados.

El resultado no demostraba una filtración. En ese caso se había aceptado como satisfactorio un control sin acreditar que hubiese procesado los commits esperados. El alcance de Git y el contador del escáner necesitaban comprobarse por separado.

La corrección exigió definir el alcance, incluir los cambios de los merges y rechazar los resultados vacíos o incompletos. Este artículo explica el problema de integración y las comprobaciones adoptadas. No presenta un descubrimiento original ni atribuye el comportamiento a todas las versiones de Gitleaks o de su acción de GitHub.

## El historial descargado no es el historial analizado

Gitleaks puede analizar los parches del historial Git. Las opciones de `--log-opts` determinan qué recibe desde Git. Tener todos los commits disponibles no obliga al escáner a procesarlos. La distinción está documentada en el [modo Git de Gitleaks](https://github.com/gitleaks/gitleaks#git).

También conviene comprobar el checkout: `actions/checkout` permite solicitar el historial completo con `fetch-depth: 0`. Los objetos necesarios deben estar disponibles y sus referencias deben resolver; la selección posterior sigue siendo responsabilidad del escaneo. [Documentación de checkout](https://github.com/actions/checkout#fetch-all-history-for-all-tags-and-branches).

Un paso llamado «escanear todo el historial» puede acabar ejecutando un rango limitado por su adaptador. El nombre del paso no acredita el alcance.

## Dos decisiones diferentes: recorrer commits y mostrar sus cambios

Las opciones de Git pueden recortar el historial o cambiar los parches producidos:

- `--first-parent` sigue únicamente el primer padre al atravesar un merge.
- `--no-merges` excluye los commits de merge.
- `--diff-merges=first-parent` muestra el cambio de un merge respecto de su primer padre. No equivale a restringir el recorrido con `--first-parent`.

Además, `git log -p` no muestra por defecto los parches de los merges, salvo opciones que activen ese comportamiento. Enumerar sus commits y obtener sus cambios son problemas distintos. [Documentación de git log](https://git-scm.com/docs/git-log).

De aquí salen dos riesgos: omitir commits de ramas incorporadas y omitir contenido introducido al resolver una fusión. En determinadas topologías y rangos, el filtrado puede incluso dejar al escáner sin commits procesables. No significa que cualquier push de un merge produzca un escaneo vacío.

## El antecedente público que hay que reconocer

La [issue #236 de gitleaks/gitleaks-action](https://github.com/gitleaks/gitleaks-action/issues/236), abierta por `sumanthratna`, ya describía un falso negativo relacionado: el recorrido por primer padre podía dejar fuera contenido incorporado desde otra rama y la acción reportaba un resultado limpio.

El [código de la acción en la revisión pública e0c47f4](https://github.com/gitleaks/gitleaks-action/blob/e0c47f4f8be36e29cdc102c57e68cb5cbf0e8d1e/src/gitleaks.js#L85-L131) permite comprobar las opciones aplicadas a los eventos y la devolución del código de salida del escáner. Esa es una revisión concreta, no una afirmación sobre todas las versiones posteriores.

El caso aquí documentado presentó el contador a cero. La issue se centra en commits omitidos a través del segundo padre: es un antecedente relacionado, no una reproducción pública exacta de aquel resultado. La aportación de este artículo es explicar la comprobación del alcance y las guardas aplicadas, sin reclamar prioridad sobre el defecto.

## La corrección: un alcance explícito por evento

Se sustituyó el adaptador por una invocación directa de Gitleaks, descargado de su distribución oficial, con versión fijada y verificación del checksum del archivo de descarga. Esa comprobación no se presenta como un recálculo independiente del binario extraído en el runner. Se adoptaron dos políticas:

| Evento | Alcance elegido |
|---|---|
| Pull request | Commits alcanzables desde el HEAD de la PR que no sean alcanzables desde su SHA base |
| Push a la rama principal o ejecución manual | Historial alcanzable desde el HEAD seleccionado |

Un rango `BASE..HEAD` expresa esa diferencia de alcanzabilidad; no es simplemente una resta de fechas ni el diff entre dos árboles. La selección puede comprobarse por separado con `git rev-list`. El historial alcanzable desde HEAD tampoco equivale a todas las ramas, etiquetas u objetos de un repositorio. [Documentación de git rev-list](https://git-scm.com/docs/git-rev-list).

Los siguientes fragmentos ilustran solo la selección del alcance, no el control completo de aceptación. Presuponen referencias SHA verificadas, objetos disponibles y una instalación previamente verificada. La sintaxis se corresponde con Gitleaks 8.24.3, la versión examinada para esta integración.

```bash
# Pull request: BASE y HEAD son los SHA verificados del evento.
range="${BASE}..${HEAD}"
git rev-list --count "$range"
gitleaks git --redact --log-opts="--diff-merges=first-parent ${range}" .
```

```bash
# Política de historial completo del HEAD seleccionado.
gitleaks git --redact --log-opts="--full-history --diff-merges=first-parent HEAD" .
```

La elección del diff respecto del primer padre permite observar lo aportado por el merge sin excluir del recorrido sus otras ramas. Puede producir hallazgos repetidos entre commits y merges; no debe confundirse con un mecanismo de deduplicación. [Formato de diffs de merge](https://git-scm.com/docs/git-log#_diff_formatting).

La sintaxis de ejecución y las opciones de informe, redacción y código de salida están en la [documentación de Gitleaks](https://github.com/gitleaks/gitleaks#usage). Las decisiones sobre eventos y alcance son políticas de esta integración, no garantías universales del producto.

Imprimir el recuento no bloquea un rango vacío. El workflow debe comprobarlo y aplicar además las guardas siguientes; copiar únicamente esos fragmentos no corrige por completo el falso verde.

## Cero hallazgos y escaneo válido necesitan comprobaciones separadas

El paso corregido rechaza:

1. Un rango sin commits enumerados por Git.
2. Hallazgos del escáner.
3. Un código de error del proceso, incluido el de una ejecución parcial notificada como error.
4. Un contador de commits procesados ausente o igual a cero.
5. Un log vacío, un informe SARIF ausente o vacío, o un fallo detectado al capturar la evidencia.

Se conservan por separado el estado de Gitleaks y el del proceso que guarda su salida. Una tubería de registro no debe convertir un fallo del escáner en éxito aparente.

La existencia y el tamaño de los archivos son guardas básicas: por sí solos no prueban que el informe sea válido ni que el alcance esté cubierto. En la validación también se inspeccionaron el SARIF y su correspondencia con la ejecución.

El contador del log se interpreta con un parser ligado a la versión fijada. Si cambia el formato y deja de poder leerse, el control falla: no sustituye un valor desconocido por un resultado satisfactorio.

Rechazar cero commits es una política conservadora. Puede bloquear rangos legítimos sin material de texto procesable; esos casos necesitan una decisión explícita. No es una regla universal de Gitleaks y tampoco convierte un contador positivo en prueba de cobertura completa.

## Los dos contadores no tienen que coincidir

`git rev-list --count` enumera objetos commit. El contador del escáner procede de su procesamiento de diffs.

En el [código de DetectGit de Gitleaks 8.24.3](https://github.com/gitleaks/gitleaks/blob/v8.24.3/detect/git.go#L18-L80), los archivos binarios y los archivos eliminados se descartan antes de incorporar el commit al contador. Borrar líneas de un archivo que sigue existiendo no es lo mismo que eliminar el archivo completo.

Por eso no se exige igualdad entre ambas cifras. Se registran las dos para detectar cambios sospechosos y facilitar el diagnóstico. La cobertura se justifica con el alcance, las opciones y las pruebas, no con una igualdad numérica.

## Cómo se validó el cambio

En la validación previa se documentaron ensayos con repositorios sintéticos:

| Caso | Resultado documentado |
|---|---|
| Historial con merge limpio | Escaneo satisfactorio, con commits procesados e informe sin hallazgos |
| Marcador ficticio con formato detectable introducido solo en la resolución de un merge | Hallazgo atribuido al merge y bloqueo del paso |
| Commit que solo elimina líneas de un archivo conservado | Ejecución válida; este ensayo no representa la eliminación completa de archivos |
| Rango vacío | Rechazo antes de lanzar el escáner |

También se documentó una ejecución sobre el historial real, sin hallazgos en el alcance seleccionado. Ese resultado no permite afirmar que nunca hubiera secretos ni que se cubran formatos o referencias excluidos de la política.

Esta redacción resume ensayos anteriores y la configuración corregida: no se han repetido las pruebas para elaborar el artículo. Los resultados de la tabla proceden de esa documentación de trabajo y no se presentan como una reproducción pública independiente. Los fragmentos de código son genéricos. Las evidencias del proyecto de origen no se publican aquí.

## Qué conservar después de cada ejecución

La evidencia útil responde a preguntas concretas:

- ¿Qué evento activó el escaneo y qué referencias se seleccionaron?
- ¿Qué versión, opciones, configuración y exclusiones se usaron?
- ¿Cuántos commits enumeró Git y cuántos reportó el escáner?
- ¿Cómo terminaron el escáner y la captura de salida?
- ¿Qué log e informe corresponden a esa ejecución?

Se conservaron un resumen de alcance, el log con redacción y el SARIF, también en ejecuciones fallidas cuando hubo material disponible. La revisión de esas copias precede a cualquier publicación: redactar coincidencias no garantiza que rutas, mensajes o metadatos carezcan de información privada.

El criterio de aceptación dejó de ser únicamente «el proceso terminó con código cero». Pasó a exigir un alcance identificado, ejecución no vacía, resultado válido y evidencia disponible. Un job verde solo resulta útil si podemos explicar qué examinó para llegar a ese resultado.
