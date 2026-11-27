---
name: testing-stack
description: Verifica que un cambio terminado no rompa el stack de ATBJJ (Next.js 14 y Flask 3) — typecheck, build, lint e import. Úsala cuando el usuario acepte o cierre un cambio, diga que ya quedó, o pida revisar que no haya errores en el código. No la uses para desplegar ni para auditar seguridad.
---

# Testing — stack ATBJJ (Next.js 14 + Flask 3)

Actúa como experto en testing del stack del proyecto. Se activa cuando el usuario **da por terminado o acepta un cambio**: tu trabajo es comprobar que ese cambio no introduce errores antes de cerrarlo. Eres un **gate de verificación**, no de generación: no implementas features nuevas ni despliegas.

## Stack y ubicación

| Parte | Repo (remoto) | Carpeta local | Stack |
|---|---|---|---|
| Web | `ATBJJ` | `c:\Users\dante\Documents\trae_projects\aztlan-2026` | Next.js 14 (App Router), React 18, TypeScript 5.5, Tailwind 3.4 |
| API | `ATBJJ-api` | `c:\Users\dante\Documents\trae_projects\aztlan-2026-api` | Python 3.10+, Flask 3, SQLAlchemy 2, pydantic 2, MySQL (PyMySQL), flasgger |

Este shell es **PowerShell 5**: sin heredoc y con comillas anidadas frágiles.

## Estado del tooling (léelo antes de invocar cualquier comando)

Hoy **ningún** repo tiene linter ni tests instalados: eso lo añaden los tickets de CI (web `#12`: eslint + `tsc`; API `#4`: ruff + black + pytest). Por eso hay dos niveles de verificación.

**Nivel 1 — siempre disponible.** Es el que tienes que usar hoy:

- Web: `npx tsc --noEmit` y `npm run build` (`next build` ya hace el chequeo de tipos). Verificado en local: funciona.
- API: `python -m compileall -q app.py config.py models routes schemas services utils` (chequeo de sintaxis/parseo; escribe `__pycache__/`, que ya está ignorado por git). Verificado en local: funciona.

> La carpeta de la API es **plana** (`app.py`, `config.py`, `models/`, `routes/`, `schemas/`, `services/`, `utils/`); aún **no** existe el paquete `app/` que describe el plan. Pasa esos nombres exactos a `compileall`.
>
> Cuidado: el `venv/` que hay dentro de la carpeta de la API es un entorno de **macOS** (`bin/`, pyenv en `/Users/dantedev`), inservible en Windows. **No lo invoques.** El smoke test real de import (`python -c "import app"`) solo corre si creas un venv propio de Windows e instalas `requirements.txt`; en su defecto, `compileall` es la verificación válida en local.

**Nivel 2 — úsalo primero en cuanto exista** (tras #4 y #12):

- Web: `npm run lint` (eslint + eslint-config-next) y `npm run typecheck` (`tsc --noEmit`).
- API: `ruff check .`, `black --check .`, `pytest -q`.

Regla dura: **detecta antes de invocar**. No ejecutes un comando cuyo script o paquete todavía no existe. En particular, **no corras `npm run lint` mientras el repo no tenga `eslint` instalado**: `next lint` abre un prompt interactivo y cuelga la terminal. Si no hay Nivel 2, cae al Nivel 1.

## Cuándo se activa

- El usuario dice que un cambio quedó listo, lo acepta, lo aprueba, o pide "revisa que no haya errores".
- Comprueba primero qué se tocó (`git -C <ruta> status --short`) para verificar solo los repos afectados; si el cambio cruza front y API, verifica **ambos**.

Fuera de alcance: desplegar (eso es `deploy`), auditar seguridad (eso es `nextjs-appsec-audit`), y el análisis profundo de suites de Next existentes (eso es la skill global `nextjs-testing`, que complementa a esta).

## Procedimiento

### 1. Localizar el alcance

```powershell
git -C "c:\Users\dante\Documents\trae_projects\aztlan-2026" status --short
git -C "c:\Users\dante\Documents\trae_projects\aztlan-2026-api" status --short
```

### 2. Verificar el front (solo si se tocó)

```powershell
npx tsc --noEmit
npm run build
```

Ejecuta en `c:\Users\dante\Documents\trae_projects\aztlan-2026`. El build debe terminar sin errores ni warnings de tipos. Si el proyecto ya tiene `typecheck` y `lint` (Nivel 2), córrelos antes que el build.

### 3. Verificar la API (solo si se tocó)

```powershell
python -m compileall -q app.py config.py models routes schemas services utils
```

Ejecuta en `c:\Users\dante\Documents\trae_projects\aztlan-2026-api`. Detecta errores de sintaxis y de parseo. Si tienes un venv de Windows con las dependencias instaladas, añade el smoke test de arranque:

```powershell
.\venv\Scripts\python.exe -c "import app"
```

`import app` no levanta el servidor ni toca MySQL; solo confirma que el módulo importa sin traceback.

### 4. Reportar y, si falla, proponer el fix

No arregles en silencio: primero deja claro qué falló y por qué. Corrige solo si el usuario lo pide (o si el propio pedido lo autoriza). Tras el arreglo, vuelve a correr el comando exacto que falló y confirma que el resultado cambia a verde.

## Formato de reporte

```
Verificación <web|api> — <PASA|FALLA>
Comando    <el que corriste>
Resultado  <resumen: 0 errores / N errores>
Causa/Fix  <solo si falla: causa raíz y arreglo propuesto>
```

Si todo pasa, dilo en una línea por repo y continúa. No hace falta pegar la salida completa salvo los errores.

## Errores comunes

| Síntoma | Causa probable |
|---|---|
| `npm run lint` se queda esperando | No hay `eslint` instalado y `next lint` abre un prompt; usa `npx tsc --noEmit` |
| Error de tipos en `next build` | `tsc` lo señala primero; corrige el tipo y repite |
| `ModuleNotFoundError` al importar `app` | Falta una dependencia o el venv no está completo: `pip install -r requirements.txt` |
| `Cannot find ... \venv\Scripts\python.exe` | El `venv/` del repo es de macOS (`bin/`), no de Windows; usa `compileall` o crea un venv propio (`python -m venv venv` + `pip install -r requirements.txt`) |
| `Can't list 'app'` en `compileall` | No existe el paquete `app/` (la carpeta es plana); pasa `app.py config.py models routes schemas services utils` |
| `import app` intenta conectar a MySQL | Acoplamiento indebido en el import (la conexión debería ser perezosa); repórtalo como smell |
| El cambio toca ambos repos y solo verificaste uno | Repite el paso del repo que falta |

## Relación con otras piezas

- **CI (futuro):** cuando existan los workflows de #4 y #12, la verificación autoritativa es GitHub Actions. Para leer su estado usa la skill `github-repos-status`.
- **Antes de publicar:** el gate de esta skill es el mismo que exige `deploy` (build del front sin errores, API que importe). No dupliques: verifica aquí y deja el deploy a su skill.
