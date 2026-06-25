# EVDESEMP_EVALUADOR — Evaluador standalone (RRHH Facorsa)

App web estática de **una sola página** para que cada evaluador realice sus evaluaciones
de desempeño. Es una versión recortada ("standalone") del portal principal **EvDesemp**
(carpeta hermana `C:\Users\dell\OneDrive\Downloads\EvDesemp`). Comparten la misma base
de datos Supabase, pero este proyecto **solo expone el flujo de "Realizar evaluación"**,
protegido por PIN.


## Stack
- HTML/CSS/JS plano, **sin build, sin framework, sin npm**. Se sirve estático.
- `@supabase/supabase-js v2` (UMD) cargado por CDN desde `index.html`.
- Backend = **Supabase** (proyecto `gsrivgwhmnbjzlbwdqlx`, org FARRHH "RRHH Facorsa").
- Repo: **GitHub org Facorsa** → `https://github.com/facorsa/EVDESEMP_EVALUADOR`
  (remote `origin`). Migrado desde la cuenta personal `joselo1261`.
- Deploy: **Vercel — cuenta Facorsa PRO** (`https://vercel.com/facorsa`), sitio estático.
  Migrado desde la cuenta personal `joselo1261s-projects`.

## Archivos
| Archivo | Qué es |
|---|---|
| `index.html` | Única página. `body[data-page="realizar"]`. Card sticky con selector de **Evaluador** + filtros + búsqueda; listado de evaluados; modal de evaluación. Filtro de búsqueda client-side embebido al final. |
| `main.js` | Toda la lógica (≈264 KB). Define `SUPABASE_URL` / `SUPABASE_ANON_KEY` y las constantes de tablas (`T_*`). Contiene módulos heredados del portal (asignaciones, resultados, comparativos, dashboard, aviso) pero acá solo corre el de **Evaluaciones/realizar**. Incluye el flujo PIN + sesión. |
| `styles.css` | Estilos principales (heredados del portal). |
| `styles-local-mobile.css` / `local-mobile.js` | Ajustes mobile específicos del standalone. |
| `logo-white.png`, `evaluacion.png` | Assets. |
| `favicon.svg` | Favicon (enlazado en `index.html` con `rel="icon" type="image/svg+xml"`). Misma base que el del portal madre (degradado celeste→azul, `rx=14`, viewBox `64×64`) pero con una **persona + tilde** en vez del gráfico de barras. |

> Nota: la clave **anon** de Supabase está en `main.js` (línea ~10). Es pública por diseño
> (es la anon key, no la service_role). La seguridad real depende de RLS + la RPC de PIN.

## Autenticación de evaluador (PIN)
No usa Supabase Auth. Es un esquema propio basado en número de legajo:

1. El evaluador elige su nombre en el selector y, al entrar a evaluar, se le pide la **clave**.
   **La clave ES su Número de Legajo** (ej. `L0016`). El front lo **uppercasea y trimea**
   (`main.js` ~305).
2. Se llama la RPC **`rrhh_validar_pin_crear_sesion(p_legajo_nro, p_pin, p_horas)`**
   con `p_pin = p_legajo_nro` (el mismo valor) y `p_horas = 12` (`main.js` ~313 / ~371).
3. La RPC (Postgres, `SECURITY DEFINER`):
   - Busca en **`rrhh_eval_pin`** la fila con `legajo_nro = p_legajo_nro AND activo = true`.
     Si no existe → `raise exception 'Legajo/PIN inválido'`.
   - Verifica `crypt(p_pin, pin_hash) = pin_hash` (bcrypt vía `pgcrypto`).
   - Crea/renueva la sesión en `rrhh_eval_session` (12 h) y devuelve `session_id`, `legajo_id`, `expires_at`.
4. El front guarda `session_id` + `legajo_id` + `legajo_nro` en `localStorage`
   (`rrhhStoreSession`). La sesión expira a las 12 h; `rrhhClearStoredSession()` limpia.
5. Al evaluar se valida que el evaluador seleccionado coincida con el de la sesión
   (`rrhhEnsureAuthorizedEvaluator`).

**Importante:** el mensaje "Legajo/PIN inválido" lo emite la **RPC en Postgres**, no el
front ni Vercel. Si todas las claves fallan, casi siempre es porque `rrhh_eval_pin` está
vacía o desincronizada — **no** es un bug del front.

