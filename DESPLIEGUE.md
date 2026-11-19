# Guía de despliegue — Basel Venue · Control de turnos

## Estado actual (19-11-2026)

| Pieza | Servicio | URL / identificador |
|---|---|---|
| Frontend | **Vercel** · equipo `dev-craft` · proyecto `control-de-turnos` | https://control-de-turnos-mu.vercel.app |
| Kiosco (trabajadores) | — | https://control-de-turnos-mu.vercel.app/asistencia |
| Backend | **Render** · plan gratuito · Oregón · servicio `control-de-turnos-backend` (`srv-dbv4lsk9v7es738lq3tg`) | https://control-de-turnos-backend.onrender.com |
| Base de datos | **Supabase** (PostgreSQL 17) · Session pooler `aws-1-us-west-2` | — |

```
 Celular / tablet / PC
          │  https
          ▼
 ┌─────────────────────┐  https (API)  ┌──────────────────────┐  SSL (pooler)  ┌─────────────┐
 │ Vercel — frontend   │ ────────────▶ │ Render — backend     │ ─────────────▶ │  Supabase   │
 │ React (build)       │               │ Express (Docker)     │               │ PostgreSQL  │
 └─────────────────────┘               └──────────────────────┘               └─────────────┘
   repo: control-de-turnos-front         repo: control-de-turnos-backend
```

> **Despliegues automáticos desde `main`:** Vercel (repo `control-de-turnos-front`) y Render
> (repo `control-de-turnos-backend`) publican solos cada cambio que llega a `main`. Para cambios
> nuevos: rama → Pull Request → fusionar en `main`.

---

## 1. Flujo de cambios

- Cambios en backend o frontend: rama → PR en su repo → fusionar en `main` → despliegue automático.
- Si el cambio toca ambos, fusionar **primero los submódulos** (con **"Create a merge commit"**) y
  luego actualizar los punteros en el repo principal.
