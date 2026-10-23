# Puerto de orquestación

Contrato mínimo entre el núcleo (`/gotta-go-fast:deliver`, `/gotta-go-fast:deliver-slice`,
`/gotta-go-fast:review-slice`) y el entorno que lanza los agentes. El núcleo solo usa estos verbos y no
nombra ningún entorno concreto; nada de GitHub pasa por aquí.

## Elegir adapter

Cada fichero de [`adapters/`](adapters/) abre con dos líneas:

```
Coordinador aplica si: <condición>
Worker aplica si: <condición>
```

Lee solo esas cabeceras (`head -2 ${CLAUDE_PLUGIN_ROOT}/orchestration/adapters/*.md`), comprueba la
condición del lado que te toca y lee **entero solo** el adapter que aplique. Si no aplica ninguno,
para y díselo al humano: no hay entorno donde lanzar agentes.

## Lado del coordinador

| Verbo | Entrada | Garantía |
|---|---|---|
| `start_worker` | prompt, `placement`, `base` (opcional, por defecto `origin/master`), `agent` (opcional, por defecto Claude) | Un agente nuevo ejecutando el prompt, con el stack Docker listo. Devuelve un `worker_id` y el `placement` efectivo |
| `wait_event` | timeout | Devuelve el siguiente evento de un worker: `done(worker_id, ok, resumen, report_path?)`, `question(worker_id, texto)` o nada (timeout, que no es fallo) |
| `send` | worker_id, texto | Entrega el texto al worker vivo, sea respuesta a una pregunta o instrucción nueva, aunque esté ocupado en un comando largo. Si no puede garantizarlo, devuelve `not_delivered` |
| `status` | — | Lista de workers con `worker_id`, slice y viveza (`live` / `exited` / `unknown`) |
| `release` | worker_id | Cierra el agente terminado. El worktree se conserva hasta que la PR se mergea |

`placement`:

- `new`: worktree propio nuevo con la rama `feature-<sub>` creada desde `base`.
- `branch:<rama>`: worktree propio sobre una rama que ya existe.
- `shared:<worker_id>`: sesión nueva en el **mismo** worktree (y stack) que otro worker. Si el adapter
  no lo soporta, coloca el worker como `branch:<rama de ese worker>` y lo dice en el `placement`
  efectivo.

Si el adapter no soporta el `agent` pedido, usa Claude y lo dice en el resultado.

## Lado del worker

| Verbo | Entrada | Garantía |
|---|---|---|
| `finish` | outcome (`succeeded` / `failed`), resumen en una línea, `report_path` (opcional) | El coordinador recibe `done` con esos datos. Se llama **una sola vez**, como último paso |
| `ask` | texto corto | Devuelve la respuesta del coordinador, o `unsupported` si el adapter no admite preguntas: entonces el worker lo resuelve como bloqueo (comentario en la sub-issue + `finish failed`) |

Si el adapter trae obligaciones de ciclo de vida propias (preámbulos, guías del entorno), mandan
sobre la skill del worker en lo que toca a arrancar y terminar, no en el trabajo del slice.

## Reglas comunes

- `unknown` no es `exited`: nunca relanzar ni liberar sin prueba positiva de que el worker terminó.
- Si un adapter no tiene un evento `done` nativo, lo deriva (p. ej., agente parado + PR abierta en
  la rama del slice) y lo documenta.
- Los eventos nativos del entorno que no estén en este contrato los traduce el adapter a `done` o
  `question`; el núcleo no los ve.
- Las capacidades extra de un entorno (puertas, DAG de tareas, buzones) no se usan desde el
  núcleo: las dependencias y los merges los resuelve el núcleo contra GitHub.
