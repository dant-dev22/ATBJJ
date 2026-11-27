# AGENTS.md — ATBJJ / Aztlan 2026

> **Para qué sirve este archivo.** Es la foto completa de los dos repositorios (rutas, componentes, endpoints, modelos, infra y convenciones) para que un agente de IA entienda el proyecto **sin releer el código**. Si necesitas el *porqué* de las decisiones y el plan de migración, ve a [`PLAN-SAAS.md`](./docs/PLAN-SAAS.md); este documento es el *qué* y el *dónde*.
>
> Última verificación contra el código real de ambos repos (no supuestos).

---

## 1. Los dos repositorios

| | Repo web | Repo API |
|---|---|---|
| Nombre remoto | `dant-dev22/ATBJJ` | `dant-dev22/ATBJJ-api` |
| Carpeta local | `c:\Users\dante\Documents\trae_projects\aztlan-2026` | `c:\Users\dante\Documents\trae_projects\aztlan-2026-api` |
| Stack | Next.js 14 (App Router) + React 18 + TypeScript 5.5 + Tailwind 3.4 | Python 3.10+ + Flask 3 + SQLAlchemy 2 + pydantic 2 + MySQL (PyMySQL) + flasgger |
| Rama de producción | `main` | `main` |
| Rama de integración | `staging` | `staging` |
| Puerto en desarrollo | 3000 | 5000 |

Los **slugs antiguos** (`aztlan-2026`, `aztlan-2026-api`) siguen redirigiendo en GitHub. La nomenclatura nueva es `ATBJJ-<componente>`.

**Plan maestro:** `docs/PLAN-SAAS.md` (fuente de verdad). **Tablero:** Project #3 "Aztlan 2026 — Migración SaaS".

## 2. Arranque y verificación rápida

```powershell
# Web — desarrollo
npm run dev          # http://localhost:3000   (en c:\...\aztlan-2026)

# Web — verificar un cambio (hoy, sin eslint instalado)
npx tsc --noEmit     # typecheck
npm run build        # build completo (next build ya chequea tipos)

# API — verificar un cambio (hoy, sin tests)
python -m compileall -q app.py config.py models routes schemas services utils
```

> **Ojo (Windows):** el `venv/` que hay dentro de la carpeta de la API es un entorno de **macOS** (`bin/`, pyenv en `/Users/dantedev`); no sirve en Windows. El smoke test `python -c "import app"` solo corre en un venv propio de Windows con `pip install -r requirements.txt`.
>
> **Ojo:** `npm run lint` **no** funciona hoy (no hay `eslint` instalado → `next lint` abre un prompt y cuelga). Los tickets web `#12` y API `#4` añaden linter y tests.

---

## 3. Repo web — estructura

```
aztlan-2026/
├── AGENTS.md                        # Este archivo
├── app/
│   ├── page.tsx                     # Home: flujo de registro (client)
│   ├── layout.tsx                   # Layout raíz
│   ├── globals.css                  # Estilos globales + tokens Tailwind
│   ├── admin/
│   │   ├── page.tsx                 # Dashboard admin (login + tabla + filtros)
│   │   └── auth/route.ts            # API route de sesión admin (GET/POST)
│   ├── [equipo]/[sucursal]/asistencias/page.tsx   # Check-in por QR
│   ├── registro/{juvenil,adultos,masters}/page.tsx
│   ├── reglamento/page.tsx
│   ├── lista-final/page.tsx
│   ├── components/                  # Ver §5
│   ├── hooks/useUsers.ts
│   ├── lib/                         # Ver §6
│   └── types/window.d.ts
├── docs/PLAN-SAAS.md
├── nginx/aztlan.conf                # Plantilla de vhost (no es el vivo)
├── public/images/logoaztlan.svg
├── .env.production                  # NEXT_PUBLIC_API_URL=/api
├── next.config.js                   # redirect /registro/infantil → /registro/juvenil
├── tailwind.config.js / postcss.config.js / tsconfig.json
└── update-front.sh                  # Script de deploy con rollback (referencia)
```

## 4. Repo web — rutas (páginas y su comportamiento)

