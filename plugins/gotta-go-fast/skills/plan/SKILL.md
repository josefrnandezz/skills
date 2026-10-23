---
name: plan
description: Planifica una user story o un cambio en slices verticales desplegables y los publica en GitHub como una issue padre y una sub-issue por slice, con dependencias nativas "blocked by", listos para /gotta-go-fast:deliver. También opera sobre un plan ya publicado (consultar estado, marcar un slice completado, replanificar, cerrar la padre). Usar con "/gotta-go-fast:plan", "planifica esta US", "trocea esto en slices", "sube el plan a sub-issues", "marcar slice en sub-issue", o cuando se planifique para /gotta-go-fast:deliver.
---

# Plan en sub-issues

Convierte una user story en un plan de slices y lo publica en GitHub. **La issue padre y sus
sub-issues son la fuente de verdad**: no hay fichero de plan local, el estado de cada slice es el
de su sub-issue y el progreso lo muestra GitHub en la padre.

Cada sub-issue es la **única** entrada que recibe el worker de `/gotta-go-fast:deliver-slice` y la vara con
la que mide `/gotta-go-fast:review-slice`. Si un slice no se entiende leyendo su sub-issue y la padre, el
plan está mal.

## Modos

- **Planificar**: el humano da una user story, una issue, un documento o lo describe en la
  conversación. Pasos 1 a 4.
- **Operar sobre un plan publicado**: el humano da el número de la issue padre (`#1234`) y quiere
  consultar estado, marcar un slice, replanificar o cerrar la padre. Paso 5.

Repositorio siempre con `gh repo view --json nameWithOwner -q .nameWithOwner`. Nunca hardcodearlo.

## 1. Entender el cambio

1. Lee la entrada entera. Si es una issue, `gh issue view <n> --json title,body,comments`.
2. Lee `CLAUDE.md` y las convenciones de `docs/conventions/` de las capas que se van a tocar.
3. Explora solo el código del dominio que cambia: dónde vive hoy el comportamiento, qué contratos
   (API, eventos, modelos) se tocan y quién los consume. No explores clientes ni código ajeno al
   cambio.
4. Lista las preguntas que bloquean el troceado (reglas de negocio ambiguas, contrato sin decidir,
   qué pasa con los datos existentes) y házselas al humano **antes** de trocear. Lo que no bloquea
   va a *Preguntas abiertas* en la padre.

## 2. Trocear

Un slice es **una PR**. Cada uno cumple todas:

- **Vertical**: atraviesa las capas que hagan falta para que el cambio sea observable (un test de
  aceptación o de integración lo demuestra), no "la capa de dominio" y luego "la de infraestructura".
- **Desplegable solo**: se puede mergear y desplegar a producción sin el siguiente. Si cambia
  comportamiento visible antes de que la US esté completa, va detrás de una feature flag; la flag
  se crea en el primer slice que la necesita.
- **Revisable**: un humano lo revisa de una sentada. Si la lista de tests pasa de ~8 o toca más de un
  concern técnico independiente (persistencia, código sin cablear, bifurcación por flag), pártelo.
- **Sin anticipar**: solo lo que exigen sus tests. Nada "para el slice siguiente".

Orden: primero el camino feliz más estrecho de punta a punta, después los casos alternativos y de
error, al final la limpieza (quitar la flag, borrar el camino viejo). Lo que pueda ir en paralelo
no debe depender de nada.

**Dependencias**: un slice depende de otro solo si necesita su **código** o una **decisión** que se
toma en él. Compartir ficheros no es dependencia: los conflictos los resuelve el merge.

**Slices manuales**: lo que no puede hacer un agente (configuración en un panel externo, activar la
flag, datos de producción, coordinar con otro equipo) es un slice con label `manual`. Si un slice
de código depende de él, se declara la dependencia.

## 3. Redactar

### Issue padre

Título: el de la user story. Body con estas secciones, en este orden:

```markdown
## Contexto
<qué problema resuelve y para quién, en 3-5 líneas>

## Decisiones
<decisiones de diseño que afectan a más de un slice, con el porqué>

## Contrato
<tablas compartidas: endpoints, payloads, eventos, enums, códigos de error, nombres de flag>

## Fuera de alcance
<lo que la US no hace, para que ningún slice lo añada>

## Preguntas abiertas
<lo que no bloquea el arranque; el slice afectado lo cita>

Slices: ver sub-issues.
```

Todo lo que cite más de un slice va aquí, no repetido en las sub-issues.

### Sub-issue por slice

Título `S<n> · <qué hace, en una frase>`. Body:

