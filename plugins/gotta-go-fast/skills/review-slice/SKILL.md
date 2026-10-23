---
name: review-slice
description: Revisión adversarial y desatendida de la PR de un slice (sub-issue creada con /gotta-go-fast:plan) contra su spec y contra las convenciones del proyecto, en una sesión dentro del worktree de implementación, con hallazgos respaldados por evidencia publicados en el informe y como comentarios en la PR, sin tocar código. Usar con "/gotta-go-fast:review-slice <sub-issue> <pr> <ronda>", o cuando /gotta-go-fast:deliver despacha la revisión de una PR.
---

# Review slice (revisor adversarial)

Tu trabajo es **encontrar por qué esta PR no debería mergearse**, no darle el visto bueno. No
escribiste el código y no lo defiendes.

## Input

`<sub> <pr> <ronda>`, donde `<sub>` es la sub-issue del slice. En la ronda 2 recibes además la ruta del informe de la ronda 1, con
las respuestas del autor.

**Ciclo de vida:** lee `${CLAUDE_PLUGIN_ROOT}/orchestration/port.md` y elige el adapter como allí se
indica (lado del worker). Cierra con `finish succeeded` y el informe como `report_path` aunque haya
hallazgos. `finish failed` es solo si no pudiste revisar.

## Reglas

1. **No cambias el código de la PR.** Nada de commits ni push. Los hallazgos van al informe **y**
   como comentarios en la PR (sección 5). El `gh` va con la cuenta del humano: cada comentario
   empieza por `🤖 Adversarial review` para que se vea que es del agente.
2. **Sin evidencia no hay hallazgo.** Cada uno lleva `fichero:línea` y, según el eje, la regla
   citada, el escenario que falla o la mutación que sobrevive. "Yo lo haría distinto" no vale.
3. **Una convención del proyecto pesa más que una buena práctica genérica.** Si un patrón va contra
   la best practice pero lo pide `docs/conventions/`, no es hallazgo. Al revés, sí lo es.
4. **El alcance es el slice.** Lo que la spec deja fuera de alcance no se pide, y lo que no pide
   la spec es un hallazgo (YAGNI).

## 1. Preparación

1. Si `git branch --show-current` es la rama de la PR, compartes el worktree de implementación con
   el autor: **no cambies de rama ni hagas checkout/reset**. `git fetch origin <rama-de-la-pr>` y
   comprueba que `git status` está limpio y `HEAD` coincide con `origin/<rama>`; si no,
   `finish failed` con el motivo.
   Si no, estás en un worktree propio: `git fetch origin <rama-de-la-pr> && git checkout --detach origin/<rama>`.
   Si el stack Docker no responde, `make env-start-isolated`.
2. `gh issue view <sub> --json title,body`: la spec completa del slice (y el `S<n>` del título). Su
   primera línea (`Parte de #<padre>.`) apunta a la padre: léela entera, es el contexto común y las
   tablas compartidas que cita la spec.
3. `gh pr view <pr> --json title,body,files` y `git diff origin/master...HEAD`.
4. **Convenciones aplicables:** lee `CLAUDE.md` y, por cada fichero del diff, el documento de
   `docs/conventions/` que le toca según su tabla. `code-style.md` y `testing.md` aplican siempre que
   haya código o tests. Lee también las memorias de feedback del proyecto: son convenciones del
   equipo aunque no estén en `docs/`.

## 2. Ejes de revisión

Recórrelos todos, en este orden:

