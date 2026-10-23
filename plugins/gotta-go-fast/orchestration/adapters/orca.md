Coordinador aplica si: `TERM_PROGRAM=Orca` y `orca status --json` responde.
Worker aplica si: el prompt trae un preámbulo de Orca con Task ID y Dispatch ID.

# Adapter: Orca

**Antes de nada:** `orca skills get orchestration` y léela entera. Es la guía versionada con la CLI
instalada y manda sobre este adapter si discrepan (`~/.claude/skills/orchestration` es solo un stub
que remite a ese comando). En el coordinador actúas como **Coordinator** de esa guía; en un worker,
aplica su sección "Worker obligations".

## Lado del coordinador

Al empezar: `orca status --json` y
`orca orchestration run-create --objective "Deliver #<padre>" --json`. Si este terminal ya está
ligado a un Run de esta issue padre (`orca orchestration run-current --json`), reutilízalo.

| Verbo | Comando |
|---|---|
| `start_worker` (`placement: new`) | `orca orchestration worker-start --spec "<prompt>" --task-title "#<sub> <slice>" --worktree new-top-level --name feature-<sub> --base-branch <base> --setup run --agent <agent> --json` |
| `start_worker` (`placement: branch:<rama>`) | `orca orchestration worker-start --spec "<prompt>" --task-title "#<sub> <slice> review" --worktree branch:<rama> --agent <agent> --json` |
| `start_worker` (`placement: shared:<worker_id>`) | `orca orchestration worker-start --spec "<prompt>" --task-title "#<sub> <slice> review r<n>" --worktree id:<worktree-id> --agent <agent> --json`: sesión nueva en el **mismo** workspace (sin `--setup`, el stack ya está). `<worktree-id>` es el `id` completo (`<repo-id>::<path>`) del efecto `kind: worktree` del recibo de `worker-start` de ese worker, o la fila de `orca worktree list --json` cuya rama es `feature-<sub>` |
| `wait_event` | `orca orchestration check --wait --types "worker_done,escalation,question" --timeout-ms 900000 --json`; procesa el lote y confirma con `--ack <delivery_id>` en la siguiente llamada. `worker_done` → `done` (con su `--outcome` y `--report-path`); `question` y `escalation` → `question` |
| `send` (respuesta a una pregunta) | `orca orchestration reply --id <message_id> --body "<texto>" --json` |
| `send` (instrucción nueva no pedida, p. ej. informe de review) | `orca orchestration send --to dispatch:<id>` **no basta**: solo encola, y un worker bloqueado en un comando largo (`gh pr checks --watch`) no mira el buzón. Escribe el texto en su terminal de agente para que entre como mensaje de usuario a mitad de turno: `orca terminal send --terminal <agentTerminalHandle> --text "<texto>" --enter --wait-submit 10 --json` (`agentTerminalHandle` sale de `worker-show --dispatch <id>`). Si el recibo no confirma que el turno se observó, devuelve `not_delivered` |
| `status` | `orca orchestration worker-list --json` (usa `projection.liveness`) |
| `release` | `orca orchestration worker-release --dispatch <dispatch_id> --json` |

`<agent>` es `claude` salvo que el núcleo pida otro (p. ej. `codex` para diversificar el revisor).

**Spec del worker.** La guía exige specs autocontenidas. El `<prompt>` de `--spec` lleva el slash
command en la primera línea y debajo los cinco campos del contrato:

```
/gotta-go-fast:deliver-slice <sub> [--review <pr> | --findings <pr> <informe>]

Target: <owner/repo>, slice <slice> = sub-issue #<sub> de #<padre> (spec completa en la sub-issue).
Change: <objetivo del slice en una frase>.
Constraints: convenciones de CLAUDE.md y docs/conventions/; solo el alcance del slice; make *-isolated.
Ownership: solo la rama feature-<sub> y su worktree; no tocar otras sub-issues ni la padre.
Observable acceptance: PR abierta con "Closes #<sub>" y CI en verde, o comentario de bloqueo en la sub-issue.
```

**Referencias de la guía que aplican aquí** (`orca skills get orchestration --reference references/<file>.md`):
`placement-and-remote.md` antes del primer `worker-start` con worktree nuevo, y
`coordinator-loop.md` antes del primer worker de review.

Notas:

- Si `worker-start` sale con código ≠ 0, **no relances**: lee `failedStage` y `residualResources`
  del recibo y avisa al humano.
- `wait_event` y el `Monitor` de GitHub conviven: lanza el `check --wait` como `Monitor` propio
  (filtrando `_keepalive` con `jq`) para que ambos lleguen como notificaciones.
- Política de setup del repo: si es `start-immediately`, el worker puede arrancar antes de que
  `make env-start-isolated` termine. `/gotta-go-fast:deliver-slice` lo cubre relanzando el stack si no
  responde, pero lo correcto es `wait-for-setup`.
- El worktree no se borra al liberar el worker: lo necesita un posible `--review`. Se archiva a
  mano tras el merge.

## Lado del worker

El preámbulo de Orca manda sobre el ciclo de vida (arranque y cierre), según "Worker obligations".

| Verbo | Comando |
|---|---|
| `finish` | `worker_done` como indica la guía, con `--outcome succeeded\|failed`, el resumen y `--report-path <ruta>` si hay informe |
| `ask` | `orchestration ask` con la pregunta |
