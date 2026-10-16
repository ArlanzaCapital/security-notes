# Remediación de una dependencia vulnerable en una cadena de herramientas de desarrollo

[English](braces-development-toolchain-remediation.en.md) | **Español**

## Estado del caso

La remediación está **integrada en la rama principal**. Antes de integrarla:
- superó validaciones en Windows y en Linux;
- pasó una revisión independiente, con reservas documentadas;
- su CI completa terminó en verde.

Tras la integración, la CI de la rama principal también terminó en verde.

Este documento no describe un descubrimiento original ni una corrección del paquete upstream. La dependencia vulnerable se ha **retirado del árbol**; no se ha reparado.

## Vulnerabilidad conocida

[GHSA-vfj7-8cjw-p6xm / CVE-2026-93687](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm) documenta un agotamiento de pila en `braces` mediante patrones de llaves profundamente anidados. Se publicó el 18 de septiembre de 2026 y se revisó el 2 de octubre. Consultado el 17 de octubre de 2026, el advisory sigue indicando como afectadas las versiones hasta la 3.0.3 y no lista ninguna versión parcheada.

## Cómo apareció el problema

Una auditoría de dependencias bloqueó la CI por esta cadena de herramientas de desarrollo:

```text
eslint-config-next → @next/eslint-plugin-next → fast-glob → micromatch → braces
```

Ese bloqueo era el comportamiento esperado de la política de seguridad. La mera presencia de la dependencia no demostraba que existiera una vía de explotación remota en la aplicación.

## Estrategia de remediación

Se sustituyó localmente el uso concreto que el plugin de Next hace de `fast-glob` por un adaptador sobre `tinyglobby`. El adaptador solo se aplica a esa dependencia del plugin y está fijado a la versión del plugin que se validó. La cadena vulnerable desaparece del árbol instalado y el control de seguridad de la CI se mantiene.

El alcance de la sustitución es limitado: no implementa toda la API ni toda la sintaxis de `fast-glob`. Su contrato admite un subconjunto explícito, rechaza las entradas no soportadas y limita la longitud, la profundidad y el trabajo de expansión de los patrones.

## Evidencias

- **Equivalencia funcional:** se compararon los resultados con la implementación original en casos representativos y se comprobó el comportamiento de la regla de lint que consume el adaptador.
- **Límites:** se probaron los límites y los patrones inválidos; los casos de trabajo excesivo se ejecutan aislados en procesos con tiempo máximo.
- **Instalación:** reproducible y con la resolución del adaptador verificada en Windows y en Linux.
- **Calidad:** lint, comprobación de tipos, pruebas unitarias y de integración favorables, tanto en la CI de la propuesta como en la de la rama principal tras integrarla.
- **Auditoría y build:** la auditoría de dependencias no dio avisos y el build de Linux fue correcto. Este último se validó en local, porque el workflow no compila.
- **Revisión independiente:** concluyó que el cambio estaba preparado para integrarse, con reservas que no exigían tocar el código: el contrato acotado se acepta de forma expresa y el adaptador queda bajo seguimiento.

Estas comprobaciones respaldan el uso probado. No demuestran una equivalencia universal ni la ausencia de otros defectos.

## Un hallazgo colateral: el escaneo de secretos

Al revisar la CI posterior a la integración se observó que el escáner de secretos había informado de «0 commits» analizados, aunque el control terminó en verde. El problema de alcance y su corrección se explican en [Gitleaks en verde, cero commits analizados](https://github.com/ArlanzaCapital/security-notes/blob/main/writeups/gitleaks-zero-commits-false-green.md). Ese artículo distingue nuestra experiencia de un antecedente público relacionado con la cobertura de commits de merge.

## Mantenimiento y aprendizaje

El adaptador es código propio que hay que mantener, con un responsable asignado. Cada vez que se actualicen el plugin, el motor de glob o Node hay que hacer tres cosas:
- comprobar si el adaptador se sigue usando;
- comprobar qué cadena de dependencias resulta;
- repetir las pruebas pertinentes.

No se debe dar por supuesto el resultado.

Hay que retirarlo cuando exista una alternativa oficial compatible. Al hacerlo, se conservarán las comprobaciones que siguen protegiendo el comportamiento y se eliminarán solo las que prueban el propio adaptador.

El aprendizaje principal es que sustituir una dependencia exige comprobar el comportamiento, la instalación, la procedencia y el coste de mantenimiento futuro. Un resultado limpio de la auditoría no basta por sí solo, y tampoco basta un control de seguridad en verde si no se verifica qué ha analizado de verdad.
