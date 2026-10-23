---
name: deliver
description: Coordina la implementación desatendida de una user story planificada con /gotta-go-fast:plan (issue padre + una sub-issue por slice con dependencias "blocked by"): construye el grafo desde GitHub, lanza un worker por sub-issue lista (/gotta-go-fast:deliver-slice), pasa cada PR por una revisión adversarial (/gotta-go-fast:review-slice) y reacciona a merges y reviews del humano hasta acabar. Usar con "/gotta-go-fast:deliver <issue-padre>", "entrega la US #NNN", "lanza los slices de la issue NNN".
model: sonnet
---

# Deliver (coordinador)

El humano solo revisa y mergea PRs. Esta sesión no escribe código: reparte sub-issues a workers,
despacha la revisión y reacciona a lo que hace el humano en GitHub.

**Fuente de verdad:** la issue padre, sus sub-issues (estado, asignación, labels, *blocked by*) y
las PRs que las cierran. El coordinador no guarda estado propio: si se reinicia,
`/gotta-go-fast:deliver <padre>` lo reconstruye todo.

**Formato de entrada:** el de `/gotta-go-fast:plan`. Si la issue no tiene sub-issues, para y sugiere
planificarla con `/gotta-go-fast:plan`. No la planifiques tú.

## 1. Elegir adapter de orquestación

Lee [`port.md`](../../orchestration/port.md) y elige el adapter como allí se indica (lado del coordinador). Desde
aquí solo se usan los verbos del puerto.

## 2. Construir el grafo

Todo sale de la API, sin parsear markdown:

```bash
R=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh api repos/$R/issues/<padre>/sub_issues --paginate \
  --jq '.[] | {number, title, state, assignees: [.assignees[].login], labels: [.labels[].name]}'
gh api repos/$R/issues/<sub>/dependencies/blocked_by --jq '.[] | {number, state}'   # por sub-issue
gh pr list --state open --json number,author,headRefName,closingIssuesReferences      # PRs que cierran cada sub-issue
```

Slice = `S<n>` del título de la sub-issue; se usa solo para mensajes. La identidad es el número
de la sub-issue.

**Reclamo de la automatización:** label `deliver` en la sub-issue + asignada a `@me`. Asignada a
`@me` **sin** la label `deliver` significa que el humano la lleva a mano.

Estado de cada sub-issue:

- **hecha**: cerrada.
- **manual**: label `manual`.
- **de otra persona**: asignada a alguien sin la label `deliver`, o con una PR abierta ajena que la
  cierra. No se toca.
- **en vuelo**: label `deliver` y worker vivo o PR abierta de la automatización.
- **apilable**: todas sus *blocked by* abiertas están **en vuelo** (con rama), y depende solo de su
  *código*, no de decisiones que esas aún tengan que resolver. Apilar **solo con OK explícito del
  humano** para esa US: si la review es el cuello de botella, más PRs abiertas no aceleran nada. Sin
  OK, trátala como **bloqueada**.
- **bloqueada**: alguna *blocked by* abierta y no apilable.
- **lista**: abierta, sin asignar, sin `manual` y con todas sus *blocked by* cerradas.