```markdown
Parte de #<padre>.

## Objetivo
<qué cambia para el usuario o el sistema cuando se mergea, en una o dos frases>

## Alcance
- <qué se toca: capas, módulos, endpoints>

## Fuera de alcance
- <lo que parece de este slice y no lo es, con el slice que lo hace>

## Contrato
<la parte del contrato de la padre que este slice implementa, o "sin cambios de contrato">

## Feature flag
<nombre de la flag y qué hace cada rama, o "no aplica" y por qué>

## Lista de tests
1. <comportamiento observable, en el orden de TDD: del caso más simple al más complejo>
2. ...

## Hecho cuando
- <criterios verificables en la PR: tests, CI, comportamiento con la flag apagada sin cambios>
```

Reglas de redacción:

- **Autocontenida**: se entiende con ella y la padre, sin leer las otras sub-issues.
- **La lista de tests describe comportamiento**, no implementación ("devuelve 409 si el carrito ya
  está confirmado", no "añadir un `if` en el servicio"). Es el punto de partida del worker, no un
  techo.
- **Sin nombres de clases o ficheros nuevos** salvo que formen parte del contrato: los decide el
  worker siguiendo las convenciones.
- Sin "etc." ni "y similares": lo que no está escrito no se hace.

## 4. Revisar con el humano y publicar

1. Presenta el plan en la conversación: la padre resumida y una tabla con Slice, Objetivo,
   Blocked by, Flag, Manual. Debajo, el grafo de dependencias en una línea por rama
   (`S1 → S2 → S4`, `S1 → S3`). **Espera su OK**: crear issues es visible para todo el equipo.
   Itera sobre el plan las veces que haga falta.
2. Labels: `aiplan` en padre y sub-issues; `manual` en los slices manuales. Si no existen,
   `gh label create`.
3. Si ya existe una padre para esta US (mismo título, label `aiplan`), trabaja sobre ella (paso 5.3).
   Si no, créala:
   ```bash
   gh issue create --title "<título>" --label aiplan --body-file - --repo <owner/repo> <<'EOF'
   <body de la padre>
   EOF
   ```
4. Una sub-issue por slice:
   ```bash
   gh issue create --parent <padre> --title "S1 · <título>" --label aiplan --body-file - --repo <owner/repo> <<'EOF'
   <body del slice>
   EOF
   ```
5. Dependencias, con el `id` interno de la issue que bloquea (no su número):
   ```bash
   BLOCKER_ID=$(gh api repos/<owner/repo>/issues/<bloqueante> --jq .id)
   gh api -X POST repos/<owner/repo>/issues/<bloqueada>/dependencies/blocked_by -F issue_id=$BLOCKER_ID
   ```
6. Resumen (paso 6).

## 5. Operar sobre un plan publicado

Lee las sub-issues de la padre:

```bash
gh api repos/<owner/repo>/issues/<padre>/sub_issues --jq '.[] | {number, title, state}'
gh api repos/<owner/repo>/issues/<sub>/dependencies/blocked_by --jq '.[] | {number, state}'   # por sub-issue
```

Por cada una: slice (`S<n>` del título), estado, PR asociada (comentario `Completado en PR #NNN.` o
PR abierta que la cierra) y sus *blocked by*.

1. **Marcar slice completado**: `gh issue close <sub> --comment "Completado en PR #NNN."` (sin PR,
   sin comentario). Antes de evaluar el cierre de la padre.
2. **Cerrar la padre**: relee las sub-issues; si **todas** están cerradas,
   `gh issue close <padre> --comment "Todos los slices completados."`.
3. **Replanificar**: aplica los pasos 2 a 4 sobre el plan publicado. Actualiza las sub-issues
   afectadas con `gh issue edit <sub> --body-file -` y crea las de los slices nuevos. Nunca borres
   una sub-issue: si un slice desaparece, avísalo al humano. El body de la padre siempre se
   reescribe completo, nunca con parches parciales.

Si un `gh issue edit`/`close` falla, **detén el flujo** e informa. No sigas cerrando.

## 6. Resumen

Tabla con columnas Slice, Sub-issue, Blocked by, Estado, PR. Estados: `created`, `updated`,
`unchanged`, `closed`. Debajo, el enlace a la issue padre.

## Reglas

- **Idempotencia**: antes de crear, lista las sub-issues de la padre. Un slice cuyo `S<n> ·` ya
  existe se actualiza, nunca se duplica.
- **Siempre `--body-file -`** (stdin), nunca `--body` inline.
- **PRs**: `Closes #<sub-issue>` cierra el slice al mergear. **Nunca** `Closes`/`Fixes`/`Resolves
  #<padre>`.
- **Fallo graceful**: si algo falla a mitad, di qué quedó creado y qué falta.
