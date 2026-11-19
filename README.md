# Basel Venue · Control de turnos

Sistema web para registrar los turnos del personal, calcular horas y sueldos, y controlar la asistencia: los trabajadores marcan entrada y salida desde el celular (kiosco con PIN, ubicación y código QR) y el administrador lo revisa todo desde un panel.

| | |
| --- | --- |
| Panel de administración | https://control-de-turnos-mu.vercel.app |
| Kiosco de los trabajadores | https://control-de-turnos-mu.vercel.app/asistencia |
| API | https://control-de-turnos-backend.onrender.com |

## Documentación

- [Manual de usuario](docs/MANUAL_USUARIO.md): cómo usar el panel, la asistencia, los PINs, la ubicación, el código QR y los respaldos.
- [Documentación técnica](docs/DOCUMENTACION_TECNICA.md): arquitectura, código, modelo de datos, API, seguridad, configuración, desarrollo local, pruebas y operación.
- [Guía de despliegue](DESPLIEGUE.md): Vercel, Render y Supabase paso a paso, puesta en marcha y solución de problemas.

## Repositorios

Este repositorio agrupa los otros dos como submódulos:

- [`backend/`](https://github.com/juanes16122002/control-de-turnos-backend): API Express 5 + PostgreSQL (Render).
- [`frontend/`](https://github.com/juanes16122002/control-de-turnos-front): aplicación React 19 (Vercel).

```bash
git clone --recurse-submodules https://github.com/juanes16122002/control-de-turnos.git
```

Para correrlo en local, ver la sección [Desarrollo local](docs/DOCUMENTACION_TECNICA.md#9-desarrollo-local).