Primero lo secuencial y después en paralelo sale solo del grafo: al cerrarse una sub-issue, todas
sus dependientes que queden listas se lanzan a la vez, con el único límite de la
[capacidad](#capacidad).

Presenta la tabla al humano (slice, sub-issue, estado, motivo, blocked by, PR) y la lista de
lanzamientos propuestos. **Espera confirmación**: es la única pausa interactiva del flujo. El humano
puede excluir sub-issues; las dependencias se corrigen en GitHub (o con `/gotta-go-fast:plan`), no aquí.

### Capacidad

No hay tope fijo de workers: antes de **cada** `start_worker`, mide la máquina y lanza solo si se
cumplen todas:

| Señal (macOS) | Comando | Umbral para lanzar |
|---|---|---|
| Carga de CPU | `sysctl -n vm.loadavg` (1 min) vs `sysctl -n hw.ncpu` | carga < 0,6 × núcleos |
| Memoria libre | `memory_pressure \| tail -1` | ≥ 30 % |
| Docker (solo repos con stack Docker) | `docker stats --no-stream --format '{{.MemPerc}}'` sumado | < 70 % de la VM de Docker |

Además, nunca más de **4 workers vivos** a la vez: es el techo de seguridad, no el objetivo. Si un
lanzamiento no cabe, déjalo en cola y reevalúa en el siguiente evento (worker terminado, merge) o
cuando el humano lo pida; menciónalo en el aviso al humano.

## 3. Lanzar

Por cada sub-issue lista y confirmada:

1. Reclámala **antes** de lanzar, para que otro coordinador o una persona la vea cogida:
   `gh issue edit <sub> --add-label deliver --add-assignee @me` (crea la label si no existe).
2. Comprueba que no hay ya un worker vivo ni una rama `feature-<sub>` y que cabe por
   [capacidad](#capacidad).
3. `start_worker` con el prompt `/gotta-go-fast:deliver-slice <sub>` y `placement: new`.

Sub-issue **apilable**: `start_worker` con `base: origin/feature-<bloqueante>`, y la
spec del worker dice que la PR va contra `feature-<bloqueante>` y que, cuando esa se mergee, rebase
sobre `master` y cambie la base de la PR a `master`. Al mergear la bloqueante, avisa al worker vivo
o, si ya terminó, lanza uno en modo `--review` para el rebase.

## 4. Bucle de eventos

Arma **un único** `Monitor` persistente que vigile GitHub cada 60 s y emita solo los cambios:

- `SUB_CLOSED <sub>`: una sub-issue de la padre pasa a cerrada (lo normal: su PR se mergeó con
  `Closes #<sub>`);
- `PR_OPENED <pr>`: aparece una PR abierta de la automatización (rama `feature-<sub>`);
- `PR_MERGED <pr>`: PR de la automatización mergeada (por si no cerraba su sub-issue);
- `PR_REVIEW <pr>`: aparecen reviews `CHANGES_REQUESTED` o comentarios humanos nuevos (no del bot
  ni del propio agente) desde el último evento;
- `PR_CLOSED <pr>`: cerrada sin mergear;
- `GRAPH_CHANGED`: cambian las sub-issues de la padre o sus *blocked by* (el humano replanifica).

Además, los eventos de workers del adapter (`wait_event`). Reacciona así:

| Evento | Acción |
|---|---|
| `PR_OPENED` | Lanza el revisor de la ronda 1 (paso 4.1) ya, sin esperar a la CI ni a que el autor termine. **Aún no avises al humano** |
| Worker terminado con PR | Libera el worker. Si el revisor de la ronda 1 aún no se lanzó (p. ej. el `PR_OPENED` se perdió), lánzalo ahora |
| Revisor terminado (ronda n) | El autor aplica los hallazgos que tengan sentido y rebate el resto, sin que el humano toque nada: si el autor sigue vivo (p. ej. esperando la CI), `send` con el informe para que pase ya al modo `--findings` y haga un único `finish` (si `send` devuelve `not_delivered`, avisa al humano); si ya terminó, lanza el autor en modo `--findings <pr> <informe>` con `placement: branch:feature-<sub>`. En ambos casos responde los hallazgos y vuelca la sección *Adversarial review* en la PR (también si vino `LISTO`) |
| Autor terminado tras `--findings` | Si el informe era `CAMBIOS` y n = 1, lanza el revisor de la ronda 2 con el informe respondido. Si no, avisa al humano: `<slice> (#<sub>) → PR #NNN lista para revisar (adversarial: N corregidos, M rebatidos, K abiertos)` |
| Worker terminado en fallo / bloqueo | Avisa al humano con el motivo y el enlace al comentario en la sub-issue. Quita la label `deliver` y la asignación, y no relances |
| Pregunta de un worker | Si se responde con la spec o las convenciones, contesta tú. Si no, trasládala al humano y contesta con su respuesta |
| `PR_REVIEW` | `start_worker` con `placement: branch:feature-<sub>` y `/gotta-go-fast:deliver-slice <sub> --review <pr>` |
| `PR_MERGED` sin `SUB_CLOSED` | Cierra la sub-issue según `/gotta-go-fast:plan` (paso 5.1, `Completado en PR #NNN.`) |
| `SUB_CLOSED` | Rebase de los apilados sobre ella (paso 3) y vuelve a evaluar el grafo: lanza **todas** las sub-issues que queden listas, según la capacidad. Si ya están todas cerradas, sigue el paso 5.2 de `/gotta-go-fast:plan` (cerrar la padre) |
| `PR_CLOSED` | Avisa al humano y deja la sub-issue fuera hasta que diga algo |
| `GRAPH_CHANGED` | Rehaz el paso 2. Las nuevas listas se lanzan sin volver a pedir confirmación solo si el humano no excluyó nada en la confirmación inicial; si no, pregunta |

Nunca lances dos workers que escriban en la misma rama. El revisor de una ronda sí puede correr
a la vez que el autor que espera la CI (el revisor no escribe en la rama), pero el autor en modo
`--findings` nunca se lanza mientras siga vivo el autor anterior.

### 4.1 Revisión adversarial

Antes de que el humano vea una PR, pasa por `/gotta-go-fast:review-slice`, como mucho en **2 rondas**:
lo que siga abierto tras la respuesta del autor a la ronda 2 lo decide el humano. La ronda 1 se
lanza en cuanto la PR está abierta, pase o no la CI: el revisor revisa el HEAD de la rama al
empezar.

- `start_worker` con el prompt `/gotta-go-fast:review-slice <sub> <pr> <ronda>` y, en la ronda 2, la
  ruta del informe de la ronda 1 con las respuestas del autor, y `placement: shared:<autor>`: una
  **sesión nueva dentro del worktree de implementación** (misma copia de trabajo y mismo stack Docker).
  Si el `placement` efectivo es `shared`, mientras el revisor está vivo ningún autor escribe en ese
  worktree: si la CI del autor se pone roja con el revisor vivo, el fix espera a que el revisor
  termine. Si el adapter lo colocó en un worktree propio, esa restricción no aplica. Cuenta para la
  [capacidad](#capacidad).
- Informes en `/tmp/gotta-go-fast/<padre>/<sub>-review-r<n>.md`: el adapter los pasa entre
  workers por ruta.
- Si el revisor termina en fallo (no pudo revisar), no bloquees la sub-issue: avisa al humano de que
  la PR va sin revisión adversarial y por qué.
- Opcional: lanzar el revisor con otro `agent` para diversificar, si el adapter lo soporta. Por
  defecto, Claude.

## 5. Fin

Cuando no queda nada listo, en vuelo ni bloqueado por algo en vuelo, para el Monitor y resume en
una línea por sub-issue: `S1 ✔ #NNN · S3 ✔ #NNN · S6 ✋ manual pendiente`.