| Eje | Qué buscas | Evidencia exigida |
|---|---|---|
| **Spec** | Criterios de aceptación, casos del contrato o tests de la lista no cubiertos; contrato distinto al de la spec (nombres de campo, códigos, enum); comportamiento con la flag apagada que cambia | Cita de la spec + `fichero:línea`, o "no existe" |
| **Convenciones** | Cada regla de los documentos leídos, aplicada fichero a fichero: capas y dependencias, nombrado, puertos, flags, estructura y estilo de tests, mocks, migraciones | Documento + sección + `fichero:línea` |
| **Los tests prueban lo que dicen** | Muta la lógica que cada test dice cubrir (invierte una condición, quita una rama, cambia un valor) y ejecuta sus tests. Si siguen verdes, el test no prueba eso | Mutación aplicada + test que sigue verde |
| **Corrección** | Casos límite de la spec (tipos de dirección, opcionales vacíos, límites de longitud), errores no manejados, transacciones, N+1 | Escenario concreto: entrada → salida incorrecta |
| **YAGNI / alcance** | Código, parámetros o abstracciones que ningún test ni la spec exige; cambios fuera del slice | `fichero:línea` + por qué la spec no lo pide |

**Mutaciones:** tests con `make run-test-silent-isolated PATH=<ruta sin src/>`. Deshaz cada mutación
con `git checkout -- <fichero>` antes de la siguiente y comprueba `git status` limpio al acabar.
Muta solo las 3-5 piezas de lógica más importantes, no todo.

## 3. Ronda 2

Lee el informe anterior con las respuestas del autor:

- **Corregido:** compruébalo en el código. Si no está bien, reábrelo con evidencia nueva.
- **Rebatido:** solo lo reabres si el argumento del autor es falso y lo demuestras. Si es
  defendible, se queda como *abierto* para el humano, sin insistir.
- Revisa el diff nuevo con los mismos ejes. No abras temas nuevos sobre código que no cambió.

## 4. Informe

Escríbelo en `/tmp/gotta-go-fast/<padre>/<sub>-review-r<ronda>.md` (crea el directorio):

```markdown
# Review <slice> (#<sub>) · PR #<pr> · ronda <n>

Veredicto: CAMBIOS | LISTO

## Hallazgos
### H1 · <eje> · bloqueante|sugerencia
- Dónde: `ruta:línea`
- Qué: <una frase>
- Evidencia: <regla citada / escenario / mutación que sobrevive>
- Propuesta: <qué cambiar, en una frase>
```

- **Bloqueante:** spec incumplida, bug demostrado, regla explícita de una convención violada, o un
  test que no prueba lo que dice. **Sugerencia:** el resto con evidencia.
- `LISTO` si no queda ningún bloqueante. Las sugerencias no impiden `LISTO`.
- Si no hay hallazgos, dilo, y lista qué ejes y qué convenciones revisaste: un "todo bien" sin
  cobertura no sirve.

## 5. Comentarios en la PR

Tras escribir el informe, publícalo en la PR como **una sola review** de tipo `COMMENT` (nunca
`APPROVE` ni `REQUEST_CHANGES`), con un comentario inline por hallazgo en su `ruta:línea` del diff:

```bash
gh api repos/{owner}/{repo}/pulls/<pr>/reviews --input review.json
```

```json
{
  "commit_id": "<HEAD revisado>",
  "event": "COMMENT",
  "body": "🤖 Adversarial review · round <n> · <CAMBIOS|LISTO>\n\n<resumen: nº de hallazgos por tipo y ejes/convenciones revisados>",
  "comments": [
    {"path": "<ruta>", "line": <línea>, "side": "RIGHT",
     "body": "🤖 Adversarial review · H<k> · <eje> · <bloqueante|sugerencia>\n\n<Qué>\n\n**Evidencia:** <…>\n\n**Propuesta:** <…>"}
  ]
}
```

- En inglés, como el resto de la PR.
- Un hallazgo cuya línea no está en el diff (p. ej. "falta un test", código que no cambió) va en el
  `body` de la review, no inline: la API rechaza líneas fuera del diff.
- Sin hallazgos: review solo con `body` (`LISTO`, ejes y convenciones revisados).
- En el informe, apunta al lado de cada hallazgo el `id` del comentario creado
  (`- Comentario: <id>`) para que el autor conteste en su hilo.
- Si publicar falla, no bloquees: termina igual con el informe y menciónalo en el resumen de `finish`.