| Ruta | Archivo | Qué hace |
|---|---|---|
| `/` | [page.tsx](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/page.tsx) | Home. Muestra instrucciones + 3 tarjetas (juvenil / adultos / masters) y los CTAs "iniciar registro" y "terminar registro". Abre modales. Si `REGISTRO_ABIERTO=false`, muestra `RegistroCerrado`. |
| `/registro/juvenil` | `registro/juvenil/page.tsx` | Página de registro infantil y juvenil. |
| `/registro/adultos` | `registro/adultos/page.tsx` | Registro adultos. |
| `/registro/masters` | `registro/masters/page.tsx` | Registro masters. |
| `/registro/infantil` | — | **Redirect 301** a `/registro/juvenil` (definido en `next.config.js`). |
| `/reglamento` | `reglamento/page.tsx` | Reglamento del torneo. |
| `/lista-final` | [page.tsx](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/lista-final/page.tsx) | Lista de aprobados por categoría. **Gateada** por `LISTA_FINAL_PUBLICA` (hoy `false` → muestra "no disponible"). |
| `/admin` | [page.tsx](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/admin/page.tsx) | Dashboard: login modal, tabla de participantes, filtros (pago/peso/categoría/tipo), aprobar comprobante, eliminar, descargar CSV. |
| `/admin/auth` | [route.ts](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/admin/auth/route.ts) | API route (no página). `GET` valida sesión; `POST` hace login. |
| `/[equipo]/[sucursal]/asistencias` | [page.tsx](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/[equipo]/[sucursal]/asistencias/page.tsx) | Check-in: escanea QR, elige clase y registra asistencia. Los slugs se convierten a etiquetas con `slugToLabel`. |

## 5. Repo web — componentes

**Raíz `app/components/`**

| Componente | Rol |
|---|---|
| `AppLogo.tsx` | Logo SVG (props de tamaño/priority). |
| `Header.tsx` | Cabecera. Props: `title`, `subtitle`, `showBackButton`, `backHref`, `centered`, `navLinks[]`. |
| `Footer.tsx` | Pie de página. |
| `Modal.tsx` | Modal genérico (`isOpen`, `onClose`, `showCloseButton`). |
| `CardsRegistro.tsx` | Las 3 tarjetas de categoría + botón volver. |
| `BotonesFlujoPrincipal.tsx` | CTAs "iniciar registro" / "terminar registro". |
| `InstruccionesRegistro.tsx` | Bloque de instrucciones del home. |
| `RegistroCerrado.tsx` | Aviso de registro cerrado (variantes `home` / `page`). |
| `RegistroForm.tsx` | Formulario de registro por `tipoRegistro` (exporta el tipo `TipoRegistro`). Llama a `postRegistro`. |
| `ModalExitoRegistro.tsx` | Modal de éxito que muestra el `aztlan_id`. |
| `FormTerminarRegistro.tsx` | Formulario para subir comprobante con un `aztlan_id` existente. Llama a `postComprobante`. |
| `ListaParticipantes.tsx` | Lista de participantes (fetch directo a `getUsuarios`, filtra por `ADMIN_TOURNAMENT`). Duplica parte de la función del admin. |

**`app/components/admin/`** — usados por `/admin`

| Componente | Rol |
|---|---|
| `AdminTable.tsx` | Tabla principal de participantes. |
| `ApprovalSwitch.tsx` | Toggle de aprobación de comprobante. |
| `ConfirmModal.tsx` | Diálogo de confirmación (borrado). |
| `DeleteButton.tsx` | Acción de eliminar registro. |
| `DownloadReportButton.tsx` | Descarga de reporte CSV. |
| `SearchBar.tsx` | Buscador. |

**`app/components/asistencias/`** — usados por `/[equipo]/[sucursal]/asistencias`

| Componente | Rol |
|---|---|
| `AttendanceModule.tsx` | Orquesta el flujo de asistencia (catálogo → ID → clase → QR → envío). |
| `QrScanner.tsx` | Escáner QR (librería `html5-qrcode`). |

## 6. Repo web — `lib`, `hooks`, `types` y configuración

