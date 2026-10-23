# gotta-go-fast

Planifica una user story en sub-issues de GitHub y la entrega de forma desatendida: el humano solo
revisa y mergea PRs.

## Skills

| Skill | Qué hace |
| :--- | :--- |
| [plan](skills/plan/SKILL.md) | Trocea una user story en slices verticales desplegables y la publica como issue padre + una sub-issue por slice con dependencias nativas *blocked by*. También opera sobre un plan ya publicado |
| [deliver](skills/deliver/SKILL.md) | Coordinador: construye el grafo de sub-issues desde GitHub, lanza un worker por sub-issue lista, pasa cada PR por revisión adversarial y reacciona a merges y reviews hasta acabar |
| [deliver-slice](skills/deliver-slice/SKILL.md) | Worker: lleva un slice de la rama a la PR con CI en verde siguiendo TDD, o atiende la review (humana o adversarial) de su PR |
| [review-slice](skills/review-slice/SKILL.md) | Revisor adversarial: revisa la PR contra la spec del slice y las convenciones del proyecto (incluidas mutaciones de los tests) y publica los hallazgos como review `COMMENT` |

## Orquestación

El núcleo solo habla con un puerto ([`orchestration/port.md`](orchestration/port.md)). Hoy hay un
adapter, [Orca](orchestration/adapters/orca.md), que lanza los workers en worktrees gestionados.

## Requisitos

- Orca con su CLI `orca`.
- `gh` autenticado con acceso al repo.
- Skills `superpowers` (TDD y recepción de code review), que los workers usan durante la entrega.
- Un `CLAUDE.md` en el proyecto con sus comandos y convenciones (`docs/conventions/`).