- La versión inicial se fusionó el 19-11-2026 (PRs #1 de los tres repos) y Render se pasó a `main`.

---

## 2. Backend en Render

### Servicio
- Tipo *Web Service*, runtime **Docker** (`Dockerfile` del repo), plan **Free**, región **Oregon**,
  health check `/`. Definición equivalente como código: `backend/render.yaml`.
- Render asigna el puerto (`PORT=10000`); no definirlo.

### Variables de entorno (Render → servicio → Environment)

| Variable | Valor |
|---|---|
| `DATABASE_URL` | Supabase → **Connect → Session pooler** (puerto 5432) |
| `JWT_SECRET` | Aleatorio, exclusivo de producción (ya generado; no reutilizar el de desarrollo) |
| `ADMIN_USER` / `ADMIN_PASS` | Credenciales del administrador |
| `CORS_ORIGIN` | `https://control-de-turnos-mu.vercel.app` (exacta, sin `/` final) |
| `JWT_EXPIRES_IN` | `15m` |
| `REFRESH_IDLE_HOURS` / `SESSION_MAX_DAYS` | `12` / `7` |
| `DB_POOL_MAX` | `5` |
| `DB_SSL` / `DB_SSL_CA` | *(recomendado, ver §2.2)* |

`TRUST_PROXY` **no se define**: el backend detecta Render y usa la cabecera `CF-Connecting-IP` de
Cloudflare como IP real del cliente para los límites de intentos (verificado: Cloudflare rechaza
esa cabecera si la envía el cliente, y falsificar `X-Forwarded-For` no evade el límite).

> ⚠️ **Errores frecuentes con `DATABASE_URL`:** reemplazar `[YOUR-PASSWORD]` **incluidos los
> corchetes**; codificar `@ # / ? : %` en la contraseña; no usar la conexión *Direct* (IPv6).

### 2.1 Verificar
En *Logs* deben aparecer:
```
Base de datos inicializada (Supabase aws-1-us-west-2.pooler.supabase.com:5432, SSL: require)
Proxies de confianza (trust proxy): 1
CORS origin: https://control-de-turnos-mu.vercel.app
```
```bash
curl https://control-de-turnos-backend.onrender.com/
# → {"message":"API de Registro de Horarios","version":"1.0.0"}

curl https://control-de-turnos-backend.onrender.com/salud     # incluye la base de datos
# → {"estado":"ok","db":"ok","ms":5}      (503 {"estado":"error",...} si la base no responde)
```
> El health check de Render usa `/` (no depende de la base) para no reiniciar el servicio en bucle
> si Supabase estuviera pausado.

### 2.2 (Recomendado) Verificar el certificado SSL de Supabase
1. Supabase → *Project Settings → Database → SSL Configuration* → **Download certificate**.
2. Convertir a una sola línea:
   `node -e "console.log(require('fs').readFileSync('prod-ca-2021.crt','utf8').trim().replace(/\n/g,'\\\\n'))"`
3. En Render: `DB_SSL=verify` y `DB_SSL_CA=<la línea>`. El log debe decir `SSL: verify`.

### 2.3 Plan gratuito: apagado por inactividad
- El servicio **se apaga tras 15 min sin tráfico** y tarda **~1 min** en volver. El kiosco lo
  tolera: espera hasta 75 s y muestra *"Iniciando el servidor, puede tardar hasta un minuto…"*.
- Cada *workspace* de Render tiene **750 h/mes** de instancias gratuitas, **compartidas** con
  `contabilidad-backend` (también gratuito). Si se agotan, Render suspende ambos servicios hasta el
  mes siguiente. Revisar el consumo en *Render → Billing*.
- **Mantener despierto en horario de operación:** `backend/.github/workflows/mantener-activo.yml`
  consulta **`/salud`** cada 10 min **solo de 9:00 a 2:59 hora de Chile** (~558 h/mes; deja ~190 h
  para `contabilidad-backend`). Fuera de ese horario la primera petición tarda ~1 min.
  - `/salud` ejecuta `SELECT 1` en la base: esa actividad diaria evita que **Supabase (plan
    gratuito) pause el proyecto** tras ~7 días sin uso. Si la base no responde, la ejecución falla
    y GitHub avisa por correo al dueño del repo.
  - Variables del repo backend (*Settings → Secrets and variables → Actions → Variables*):
    `BACKEND_URL` (configurada), `KEEPALIVE_INICIO` / `KEEPALIVE_FIN` (opcionales, por defecto 9 y 3).
  - Activa desde el 19-11-2026 (rama `main`). Se puede lanzar a mano en
    *Actions → Mantener backend activo → Run workflow*.
  - GitHub desactiva las tareas programadas de repos públicos tras 60 días sin actividad;
    reactivarla desde la pestaña *Actions* si ocurre.
- Para eliminar el apagado: plan pago de Render (desde ~US$7/mes por servicio).

---

## 3. Frontend en Vercel

- Proyecto `control-de-turnos` (equipo `dev-craft`), conectado a `control-de-turnos-front`.
  Framework *Create React App*; usa `frontend/vercel.json` (rutas del SPA + cabeceras de seguridad).
- Variable **`REACT_APP_API_URL`** = `https://control-de-turnos-backend.onrender.com`
  (Production y Preview). Es de **build**: si cambia, redesplegar.
- `frontend/.env.production` fija `INLINE_RUNTIME_CHUNK=false` (necesario para la CSP).

Verificar:
```bash
curl -sI https://control-de-turnos-mu.vercel.app/login | grep -iE "content-security|x-frame|permissions|referrer|nosniff"
curl -s -o /dev/null -w "%{http_code}\n" https://control-de-turnos-mu.vercel.app/asistencia   # → 200
```

> Si se agrega un dominio propio, actualizar `CORS_ORIGIN` en Render y volver a imprimir el QR.
> Revisar los términos del plan **Hobby** de Vercel: está pensado para uso personal/no comercial.

---

## 4. Puesta en marcha (una sola vez)

En https://control-de-turnos-mu.vercel.app:

1. **Iniciar sesión** con `ADMIN_USER` / `ADMIN_PASS` (las sesiones anteriores se invalidaron).
2. **Ubicación del local** — *Asistencia → Ubicación*: desde un **celular dentro del local**,
   **Usar mi ubicación actual**, revisar en Google Maps, radio **150–200 m**, probar con
   **Probar desde este dispositivo** en varios puntos, activar **Exigir ubicación** y **Guardar**.
3. **PINs** — *Asistencia → PIN del kiosco*: asignar y entregar (se muestran una sola vez).
4. **Cartel QR** — *Asistencia → Código QR → Imprimir cartel*; pegarlo en la entrada.
5. **Comunicar al personal**: escanear → nombre → PIN → Entrada/Salida; permitir la ubicación;
   la ubicación queda registrada al marcar.

## 5. Verificación final

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Abrir el panel desde un celular | Barra superior con menú ☰; sin desplazamiento lateral |
| 2 | Escanear el QR **dentro** del local y marcar Entrada | "Entrada registrada… · a X m del local" |
| 3 | Intentar marcar **fuera** del radio | "Estás a X m del local" |
| 4 | *Asistencia → En turno ahora* | Aparece el trabajador |
| 5 | Marcar Salida y *Convertir en turno* | Turno creado con sueldo calculado |
| 6 | `/dashboard` en ventana privada | Redirige a `/login` sin mostrar el panel |
| 7 | Panel abierto > 15 min | Sigue funcionando (renovación automática) |
| 8 | Cerrar sesión con dos pestañas | Ambas vuelven a `/login` |

## 6. Si algo falla

| Síntoma | Causa probable | Solución |
|---|---|---|
| Error de *CORS* en la consola | `CORS_ORIGIN` no coincide con la URL de Vercel | Corregir en Render (sin `/` final) |
| *Refused to connect … Content Security Policy* | `REACT_APP_API_URL` con `http://` | Usar `https://` y redesplegar Vercel |
| El kiosco tarda ~1 min la primera vez | Render gratuito despertando | Normal (§2.3) |
| Ambos backends de Render suspendidos | Se agotaron las 750 h del mes | Esperar al mes siguiente o plan pago |
| Falla la tarea "Respaldo de la base de datos" | Cambió la contraseña de la base o falta un secreto | Revisar `RESPALDO_DATABASE_URL` / `RESPALDO_CLAVE` (§7) |
| `/salud` responde 503 / la app no carga datos | Supabase pausó el proyecto por inactividad | Supabase → proyecto → **Resume project** (datos intactos) |
| `password authentication failed` | Corchetes / caracteres sin codificar en `DATABASE_URL` | Ver §2 |
| `self-signed certificate in certificate chain` | `DB_SSL=verify` sin el CA correcto | Revisar `DB_SSL_CA` |
| `ENETUNREACH` al conectar a la BD | Conexión *Direct* (IPv6) | Usar Session pooler |
| "La ubicación solo funciona en conexiones seguras" | Kiosco abierto por `http://` | Usar `https://` |
| "Permiso de ubicación denegado" | El trabajador rechazó el permiso | Ajustes del navegador → Ubicación → permitir |
| Todos rechazados por "ubicación poco precisa" | Equipos sin GPS | Celulares con GPS o subir la precisión mínima |
| 404 al recargar `/asistencia` | Vercel no usa `frontend/vercel.json` | Revisar Root Directory del proyecto |

## 7. Respaldos de la base de datos

El plan gratuito de Supabase **no tiene respaldos automáticos**. La tarea
`backend/.github/workflows/respaldo-db.yml` hace uno **cada día a las 04:30 (hora de Chile)**:

1. Exporta el esquema `public` con `pg_dump` 17.
2. Lo restaura en una base temporal y **compara las filas con producción** tabla por tabla.
3. Lo cifra con **AES-256** y verifica que se descifra a un archivo idéntico.
4. Guarda **solo el archivo cifrado** en GitHub durante **30 días**
   (*Actions → Respaldo de la base de datos → ejecución → Artifacts*).

Si algún paso falla, GitHub avisa por correo. También se puede lanzar a mano con **Run workflow**.

**Secretos** (*backend → Settings → Secrets and variables → Actions*): `RESPALDO_DATABASE_URL`
(Session pooler de Supabase) y `RESPALDO_CLAVE`.

> ⚠️ **La contraseña `RESPALDO_CLAVE` debe estar guardada en un gestor de contraseñas.** GitHub no la
> vuelve a mostrar y sin ella los respaldos no se pueden abrir. Si cambia la contraseña de la base
> de datos en Supabase, actualizar también `RESPALDO_DATABASE_URL`.

### Restaurar un respaldo

Requisitos: `gh` (GitHub CLI), `gpg` (incluido en Git Bash) y `pg_restore` 17 o superior.

```bash
cd backend

# 1. Ver los respaldos disponibles y descargar uno (por número de ejecución)
gh run list --workflow respaldo-db.yml --limit 10
gh run download <NUMERO_DE_EJECUCION> -D "$HOME/respaldos"   # fuera del repo

# 2. Restaurar (pide la contraseña RESPALDO_CLAVE y escribir RESTAURAR para confirmar)
bash scripts/restaurar-respaldo.sh "$HOME/respaldos/<nombre>/<nombre>.dump.gpg" "<DATABASE_URL destino>"
```

- La restauración reemplaza las tablas del esquema `public` del destino **en una sola transacción**:
  si algo falla, el destino queda como estaba.
- **Practica primero en una base local de prueba**, nunca directamente en producción. Por ejemplo:
  `psql "postgresql://postgres:<clave>@localhost:5432/postgres" -c "CREATE DATABASE prueba_restauracion"`
  y usar `postgresql://postgres:<clave>@localhost:5432/prueba_restauracion` como destino.
- Para recuperar producción: destino = la URL del Session pooler de Supabase. Después, las sesiones
  abiertas pueden requerir volver a iniciar sesión.
- Se descargan **fuera del repo** (`~/respaldos`) para no subirlos por accidente en un commit.
  Borrar la carpeta al terminar: aunque el archivo está cifrado, no conviene dejarlo suelto.

## 8. Volver a la versión anterior

- **Frontend:** Vercel → *Deployments* → despliegue anterior → **Instant Rollback**.
- **Backend:** Render → *Events/Deploys* → despliegue anterior → **Rollback**.
- **Base de datos:** sin acción; los cambios de esquema son aditivos y compatibles.

## 9. Pendientes y recomendaciones

- [x] PRs fusionados y Render desplegando desde `main` (19-11-2026).
- [ ] Revocar la API key de Render usada para el despliegue inicial (*Account Settings → API Keys*).
- [ ] Verificación SSL del certificado de Supabase (§2.2).
- [x] *Keep-alive* en horario de operación 9:00–3:00 (§2.3) — activo y verificado en GitHub Actions.
- [x] Evitar la pausa de Supabase por inactividad: `/salud` + tarea programada (19-11-2026).
- [x] Respaldo diario cifrado de la base (§7) — primera ejecución verificada el 19-11-2026.
- [ ] Hacer una restauración de prueba con tu contraseña en una base local (§7).
- [ ] Desarrollo local: `backend/.env` apunta a Supabase (producción); para desarrollar sin tocar
      datos reales: `docker compose -f docker-compose.yml -f docker-compose.local-db.yml up`.

---

## Anexo A: crear el despliegue desde cero

**Backend (Render):** *New → Blueprint* → repo `control-de-turnos-backend` (usa `render.yaml`), o
*New → Web Service* → Docker, plan Free, región Oregon; cargar las variables de §2.

**Frontend (Vercel):** *Add New → Project* → repo `control-de-turnos-front` → Create React App;
`REACT_APP_API_URL` = URL del backend; desplegar. Luego poner la URL de Vercel en `CORS_ORIGIN`
del backend.

**Migrar datos a una base nueva:**
```bash
cd backend
# DATABASE_URL (backend/.env) = base DESTINO
ORIGEN_DATABASE_URL="postgresql://usuario:clave@host:5432/base" npm run migrar:supabase -- --dry-run
ORIGEN_DATABASE_URL="postgresql://usuario:clave@host:5432/base" npm run migrar:supabase
```

## Anexo B: variables

**Backend:** `DATABASE_URL`, `JWT_SECRET`, `ADMIN_USER`, `ADMIN_PASS`, `CORS_ORIGIN`, `PORT`,
`JWT_EXPIRES_IN`, `REFRESH_IDLE_HOURS`, `SESSION_MAX_DAYS`, `DB_POOL_MAX`, `DB_SSL`, `DB_SSL_CA`,
`DB_SSL_CA_PATH`, `TRUST_PROXY`, `ZONA_HORARIA` (por defecto `America/Santiago`),
`ORIGEN_DATABASE_URL` (solo migración).

**Frontend:** `REACT_APP_API_URL` (build).
