# Guía de despliegue — Basel Venue · Control de turnos

Esta guía cubre el despliegue de la versión que incluye: módulo de asistencia (kiosco con PIN,
geolocalización y código QR), base de datos en Supabase, identidad visual de Basel Venue,
endurecimiento de seguridad, nueva política de sesiones y diseño responsivo.

## Arquitectura

```
 Celular / tablet / PC
          │  https
          ▼
 ┌─────────────────────┐  https (API)  ┌──────────────────────┐  SSL (pooler)  ┌─────────────┐
 │ Vercel — frontend   │ ────────────▶ │ Railway — backend    │ ─────────────▶ │  Supabase   │
 │ React (build)       │               │ Express (Docker)     │               │ PostgreSQL  │
 └─────────────────────┘               └──────────────────────┘               └─────────────┘
   repo: control-de-turnos-front         repo: control-de-turnos-backend
```

El repo `control-de-turnos` agrupa ambos como submódulos (`frontend/`, `backend/`) y contiene
`docker-compose.yml` para correr todo localmente.

> **Orden obligatorio:** 1) base de datos → 2) backend → 3) frontend. El frontend necesita la URL
> del backend en el momento del build, y el backend necesita la URL del frontend para CORS.

---

## 0. Antes de empezar

- [ ] Acceso a las cuentas de **GitHub**, **Supabase**, **Railway** y **Vercel**.
- [ ] La base de datos ya fue migrada a Supabase (18-11-2026). Si partes de cero, ver el
      [Anexo A](#anexo-a-migrar-datos-a-una-base-nueva).
- [ ] Ten a mano: la cadena de conexión del **Session pooler** de Supabase y las URLs públicas de
      Railway y Vercel (si aún no existen, se obtienen en los pasos 2 y 3).

## 1. Subir el código

Los cambios están en la rama `feature/asistencia` de los tres repos. Se integran a `main`
**primero en los submódulos y al final en el repo principal** (que apunta a commits de ellos):

```bash
# 1) Backend
cd backend
git push -u origin feature/asistencia
#    → abrir Pull Request feature/asistencia → main en GitHub, revisar y fusionar

# 2) Frontend
cd ../frontend
git push -u origin feature/asistencia
#    → abrir y fusionar el Pull Request

# 3) Repo principal (actualiza los punteros de los submódulos)
cd ..
git push -u origin feature/asistencia
#    → abrir y fusionar el Pull Request
```

> Si Railway y Vercel despliegan automáticamente desde `main`, **el despliegue arranca al fusionar**.
> Deja configuradas las variables de los pasos 2 y 3 **antes** de fusionar.

---

## 2. Backend en Railway

### 2.1 Servicio

El servicio debe estar conectado al repo **`control-de-turnos-backend`**, rama `main`.
Railway usa el `Dockerfile` automáticamente (`railway.json`). El puerto lo asigna Railway
(variable `PORT`); **no la definas**.

### 2.2 Variables de entorno

En *Railway → servicio → Variables*:

| Variable | Valor | Obligatoria |
|---|---|---|
| `DATABASE_URL` | Supabase → **Connect → Session pooler** (puerto **5432**). Ej.: `postgresql://postgres.<ref>:<contraseña>@aws-1-<región>.pooler.supabase.com:5432/postgres` | Sí |
| `JWT_SECRET` | Cadena aleatoria larga (ver abajo). **Nunca** reutilices la de desarrollo. | Sí |
| `ADMIN_USER` | Usuario del administrador (ej. `admin`). | Sí |
| `ADMIN_PASS` | Contraseña **larga y única** (16+ caracteres). | Sí |
| `CORS_ORIGIN` | URL exacta del frontend en Vercel, **sin `/` final**. Ej.: `https://control-turnos.vercel.app` | Sí |
| `JWT_EXPIRES_IN` | `15m` — **si existe con `8h`, cámbiala o bórrala** (tiene prioridad sobre el código). | Recomendada |
| `REFRESH_IDLE_HOURS` | `12` (horas de inactividad antes de pedir login). | Opcional |
| `SESSION_MAX_DAYS` | `7` (duración máxima de una sesión). | Opcional |
| `DB_POOL_MAX` | `10` (conexiones máximas a Supabase). | Opcional |
| `DB_SSL` / `DB_SSL_CA` | Verificación del certificado de Supabase (ver 2.4). | Recomendada |
| `TRUST_PROXY` | **No definir**: se detecta automáticamente en Railway (= 1). | No |

Generar `JWT_SECRET`:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"
```

> ⚠️ **Errores frecuentes con `DATABASE_URL`:**
> - La plantilla de Supabase trae `[YOUR-PASSWORD]`: hay que reemplazar **también los corchetes**.
> - Si la contraseña tiene `@ # / ? : %`, debe ir codificada (`@` → `%40`, `#` → `%23`, …).
> - No uses la conexión **Direct** (`db.<ref>.supabase.co`): es solo IPv6 y falla desde Railway.

### 2.3 Desplegar y verificar

Tras el despliegue, en *Deploy Logs* deben aparecer:

```
Base de datos inicializada (Supabase aws-1-...pooler.supabase.com:5432, SSL: require)
Servidor iniciado en puerto ...
Proxies de confianza (trust proxy): 1
```

Comprobación rápida (reemplaza la URL):

```bash
curl https://TU-BACKEND.up.railway.app/
# → {"message":"API de Registro de Horarios","version":"1.0.0"}
```

> Al arrancar, el backend **actualiza el esquema solo** (columnas de asistencia/ubicación, tabla de
> configuración, sesiones con hash, RLS). Todos los cambios son **aditivos**; no se pierden datos.
> Las sesiones abiertas del formato anterior se invalidan: **el admin deberá iniciar sesión otra vez**.

### 2.4 (Recomendado) Verificar el certificado SSL de Supabase

Por defecto la conexión va cifrada pero sin verificar el certificado (el log lo advierte). Para
verificarlo:

1. Supabase → *Project Settings → Database → SSL Configuration* → **Download certificate**.
2. Convierte el archivo a una sola línea con `\n`:
   ```bash
   node -e "console.log(require('fs').readFileSync('prod-ca-2021.crt','utf8').trim().replace(/\n/g,'\\\\n'))"
   ```
3. En Railway: `DB_SSL=verify` y `DB_SSL_CA=<la línea obtenida>`. Redespliega.
4. El log debe decir `SSL: verify` y desaparece la advertencia.

---

## 3. Frontend en Vercel

### 3.1 Proyecto

- **Root Directory:** `frontend` (*Project Settings → Build and Deployment → Root Directory*).
  Así Vercel usa `frontend/vercel.json` (rutas del SPA + cabeceras de seguridad).
- Framework: *Create React App* · Build: `npm run build` · Output: `build`.
- Si el proyecto está conectado al repo principal (`control-de-turnos`), los submódulos deben ser
  accesibles para Vercel (repos públicos o con acceso concedido).

### 3.2 Variable de entorno

| Variable | Valor |
|---|---|
| `REACT_APP_API_URL` | URL pública del backend en Railway, **sin `/` final**. Ej.: `https://TU-BACKEND.up.railway.app` |

> Es una variable **de build**: si la cambias, hay que **redesplegar** para que tome efecto.

### 3.3 Verificar

```bash
# Cabeceras de seguridad (deben aparecer las 5)
curl -sI https://TU-FRONTEND.vercel.app/login | grep -iE "content-security|x-frame|permissions|referrer|nosniff"

# Las rutas del SPA no deben dar 404 al recargar
curl -s -o /dev/null -w "%{http_code}\n" https://TU-FRONTEND.vercel.app/asistencia   # → 200
```

Si faltan las cabeceras, el Root Directory no es `frontend` o el proyecto no está leyendo
`frontend/vercel.json`.

> Si cambia la URL de Vercel (dominio propio, etc.), actualiza `CORS_ORIGIN` en Railway.

---

## 4. Puesta en marcha (una sola vez)

En el sistema **publicado** (no en `localhost`):

1. **Iniciar sesión** con `ADMIN_USER` / `ADMIN_PASS`.
2. **Ubicación del local** — *Asistencia → Ubicación*:
   - Desde un **celular dentro del local**, pulsa **Usar mi ubicación actual** y revisa el punto en
     Google Maps.
   - Radio sugerido: **150–200 m**. Usa **Probar desde este dispositivo** en distintos puntos del
     local (entrada, bodega) y ajusta.
   - Activa **Exigir ubicación para marcar** y **Guardar**.
3. **PINs** — *Asistencia → PIN del kiosco*: asigna un PIN a cada trabajador y entrégaselo
   (se muestra **una sola vez**).
4. **Cartel QR** — *Asistencia → Código QR → Imprimir cartel* y pégalo en la entrada.
   *(Generarlo desde el sistema publicado: si se genera en `localhost`, el QR no sirve.)*
5. **Comunicar al personal** cómo marcar (escanear → nombre → PIN → Entrada/Salida), que deben
   **permitir la ubicación** en el navegador y que la ubicación queda registrada al marcar.

---

## 5. Verificación final

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Abrir el panel desde un celular | Barra superior con menú ☰; sin desplazamiento lateral |
| 2 | Escanear el QR con un celular **dentro** del local, marcar Entrada | "Entrada registrada… · a X m del local" |
| 3 | Intentar marcar **fuera** del radio (p. ej. desde casa) | Rechazado: "Estás a X m del local" |
| 4 | Ver *Asistencia → En turno ahora* | Aparece el trabajador con su hora de entrada |
| 5 | Marcar Salida y *Convertir en turno* desde el historial | Se crea el turno con el sueldo calculado |
| 6 | Abrir `https://TU-FRONTEND/dashboard` en una ventana privada | Redirige a `/login` sin mostrar el panel |
| 7 | Dejar el panel abierto > 15 min y seguir usándolo | Sigue funcionando (renovación automática) |
| 8 | Cerrar sesión con dos pestañas abiertas | Ambas vuelven a `/login` |

---

## 6. Si algo falla

| Síntoma | Causa probable | Solución |
|---|---|---|
| El frontend no carga datos; en la consola: *CORS* | `CORS_ORIGIN` no coincide exactamente con la URL de Vercel | Corregir en Railway (sin `/` final, con `https://`) |
| Consola: *Refused to connect … Content Security Policy* | `REACT_APP_API_URL` apunta a `http://` | Usar la URL `https://` de Railway y redesplegar |
| `password authentication failed` en logs del backend | Contraseña con corchetes o caracteres sin codificar | Revisar `DATABASE_URL` (ver 2.2) |
| `self-signed certificate in certificate chain` | `DB_SSL=verify` sin el CA correcto | Revisar `DB_SSL_CA` o quitar `DB_SSL` |
| `ENETUNREACH` / timeout al conectar a la BD | Se usó la conexión *Direct* (IPv6) | Usar el **Session pooler** |
| `too many connections` / `max clients reached` | Pool grande o varias instancias | Bajar `DB_POOL_MAX` |
| El kiosco dice "La ubicación solo funciona en conexiones seguras" | Se abrió por `http://` | Abrir por `https://` |
| "Permiso de ubicación denegado" en un celular | El trabajador rechazó el permiso | Ajustes del navegador → Ubicación → permitir para el sitio; recargar |
| Todos son rechazados por "ubicación poco precisa" | Tablet/PC sin GPS (ubicación por red) | Usar celulares con GPS o subir "precisión mínima" en *Ubicación* |
| El admin queda bloqueado en el login tras intentos fallidos de otros | `TRUST_PROXY` forzado a 0 | Quitar la variable `TRUST_PROXY` en Railway |
| Al recargar `/asistencia` aparece 404 | Vercel no lee `frontend/vercel.json` | Root Directory = `frontend` (ver 3.1) |

## 7. Volver a la versión anterior

- **Frontend:** Vercel → *Deployments* → despliegue anterior → **Instant Rollback**.
- **Backend:** Railway → *Deployments* → despliegue anterior → **Redeploy**.
- **Base de datos:** no requiere acción. Los cambios de esquema son aditivos y compatibles con la
  versión anterior; solo las sesiones abiertas se invalidan (re-login).

## 8. Recomendaciones pendientes

- Activar la verificación SSL (paso 2.4).
- Revisar el plan de **respaldos** de Supabase y, si no incluye respaldos automáticos, programar
  exportaciones periódicas.
- Si habrá más de un administrador: cuentas individuales con roles y registro de cambios.
- Desarrollo local: `backend/.env` apunta hoy a Supabase (producción). Para desarrollar sin tocar
  datos reales usa `docker compose -f docker-compose.yml -f docker-compose.local-db.yml up`.

---

## Anexo A: migrar datos a una base nueva

Si se crea otro proyecto de Supabase o hay que repetir la migración desde una base PostgreSQL:

```bash
cd backend
# DATABASE_URL (en backend/.env) = base DESTINO (Supabase)
ORIGEN_DATABASE_URL="postgresql://usuario:clave@host:5432/base" npm run migrar:supabase -- --dry-run
ORIGEN_DATABASE_URL="postgresql://usuario:clave@host:5432/base" npm run migrar:supabase
```

El script crea el esquema, copia los datos conservando los IDs, ajusta las secuencias, verifica los
conteos y se niega a ejecutarse si el destino ya tiene datos.

## Anexo B: referencia de variables

**Backend** (`backend/.env.example`): `DATABASE_URL`, `JWT_SECRET`, `ADMIN_USER`, `ADMIN_PASS`,
`CORS_ORIGIN`, `PORT`, `JWT_EXPIRES_IN`, `REFRESH_IDLE_HOURS`, `SESSION_MAX_DAYS`, `DB_POOL_MAX`,
`DB_SSL`, `DB_SSL_CA`, `DB_SSL_CA_PATH`, `TRUST_PROXY`, `ZONA_HORARIA` (por defecto
`America/Santiago`), `ORIGEN_DATABASE_URL` (solo migración).

**Frontend:** `REACT_APP_API_URL` (build). `frontend/.env.production` fija
`INLINE_RUNTIME_CHUNK=false`, requerido por la política de seguridad de contenido.