| Archivo | Contenido |
|---|---|
| [app/lib/api.ts](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/lib/api.ts) | **Cliente HTTP único.** `BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://<HOST_VPS>:5000'`. Define los tipos (`RegistroPayload`, `UsuarioApi`, `AsistenciaPayload`, …) y todas las funciones: `postRegistro`, `postComprobante`, `getUsuarios`, `patchUsuarioAprobar`, `deleteUsuario`, `getAsistenciasCatalogo`, `postAsistencia`. |
| [app/lib/listaFinal.ts](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/lib/listaFinal.ts) | `buildListaFinal()` (agrupa aprobados por categoría, ordena por cinta) y `buildCsvListaFinal()`. `TORNEO_ACTUAL = '2026-b'` hardcodeado. |
| [app/lib/registroConfig.ts](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/lib/registroConfig.ts) | Flags de features: `REGISTRO_ABIERTO = true`, `LISTA_FINAL_PUBLICA = false`. |
| [app/lib/slug.ts](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/lib/slug.ts) | `slugToLabel`, `labelToSlug`, `getInitials`. |
| [app/hooks/useUsers.ts](file:///c:/Users/dante/Documents/trae_projects/aztlan-2026/app/hooks/useUsers.ts) | Hook de estado de participantes: cargar, eliminar, aprobar comprobante, reemplazar lista. |
| `app/types/window.d.ts` | Tipos globales de `window`. |

**Constantes hardcodeadas a vigilar** (el plan las quiere mover a configuración por torneo): `TORNEO_ACTUAL` en `listaFinal.ts`, `ADMIN_TOURNAMENT` en `admin/page.tsx` y `ListaParticipantes.tsx`, y el torneo por defecto `2026-b` en `config.py` de la API.

---

## 7. Repo API — estructura

```
aztlan-2026-api/            # carpeta PLANA (aún no existe el paquete app/ del plan)
├── app.py                  # create_app(): db.init_app, db.create_all(), CORS '*', Swagger, blueprints
├── config.py               # Configuración desde .env
├── models/
│   ├── registro.py         # db = SQLAlchemy()  +  Registro
│   └── asistencia.py       # Asistencia
├── routes/
│   ├── registro.py         # blueprint "registro"     url_prefix=/registro
│   ├── comprobante.py      # blueprint "comprobante"  url_prefix=/comprobante
│   ├── usuarios.py         # blueprint "usuarios"      url_prefix=/usuarios
│   └── asistencias.py      # blueprint "asistencias"   url_prefix=/asistencias
├── schemas/                # pydantic (registro, comprobante, usuarios, asistencias)
├── services/
│   ├── email_service.py    # Gmail SMTP
│   └── registro_service.py
├── utils/aztlan_id.py      # generate_aztlan_id() → "azt" + 3 alfanuméricos
├── requirements.txt
├── env.example
└── README.md               # Documenta endpoints y deploy (desactualizado en el nombre del repo)
```

## 8. Repo API — endpoints

| Método | Ruta | Handler | Notas |
|---|---|---|---|
| `POST` | `/registro` | `routes/registro.py::post_registro` | Crea un registro; devuelve `aztlan_id`. **Sin auth.** |
| `POST` | `/comprobante` | `routes/comprobante.py::post_comprobante` | Asocia comprobante (base64) a un `aztlan_id`; queda pendiente de aprobación. **Sin auth.** |
| `GET` | `/usuarios` | `routes/usuarios.py::get_usuarios` | Lista **todos** los registros. **Sin auth** (bloqueo de seguridad). |
| `PATCH` | `/usuarios/<id>` | `routes/usuarios.py::patch_usuario` | Aprueba/rechaza comprobante (`{comprobanteAprobado}`). **Sin auth.** |
| `DELETE` | `/usuarios/<id>` | `routes/usuarios.py::delete_usuario` | Elimina registro. **Sin auth.** |
| `GET` | `/asistencias/catalogo` | `routes/asistencias.py::get_catalogo_asistencias` | Clases disponibles por `equipo` y `sucursal`. |
| `POST` | `/asistencias` | `routes/asistencias.py::post_asistencia` | Registra asistencia validando QR. |
| `GET` | `/` | `app.py::index` | Redirige a Swagger. |
| `GET` | `/apidocs/` | flasgger | Swagger UI. |
| `GET` | `/apispec.json` | flasgger | Especificación OpenAPI. |
| `GET` | `/comprobantes/<filename>` | `app.py::serve_comprobante` | Sirve archivos subidos desde `UPLOAD_FOLDER`. |

> Nota del plan: el endpoint `/usuarios` en realidad devuelve **competidores**, no usuarios de administración; en `/api/v1` se renombrará a `/v1/participants`.

## 9. Repo API — modelos

**`registros`** (`models/registro.py` → `Registro`): `id` (uuid), `aztlan_id` (unique), `torneo` (string, **indexado**), `tipo_registro` (`juvenil`|`adultos`|`masters`), `nombre_completo`, `email`, `equipo`, `timestamp` (ISO string); campos opcionales juveniles (`sexo`, `cinta`, `nivel_experiencia`, `categoria_edad`, `categoria_peso`) y de adultos/masters (`edad`, `categoria_peso_tipo`); y el comprobante **embebido**: `comprobante_aprobado`, `comprobante_url`, `comprobante_filename`, `comprobante_media_type`, `comprobante_size_bytes`. `created_at` con `server_default=now()`.

**`asistencias`** (`models/asistencia.py` → `Asistencia`): `id`, `torneo`, `registro_id` (FK → `registros.id`), `aztlan_id`, `nombre_completo`, `equipo` + `equipo_slug`, `sucursal` + `sucursal_slug`, `clase_id`, `clase_nombre`, `qr_contenido`, `timestamp`, `fecha_operativa` (YYYY-MM-DD), `created_at`.

## 10. Repo API — configuración (`config.py` / `env.example`)

| Variable | Default | Uso |
|---|---|---|
| `PORT` | `5000` | Puerto del servidor. |
| `CURRENT_TOURNAMENT` | `2026-b` | Torneo activo (hoy string, objetivo `tournament_id`). |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` | `localhost` / `3306` / `root` / `""` / `aztlan2026` | Conexión MySQL. La contraseña se escapa con `quote_plus`. |
| `UPLOAD_FOLDER` | `uploads/comprobantes` | Carpeta de comprobantes (se crea al arrancar). |
| `GMAIL_USER` / `GMAIL_APP_PASSWORD` | `""` | SMTP para correos. `EMAIL_ENABLED` es `true` solo si ambos existen. |
| `ADMIN_EMAIL` | `""` | Recibe aviso cuando un usuario sube comprobante. |
| `ATTENDANCE_CLASSES` | CSV de 4 clases (`fundamentos`, `intermedios`, `avanzados`, `open-mat`) | Formato `id\|Nombre\|Descripción`, separado por comas. |

**Esquema de BD:** lo crea la propia API con `db.create_all()` al arrancar (`app.py`). **No hay migraciones** (el plan introduce Alembic en la Fase 2).

**Variables del front (web):** `NEXT_PUBLIC_API_URL` (en producción `/api` vía proxy), y para `/admin`: `ADMIN_USER`, `ADMIN_PASSWORD`, `ADMIN_SESSION_SECRET` (mín. 16 caracteres).

---

## 11. Contrato front ↔ API

El front solo habla con la API a través de `app/lib/api.ts`. Correspondencia directa:

| Función del front | Endpoint | Modelo involucrado |
|---|---|---|
| `postRegistro` | `POST /registro` | `registros` |
| `postComprobante` | `POST /comprobante` | `registros` (comprobante embebido) |
| `getUsuarios` | `GET /usuarios` | `registros` |
| `patchUsuarioAprobar` | `PATCH /usuarios/<id>` | `registros` |
| `deleteUsuario` | `DELETE /usuarios/<id>` | `registros` |
| `getAsistenciasCatalogo` | `GET /asistencias/catalogo` | config `ATTENDANCE_CLASSES` |
| `postAsistencia` | `POST /asistencias` | `asistencias` |

Los tipos del front están **escritos a mano** en `api.ts` (no se generan del OpenAPI; el plan lo corrige con `ATBJJ-contracts`).

## 12. Infraestructura y despliegue

| Elemento | Valor |
|---|---|
| VPS | `<HOST_VPS>` (el valor real está en la skill `deploy` y hardcodeado como fallback en `app/lib/api.ts`) |
| Ruta del front en el VPS | `/root/aztlan2026/aztlan-2026` |
| Ruta de la API en el VPS | `/root/aztlan2026/aztlan-2026-api` (+ `venv` de Linux) |
| Proceso front | PM2 `react-ui-app`, puerto 3000 (`npm start`) |
| Proceso API | Gunicorn en 5000, **sin gestor de procesos** (`nohup ... &`, pid en `api.pid`) |
| Nginx | `/api/` → `127.0.0.1:5000/` ; `/` → `127.0.0.1:3000`. Plantilla en `nginx/aztlan.conf`; el vhost vivo es `/etc/nginx/sites-available/torneoaztlan`. Alias `/comprobantes/` → uploads de la API. |
| Dominios | `torneoaztlan.com` (principal, SSL Let's Encrypt); `torneoaztlan.cloud` redirige 301 al `.com`. |
| Script de referencia | `update-front.sh` (git reset + build + pm2 restart + healthcheck + rollback). |

Detalle del procedimiento y reglas del servidor: skill `deploy`.

## 13. Convenciones

- **Ramas:** `main` (producción), `staging` (pre-prod), `feature/<id>-descripcion`, `fix/…`, `chore/…`. Un PR va siempre contra `staging`; `staging → main` con 1 aprobación. `main` no admite push directo.
- **Commits:** Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`). Sin heredoc (PowerShell).
- **Nombres:** repos `ATBJJ-<componente>`; tags de release `web-vX.Y.Z` / `api-vX.Y.Z`.
- **Rutas alias:** el import usa `@/app/...` en el front.
- **Gitignored:** web → `node_modules/`, `.next/`, `.env*.local`, `next-env.d.ts`. API → `venv/`, `__pycache__/`, `*.pyc`, `.env*`, `uploads/`, `*.log`.

