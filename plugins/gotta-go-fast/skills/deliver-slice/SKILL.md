---
name: deliver-slice
description: Implementa de forma desatendida UN slice, representado por una sub-issue de GitHub creada con /gotta-go-fast:plan, de la rama a la PR con CI en verde, o atiende la review (humana o adversarial) de su PR. Usar con "/gotta-go-fast:deliver-slice <sub-issue>", "/gotta-go-fast:deliver-slice <sub-issue> --review <pr>", "/gotta-go-fast:deliver-slice <sub-issue> --findings <pr> <informe>", o cuando un coordinador (/gotta-go-fast:deliver) despacha una sub-issue a un worker.
---

# Deliver slice (worker desatendido)

Lleva un slice (una sub-issue) a una PR lista para revisión humana, sin preguntar nada a nadie salvo
bloqueo real.

## Input

- `<sub>` → modo **implementar**. `<sub>` es el número de la sub-issue del slice.
- `<sub> --review <pr>` → modo **atender review** humana de una PR ya abierta.
- `<sub> --findings <pr> <informe>` → modo **atender review adversarial**
  (`/gotta-go-fast:review-slice`).

**Ciclo de vida:** lee `${CLAUDE_PLUGIN_ROOT}/orchestration/port.md` y elige el adapter como allí se
indica (lado del worker). Terminas siempre con un único `finish` y preguntas con `ask`, tal como
los implementa ese adapter.

## Reglas de desatendido

1. **Nadie contesta.** Donde una skill que uses pida confirmación (`START`, "¿siguiente test?",
   "¿creamos la PR?"…), decide tú con estas reglas y sigue. No abras preguntas interactivas.
2. **Bloqueo = parar, no adivinar.** Si la spec es ambigua o contradictoria, si el slice exige algo
   fuera de su alcance, o si algo falla y no es trivial: deja un comentario en la sub-issue con la
   pregunta concreta (`gh issue comment <sub> --body-file -`, en español, empezando por
   `🤖 Bloqueado:`), deja el worktree limpio y termina con `finish failed`. Si la pregunta es
   corta, antes puedes probar `ask`; si devuelve `unsupported`, bloqueo.
3. **Solo tu slice.** Nada de refactors de paso ni de adelantar slices siguientes.
4. Nunca `git stash`, nunca `--no-verify`, nunca push a `master`.

## Modo implementar

### 1. Contexto

1. `gh issue view <sub> --json title,body,state,assignees,labels`. El body es la spec completa del
   slice (objetivo, alcance, fuera de alcance, contrato, lista de tests, "hecho cuando"). Del título,
   el slice (`S<n>`).
2. **Issue padre**: la de la primera línea del body (`Parte de #<padre>.`). Léela entera
   (`gh issue view <padre> --json title,body`): es el contexto común y las tablas compartidas que la
   spec cita. Si la spec contradice a la padre → bloqueo.
3. Comprueba que la sub-issue sigue siendo tuya: abierta, y asignada a `@me` con la label `deliver`
   (si te lanzó el coordinador) o sin asignar (si te lanzó el humano; entonces asígnatela con
   `gh issue edit <sub> --add-assignee @me --add-label deliver`). Si está asignada a otra persona o
   hay una PR abierta ajena que la cierra
   (`gh pr list --state open --json number,author,closingIssuesReferences`) → bloqueo (regla 2),
   no dupliques.
4. Lee `CLAUDE.md` y, según las capas que vayas a tocar, los ficheros de `docs/conventions/` de su
   tabla. Las convenciones del proyecto mandan.
5. Lee las memorias del proyecto que apliquen (feedback de tests, flags, Hermes, etc.).

### 2. Rama

```bash
git fetch origin master
git checkout -b feature-<sub> origin/master   # p.ej. feature-123
```

Si la rama ya existe y otro worktree la tiene (`git worktree list`), el slice está reclamado →
bloqueo. Comandos `make`: si la ruta contiene `workspaces/` o es un worktree, siempre `-isolated`.
Si el stack Docker no responde, `make env-start-isolated` una vez antes de seguir.

### 3. TDD

Sigue `/superpowers:test-driven-development` (Rojo → Verde → Refactor) con estas reglas del
proyecto encima, que prevalecen sobre la skill:

- Antes del primer test, lee `CLAUDE.md` y las convenciones de `docs/conventions/` de cada capa que
  vayas a tocar.
- Un test cada vez, tomado de la "Lista de tests" de la spec en su orden.
- En Verde, YAGNI estricto: hardcodear el valor esperado hasta que otro test exija la lógica real.
- Un commit por Verde y otro por Refactor.
- Completado cuando todos los tests de la lista y el "hecho cuando" estén cubiertos.
- Sin pausas humanas: no hay `START`, la spec de la sub-issue es el resumen del slice; si la rama recién
  creada trae commits o cambios → bloqueo. El plan vive en GitHub: no tocar `.aiplans/`. PR en el
  paso 5, sin retrospectiva ni report.