## Tablas Supabase que toca (constantes en `main.js`)
- `rrhh_legajos_stage` (`T_LEGAJOS`) — padrón. Columnas con espacios/mayúsculas:
  `"ID"` (uuid), `"Nombre Completo"`, `"Sucursal"`, `"Gerencia"`, `"Sector"`,
  `"Baja"` (activo = `'Activo'`), `"Legajo Nro."` (ej. `L0016`).
- `rrhh_eval_pin` (`T_PIN`) — credenciales de acceso del evaluador.
  Columnas: `legajo_id uuid`, `legajo_nro text`, `pin_hash text` (bcrypt), `activo bool`, `updated_at`.
  **Source of truth del login. NO es data de prueba ni efímera.**
- `rrhh_eval_session` — sesiones (sí efímera; se regenera al loguear).
- `rrhh_eval_asignaciones` (`T_ASIG`) — Evaluador → Evaluados por año.
- `rrhh_eval_cab` / `rrhh_eval_items` / `rrhh_eval_respuestas` — evaluación de desempeño.
- `rrhh_cp_cab` / `rrhh_cp_items` / `rrhh_cp_respuestas` — Compromiso y Presentismo.
- `rrhh_legajo_flags` — elegibilidad (`es_evaluable_cp`).
- `rrhh_eval_aviso_config` / `rrhh_eval_emails` — config y emails de aviso a evaluadores.

## Convenciones / cosas a saber
- Este proyecto **no debería tocar el esquema de la base**. Cambios de datos/DDL se hacen
  desde el portal principal `EvDesemp` o directo en el SQL Editor de Supabase.
- El número de legajo se compara **exacto** en la RPC; el front siempre manda mayúsculas.
  Por eso cualquier seed de `rrhh_eval_pin` debe guardar `legajo_nro` en MAYÚSCULA/sin espacios.
- `pgcrypto` está instalado (la RPC usa `crypt`/`gen_salt`).

---

## Registro de incidentes

### 2026-06-25 — Migración de hosting a cuentas Facorsa
- **GitHub:** el repo pasó de la cuenta personal `joselo1261`
  (`https://github.com/joselo1261`) a la **org Facorsa**
  (`https://github.com/orgs/facorsa/` → repo `facorsa/EVDESEMP_EVALUADOR`).
  El remote `origin` local ya apunta a `https://github.com/facorsa/EVDESEMP_EVALUADOR.git`.
- **Vercel:** el deploy pasó de la cuenta personal `joselo1261s-projects`
  (`https://vercel.com/joselo1261s-projects`) a la **cuenta Facorsa PRO**
  (`https://vercel.com/facorsa`).
- **Sin cambios de código ni de base:** misma Supabase, mismo flujo. Solo cambió
  dónde vive el repo y desde dónde se despliega.

### 2026-06-23 — "Legajo/PIN inválido" en todas las claves
- **Síntoma:** todos los logins de evaluador fallaban con "Legajo/PIN inválido", incluida
  `L0016` (Caramagna, Mariela — Activo).
- **Diagnóstico:** el mensaje sale de la RPC `rrhh_validar_pin_crear_sesion`. La tabla
  `rrhh_eval_pin` estaba **vacía** (0 filas), así que la RPC nunca encontraba la fila.
- **Causa raíz:** el script `supabase/limpiar_datos_prueba.sql` del portal **EvDesemp**
  incluía `DELETE FROM rrhh_eval_pin` bajo el supuesto (erróneo) de que "se regenera sola
  al loguearse". No se regenera: la RPC la **lee** para validar. Al limpiar datos de prueba
  se borraron las credenciales.
- **Arreglo (solo data, sin tocar código):** se re-sembró `rrhh_eval_pin` con
  `EvDesemp/supabase/resembrar_pins.sql`:
  - Fuente: `rrhh_legajos_stage` (`"Baja" = 'Activo'`), un PIN por legajo activo.
  - `legajo_id = "ID"`, `legajo_nro = upper(trim("Legajo Nro."))`,
    `pin_hash = crypt(legajo_nro, gen_salt('bf'))`, `activo = true`. PIN = nº de legajo.
  - Idempotente (`NOT EXISTS`). Verificado: `crypt('L0016', pin_hash) = pin_hash` → `true`.
- **Prevención:** quitar `DELETE FROM rrhh_eval_pin` del script de limpieza (mover a
  "CONSERVAR"): es credencial de acceso, no data de prueba.