## 14. Estado actual y deuda conocida (del diagnóstico del plan)

Ya funciona y es reutilizable: registro público (3 flujos), admin, lista final, reglamento, asistencias, API en capas, Swagger, cliente HTTP único.

Bloqueos del SaaS (en orden de fase):

| Fase | Bloqueo / objetivo | Dónde |
|---|---|---|
| 0 (higiene) | `venv` y `__pycache__` fuera de git; ramas `main`/`staging`; CI de lint | repo API (hecho el purge), ambos |
| 1 (crítico) | **Sin autorización en la API**; admin protegido solo en cliente; `CORS '*'`; puerto `:5000` expuesto | `app.py`, `routes/usuarios.py`, `admin/page.tsx` |
| 2 | Sin migraciones (Alembic) ni staging ni Docker | `app.py` (`db.create_all`) |
| 3 (riesgo alto) | Sin multitenancy: un solo torneo hardcodeado, sin `tenant_id` | `config.py`, `listaFinal.ts`, `admin/page.tsx`, todo el esquema |
| 4 | Contrato a mano (tipos duplicados) | `app/lib/api.ts` |
| 5–6 | Sin identidad compartida ni onboarding self-serve | — |

La **fuga de datos entre organizaciones** es el riesgo más grave del proyecto (aislamiento solo en código; MySQL sin RLS).

## 15. Piezas de trabajo y contexto

| Pieza | Ruta / ubicación |
|---|---|
| Plan maestro | `docs/PLAN-SAAS.md` |
| Tablero | Project #3 "Aztlan 2026 — Migración SaaS" (milestone actual "Fase 0 - Higiene") |
| Skill: estado de GitHub | `.trae/skills/github-repos-status/SKILL.md` |
| Skill: ingeniero de proyectos | `.trae/skills/ingeniero-proyectos/SKILL.md` |
| Skill: testing del stack | `.trae/skills/testing-stack/SKILL.md` |
| Skill: deploy | `.trae/skills/deploy/SKILL.md` |

> **Nota de seguridad:** este documento vive en la raíz del repo web (público). El IP real del VPS se ha marcado como `<HOST_VPS>` igual que en `PLAN-SAAS.md`; el valor real sigue hardcodeado como fallback en `app/lib/api.ts` (ya público) y en la skill `deploy` (no versionada). Si vas a commitear este archivo, revisa antes qué secretos o rutas internas quieres exponer.