La "Lista de tests" de la spec es el punto de partida, no un techo ni un contrato literal: aplica
las memorias (extender tests existentes antes que crear nuevos, test suelto para la rama de una
flag, no tests de dominio aislados, no asserts de logs ni de orden de llamadas).

Commits en inglés, conventional commits, y **siempre** con el trailer:

```
Co-Authored-By: Claude <noreply@anthropic.com>
```

**Feature flag:** si la spec dice que va detrás de flag, o el slice toca una Action con consumidor
real en producción, va con flag (lee `docs/conventions/feature-flags.md`).

### 4. Verificación

1. `make fix-linting-isolated` (incluye mypy). Si deja cambios, commitéalos.
2. Tests de los módulos tocados: `make run-test-silent-isolated PATH=<ruta sin src/>`. No lances la
   suite completa: la cubre la CI.
3. Sin verde no hay PR. Si no consigues verde → bloqueo.

No te autorrevises: la revisión la hace después un revisor adversarial con contexto limpio
(`/gotta-go-fast:review-slice`).

### 5. PR

`gh pr create --body-file -` contra `master`, sin preguntar.

- Título en inglés: `<qué hace>`.
- Body con `.github/pull_request_template.md` si existe. En *Why?*: `Slice <slice> of #<padre> (...)` y una
  línea `Closes #<sub>`, para que GitHub cierre la sub-issue al mergear. Nunca
  `Closes`/`Fixes`/`Resolves #<padre>`.
- Termina el body con `Generated with Claude Code`.

### 6. CI

Espera a la CI de tu último push con `gh pr checks <pr> --watch`. **No uses `/babysit-pr` ni
actualices la rama con master**: si la PR queda behind no es un fallo, y cada actualización
relanza la CI y retrasa el cierre. La rama se pone al día después, al mergear. Si queda en rojo y
no es trivial → bloqueo, citando el check y el fallo.

### 7. Cierre

No cierres la sub-issue: la cierra GitHub al mergear la PR por el `Closes #<sub>`. `finish
succeeded` con el resumen `<slice> (#<sub>) → PR #NNN, CI verde` (o `finish failed` con el
bloqueo).

## Modo atender review

1. `gh pr view <pr> --json headRefName,reviews,comments` y los comentarios de línea sin resolver
   (`gh api repos/{owner}/{repo}/pulls/<pr>/comments`).
2. Si no estás en la rama de la PR, `git fetch origin <rama> && git checkout <rama>`. Siempre
   `git reset --hard origin/<rama>` antes de tocar nada: el remoto reescribe la rama en cada push.
3. Aplica `superpowers:receiving-code-review`: valida cada comentario técnicamente antes de
   cambiar nada. Si no estás de acuerdo, contesta en el hilo con el porqué en vez de cambiar.
4. Cambios con TDD si alteran comportamiento. Después, verificación (paso 4 de implementar),
   commit y `git push --force-with-lease origin HEAD:<rama>`.
5. Contesta cada hilo atendido con qué cambió (en español, breve), espera la CI como en el paso 6
   de implementar y `finish` como en el paso 7.

## Modo atender review adversarial

1. Rama de la PR y `git reset --hard origin/<rama>`, como en el modo anterior.
2. Lee el informe. Por cada hallazgo, aplica `superpowers:receiving-code-review`: compruébalo
   tú (reproduce el escenario o la mutación) antes de aceptarlo. Un hallazgo de convenciones se
   valida leyendo la regla citada, no fiándote de la cita.
3. Decide **Corregido** (cambio con TDD si altera comportamiento) o **Rebatido** (con el porqué:
   fuera de alcance según la spec, la convención dice otra cosa, el escenario no se da…).
4. Debajo de cada hallazgo del informe, añade `- Respuesta: Corregido (<sha>) | Rebatido: <porqué>`.
   Si el hallazgo trae `- Comentario: <id>`, contesta también en su hilo de la PR
   (`gh api repos/{owner}/{repo}/pulls/<pr>/comments/<id>/replies -f body=...`), empezando por
   `🤖 Adversarial review` y en inglés: `Fixed in <sha>` o `Rebutted: <porqué>`.
5. Verificación (paso 4 de implementar), commit y `git push --force-with-lease origin HEAD:<rama>`.
6. Actualiza el body de la PR (`gh pr edit <pr> --body-file`) con esta sección antes del separador
   `---` del template, acumulando las rondas:

   ```markdown
   ### 🤖 Adversarial review
   - Fixed: <hallazgo en una línea> (round N)
   - Rebutted: <hallazgo> → <porqué en una línea>
   - Open, your call: <lo que el revisor mantuvo tras tu respuesta>
   ```

   En inglés, como el resto del body. Si la review vino `LISTO` sin hallazgos:
   `- No findings (spec, conventions, test mutation, correctness, scope).`
7. Espera la CI como en el paso 6 de implementar. `finish` con el informe con tus respuestas como
   `report_path`.
