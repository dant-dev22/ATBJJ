---
name: github-repos-status
description: Revisa el estado de GitHub de ambos repos ATBJJ (web y API) — ramas, PRs, issues, protección y el Project 3. Úsala siempre que se mencione github, git, control de versiones, ramas, PRs, issues o repos. Para desplegar usa la skill deploy.
---

# Estado de GitHub — ATBJJ

Diagnóstico de **solo lectura** del estado de los dos repositorios de ATBJJ y del tablero de la migración SaaS. Devuelve una foto clara de qué está pendiente, qué está desincronizado y qué riesgo hay antes de tocar nada.

No hace commit, push, merge, rebase ni ningún cambio. Si el usuario pide modificar algo, eso es trabajo de la skill `deploy` (publicar/cambiar) y de las reglas de seguridad de Git; esta skill solo mira y reporta.

## Datos fijos

| Dato | Valor |
|---|---|
| Propietario | `dant-dev22` |
| Repo web (remoto) | `dant-dev22/ATBJJ`, rama por defecto `main`, integración en `staging` |
| Repo API (remoto) | `dant-dev22/ATBJJ-api`, rama por defecto `main`, integración en `staging` |
| Repo web (local) | `c:\Users\dante\Documents\trae_projects\aztlan-2026` |
| Repo API (local) | `c:\Users\dante\Documents\trae_projects\aztlan-2026-api` |
| Project board | `#3` "Aztlan 2026 — Migración SaaS", en `https://github.com/users/dant-dev22/projects/3` |
| Estados del board | `Backlog` · `En curso` · `En revisión` · `Hecho` |
| Fuente de verdad técnica | `docs/PLAN-SAAS.md` (en el repo web) |
| Flujo de ramas | `feature/* → PR a staging → 1 aprobación → merge a main (deploy)` |

Ojo con el desfase de nombres: las **carpetas locales** siguen llamándose `aztlan-2026` y `aztlan-2026-api`, pero los **repos remotos** ya se llaman `ATBJJ` y `ATBJJ-api`. Usa las rutas locales para el estado en disco y los nombres `ATBJJ*` para todo lo de `gh`.

Los otros tableros del usuario (`@dant-dev22's SPA` #2, `Aritmetrica network` #1) son de proyectos ajenos: ignóralos.

## Herramientas

`gh` (GitHub CLI, autenticado como `dant-dev22`) y `git`. Verifica con `gh auth status` si algo falla. Scopes actuales del token: `repo`, `project`, `read:org`, `gist`, `admin:public_key`. **No incluye `workflow`**: la API de `gh` no puede crear ni editar `.github/workflows/*` (los push por SSH sí), aviso útil para los tickets de CI.

## Procedimiento

Ejecuta los bloques en paralelo cuando puedas y luego resume. Este shell es **PowerShell 5** (sin heredoc; `--jq` va entre comillas simples).

### 1. Estado local de ambos repos

```powershell
foreach ($r in @("c:\Users\dante\Documents\trae_projects\aztlan-2026","c:\Users\dante\Documents\trae_projects\aztlan-2026-api")) { echo "=== $r ==="; git -C $r fetch --all --prune -q; git -C $r status -sb; git -C $r branch -vv; git -C $r log --oneline -3; git -C $r stash list }
```

De ahí lee: rama actual, upstream y ahead/behind, ficheros modificados/untracked, operación a medias (rebase/merge), ramas locales sin publicar y stashes. `git fetch` solo actualiza referencias remotas; no toca el working tree.

### 2. Estado remoto de ambos repos

```powershell
foreach ($r in @("ATBJJ","ATBJJ-api")) { echo "=== $r ==="; gh api "repos/dant-dev22/$r/branches" --jq '.[].name'; echo "-- main --"; gh api "repos/dant-dev22/$r/branches/main/protection" --jq '{reviews: .required_pull_request_reviews.required_approving_review_count, enforce_admins: .enforce_admins.enabled, checks: .required_status_checks}'; echo "-- PRs --"; gh pr list -R "dant-dev22/$r" --state open --json number,title,headRefName,author,isDraft; echo "-- issues --"; gh issue list -R "dant-dev22/$r" --state open --json number,title,labels }
```

### 3. Tablero de la migración

```powershell
gh project item-list 3 --owner dant-dev22 --format json
```

Resume el JSON en una tabla por estado: ticket (`repo#número`), título, estado y etiquetas. Marca aparte los que dependen de tickets aún abiertos.

### 4. Reporte

Sé breve. Un bloque compacto por repo más una línea del tablero:

```
Web  (ATBJJ)     rama <x> · upstream <sí/no> · <N> ahead / <N> behind · <limpio|N cambios>
API  (ATBJJ-api) rama <x> · upstream <sí/no> · <N> ahead / <N> behind · <limpio|N cambios>
Remoto           PRs abiertos: <N> · issues abiertos: <N> · main: <1 aprobación|X> + checks <sí/no>
Board #3         Hecho <a>/<b> · En curso <c> · Backlog <d>
Riesgos          <lista corta o "ninguno">
Sugerencia       <siguiente paso concreto o "nada que hacer">
```

Si todo está sincronizado y sin riesgos, dilo en una línea y sigue con la tarea del usuario.

## Alertas que hay que mirar siempre

| Señal | Lectura |
|---|---|
| `checks: null` en la protección de `main` | Sin CI bloqueante: la aprobación es lo único que frena un merge (tickets de CI #4 y #12 pendientes) |
| Falta `staging` en alguno de los repos | El flujo `feature/* → staging → main` está roto |
| `venv/` o `*.pyc` rastreados en `ATBJJ-api` | Regresión de higiene (ticket #3); debería estar ignorado |
| Divergencia local (ahead **y** behind) | No hacer push a ciegas: `pull --rebase` o preguntar |
| Ramas `feature/*` viejas sin PR abierto | Trabajo huérfano: ¿cerrar o retomar? |
| PRs abiertos sin revisor | Sin aprobación no se puede fusionar a `main` |
| VPS con el historial viejo de la API | `git pull` en el VPS fallará hasta resincronizar (`fetch` + `reset`) |

## Errores y bloqueos

- Si `gh` responde `authentication required`, ejecuta `gh auth status` y, si hace falta, `gh auth login`.
- Si un `--jq` con comillas rotas falla en PowerShell, simplifica la expresión o consulta dos veces.
- Si `git status` reporta conflicto, rebase o merge en curso, **detente** y repórtalo antes de seguir.
- No modifiques nada para "arreglar" lo que reportes sin que el usuario lo pida explícitamente.
