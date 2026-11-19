# Manual de usuario — Basel Venue · Control de turnos

Versión: 19-11-2026 · [Versión compartible](https://claude.ai/artifact/WmCmSXPDqR2MWWyQctPE5B)

## Contenido

1. [Introducción](#1-introducción)
2. [Iniciar y cerrar sesión](#2-iniciar-y-cerrar-sesión)
3. [Moverse por el panel](#3-moverse-por-el-panel)
4. [Dashboard](#4-dashboard)
5. [Agregar turno](#5-agregar-turno)
6. [Todos los turnos](#6-todos-los-turnos)
7. [Asistencia](#7-asistencia)
8. [PIN del kiosco](#8-pin-del-kiosco)
9. [Ubicación del local](#9-ubicación-del-local)
10. [Código QR](#10-código-qr)
11. [Cómo marcan los trabajadores](#11-cómo-marcan-los-trabajadores)
12. [Respaldos de la base de datos](#12-respaldos-de-la-base-de-datos)
13. [Preguntas frecuentes y solución de problemas](#13-preguntas-frecuentes-y-solución-de-problemas)

## 1. Introducción

El sistema de Control de turnos de Basel Venue registra los turnos del personal, calcula horas y sueldos, y controla la asistencia: los trabajadores marcan su entrada y salida desde el celular y el administrador lo revisa todo desde un panel web.

Este manual es para la persona que administra el sistema. La guía para los trabajadores está resumida en la sección 11.

| Dato | Valor |
| --- | --- |
| Panel de administración | [control-de-turnos-mu.vercel.app](https://control-de-turnos-mu.vercel.app) |
| Kiosco de los trabajadores | [control-de-turnos-mu.vercel.app/asistencia](https://control-de-turnos-mu.vercel.app/asistencia) |
| Usuario | `admin` |
| Contraseña | La variable `ADMIN_PASS` de la configuración del servidor (Render → Environment) |
| Funciona en | Computador, tablet y celular, con cualquier navegador actual |

## 2. Iniciar y cerrar sesión

1. Abre el panel y escribe el usuario y la contraseña.
2. Pulsa **Entrar**. Llegarás al panel.
3. Para salir, pulsa **Cerrar sesión** al final del menú.

La sesión se renueva sola mientras usas el panel. Vence si pasas **12 horas sin usarlo** y, como máximo, **7 días** después de iniciarla. Cuando vence, el login muestra "Tu sesión expiró" y basta con volver a entrar.

Si tienes el panel abierto en varias pestañas, cerrar sesión en una cierra todas.

## 3. Moverse por el panel

- **En computador:** el menú está siempre visible a la izquierda.
- **En celular o tablet:** pulsa el botón ☰ de la barra superior para abrir el menú; se cierra al elegir una sección o al tocar fuera.

| Sección | Para qué sirve |
| --- | --- |
| Dashboard | Resumen del mes en curso |
| Agregar turno | Empleados, empresas y turnos de cada empleado |
| Todos los turnos | Todos los turnos con filtros y exportación |
| Asistencia | Marcas del kiosco, PINs, ubicación y código QR |

## 4. Dashboard

El Dashboard resume el mes en curso; se actualiza cada vez que lo abres.

| Indicador | Qué muestra |
| --- | --- |
| Empleados / Empresas | Cantidad registrada en el sistema |
| Total turnos | Todos los turnos registrados, de cualquier fecha |
| Turnos este mes | Turnos con fecha en el mes actual |
| Horas este mes | Suma de horas trabajadas del mes |
| Sueldo este mes | Suma de lo que corresponde pagar por los turnos del mes |
| Top empleados del mes | Empleados ordenados por horas trabajadas, con turnos y monto |
| Turnos por empresa | Cantidad de turnos del mes por empresa |
| Últimos turnos registrados | Los 5 turnos cargados más recientemente |

## 5. Agregar turno

Esta pantalla tiene la lista de empleados (a la izquierda en computador, arriba en celular) y los turnos del empleado elegido.

**Empleados**

- **Crear:** pulsa **Agregar**, escribe el nombre y confirma.
- **Editar:** pulsa el lápiz junto al empleado para cambiar el nombre, la tarifa por hora y la tarifa por hora extra.
- **Eliminar:** pulsa la papelera. ⚠️ También se eliminan todos sus turnos y asistencias.

**Empresas:** pulsa **Empresas** en la barra de filtros para crear, renombrar o eliminar empresas. Al eliminar una empresa, sus turnos quedan como "Sin empresa".

**Turnos del empleado**

1. Elige el empleado y el mes (y, si quieres, una empresa).
2. Pulsa **Agregar turno** y completa fecha, hora de entrada y salida, evento, área y empresa.
3. Guarda. El sistema calcula solo las horas y el sueldo.

Cada turno se puede **editar**, **duplicar** a otra fecha (por ejemplo, el mismo evento varios días) o **eliminar**.

**Cómo se calcula:** las primeras **8 horas** son horas base, pagadas con la tarifa por hora; lo que pase de 8 son horas extra, con la tarifa por hora extra. Si la salida es anterior a la entrada, se entiende que el turno cruzó la medianoche.

**Exportar:** con un rango de fechas, **Excel** o **PDF** descargan los turnos del empleado con sus totales.

## 6. Todos los turnos

Muestra los turnos de todos los empleados en una tabla, 50 por página, con los totales del periodo al final.

1. Filtra por **empleado**, **empresa** y un rango de **fechas** (desde y hasta); deja vacío lo que no quieras filtrar.
2. Pulsa **Buscar**.
3. Revisa los totales: horas trabajadas, horas extra, valor de horas extra, valor fijo y sueldo total.
4. Pulsa **Excel** o **PDF** para descargar lo filtrado, por ejemplo para pagar una quincena o enviarlo a una productora.

En celular la tabla se desliza hacia los lados; la columna del empleado queda fija.

## 7. Asistencia

Aquí se ven las marcas de entrada y salida que hacen los trabajadores en el kiosco. Arriba hay tres indicadores: personas trabajando ahora, asistencias abiertas hace más de 12 horas y trabajadores con PIN.

**En turno ahora:** quienes marcaron entrada y aún no salida, con el tiempo que llevan. Una tarjeta en ámbar indica más de 12 horas abierta: probablemente olvidó marcar la salida. **Marcar salida ahora** la cierra con la hora actual; el lápiz permite corregir la hora.

**Historial**

1. Elige el rango de fechas, el empleado y el estado (en curso o cerradas) y pulsa **Buscar**.
2. Cada fila muestra entrada, salida, horas y estado. La etiqueta **kiosco** indica que la marcó el trabajador; **manual**, que la registró o corrigió el administrador. El ícono de ubicación muestra a cuántos metros del local se marcó y abre el punto en el mapa.
3. Acciones de cada fila: **corregir** horas u observación, **eliminar**, y **Turno** para convertirla en turno.
4. **Excel** descarga el historial filtrado, incluidas las distancias.

**Convertir en turno:** en una asistencia cerrada, pulsa **Turno**, indica evento, área y empresa y pulsa **Crear turno**. Se crea el turno con el sueldo calculado y la asistencia queda bloqueada para edición.

**Registro manual:** para quien no pudo marcar (sin celular, sin batería), pulsa **Registro manual**, elige el empleado, la entrada y, si la sabes, la salida, y agrega una observación.

## 8. PIN del kiosco

Cada trabajador marca con un PIN personal de 4 a 6 dígitos. Solo quienes tienen PIN aparecen en el kiosco.

1. Ve a **Asistencia → PIN del kiosco**.
2. Pulsa **Asignar PIN** (o **Reasignar**) junto al trabajador. El sistema propone un PIN aleatorio; puedes cambiarlo o generar otro.
3. Pulsa **Guardar** y entrégale el PIN en ese momento.

- El PIN se muestra **una sola vez**: el sistema lo guarda cifrado y no puede volver a mostrarlo. Si el trabajador lo olvida, asígnale uno nuevo.
- Tras **5 intentos fallidos**, el PIN se bloquea 15 minutos. Reasignarlo lo desbloquea al instante.
- **Quitar** saca al trabajador del kiosco sin borrar su historial.

## 9. Ubicación del local

Con la ubicación activa, solo se puede marcar entrada o salida estando dentro del radio del local. Las correcciones del administrador no la necesitan.

1. Desde un **celular dentro del local**, entra a **Asistencia → Ubicación**.
2. Pulsa **Usar mi ubicación actual** y revisa el punto con **Ver el punto en Google Maps**.
3. Define el **radio permitido**: se recomiendan 150 a 200 metros, porque en interiores el GPS puede desviarse decenas de metros.
4. Deja la **precisión mínima** en 150 metros: rechaza ubicaciones demasiado imprecisas.
5. Usa **Probar desde este dispositivo** en varios puntos del local (entrada, bodega, barra) y ajusta el radio si alguno queda fuera.
6. Activa **Exigir ubicación para marcar** y pulsa **Guardar**.

Al desactivarla se conservan las coordenadas guardadas. La ubicación es una barrera disuasiva: alguien con conocimientos técnicos podría falsearla, por eso cada marca registra a qué distancia se hizo.

## 10. Código QR

El QR lleva directo al kiosco para que los trabajadores lo abran con la cámara del celular.

1. En **Asistencia**, pulsa **Código QR**.
2. Elige una opción:
    - **Copiar** el enlace, para enviarlo por WhatsApp.
    - **Descargar PNG**, una imagen de alta resolución.
    - **Imprimir cartel**, una hoja A4 con el logo, el QR y las instrucciones.
3. Pega el cartel en la entrada o donde marquen los trabajadores.

Genera el QR siempre desde el sistema publicado. Si aparece el aviso de "entorno local", ese QR no funcionará en otros celulares.

## 11. Cómo marcan los trabajadores

Los trabajadores no necesitan usuario ni contraseña: solo el enlace del kiosco (o el QR) y su PIN.

1. Escanean el QR o abren el enlace. Conviene agregarlo a la pantalla de inicio del celular.
2. Tocan su nombre (o lo buscan).
3. La primera vez, el navegador pide permiso de ubicación: deben pulsar **Permitir**.
4. Escriben su PIN y pulsan **Entrada** al llegar o **Salida** al irse.
5. Ven la confirmación con la hora, la distancia al local y, al salir, el tiempo trabajado.

La hora la pone el servidor, no el celular. Si alguien deja el PIN a medio escribir, la pantalla vuelve al inicio a los 30 segundos.

| Mensaje que puede ver | Qué hacer |
| --- | --- |
| "Estás a X m del local" | Marcar dentro del local |
| "Ubicación poco precisa" | Activar el GPS o acercarse a una ventana y reintentar |
| "Permiso de ubicación denegado" | Permitir la ubicación en los ajustes del navegador y recargar |
| "Ya tienes una entrada registrada" | Marcar Salida primero |
| "PIN bloqueado" | Esperar 15 minutos o pedir un PIN nuevo |
| "Iniciando el servidor…" | Esperar hasta un minuto (ver sección 13) |

## 12. Respaldos de la base de datos

Cada día a las 04:30 (hora de Chile) se hace un respaldo automático de toda la información. Cada respaldo se compara con producción, se cifra y se guarda 30 días en GitHub; si alguno falla, GitHub avisa por correo.

**Dónde están:** en el repositorio del backend, [Actions → Respaldo de la base de datos](https://github.com/juanes16122002/control-de-turnos-backend/actions/workflows/respaldo-db.yml). Cada ejecución tiene el archivo cifrado en la sección **Artifacts**. Con **Run workflow** se puede hacer un respaldo a mano.

**Verificar:** la ejecución debe tener una marca verde, y el paso "Restaurar en una base temporal y comparar con producción" debe mostrar "ok" en cada tabla.

**Restaurar en pgAdmin:**

1. Descarga el archivo desde **Artifacts** y descomprímelo.
2. Descifra en Git Bash: `gpg --output respaldo.dump --decrypt <archivo>.dump.gpg` (pide la contraseña de respaldos).
3. En pgAdmin crea una base nueva, por ejemplo `prueba_restauracion`, y usa **Restore…** con formato *Custom or tar*, el archivo `respaldo.dump` y, en *Do not save*, **Owner** y **Privileges**.
4. El aviso `schema "public" already exists` es normal.
5. Borra `respaldo.dump` al terminar: está sin cifrar.

⚠️ La contraseña de respaldos (`RESPALDO_CLAVE`) debe estar guardada en un gestor de contraseñas: sin ella los respaldos no se pueden abrir. Restaura siempre primero en una base de prueba, nunca directamente en producción.

## 13. Preguntas frecuentes y solución de problemas

| Situación | Causa | Qué hacer |
| --- | --- | --- |
| El panel o el kiosco tarda cerca de un minuto en cargar | El servidor gratuito se duerme entre las 3:00 y las 9:00 sin uso | Esperar; luego responde normal |
| Un trabajador olvidó marcar la salida | Ya salió del local y no puede marcarla | **Asistencia → En turno ahora → Marcar salida ahora**, o corregir la hora |
| Un trabajador olvidó su PIN o lo bloqueó | El PIN no se puede volver a mostrar | **PIN del kiosco → Reasignar** |
| Nadie puede marcar por la ubicación | Punto o radio mal configurados | Revisar **Ubicación** y probar desde el local |
| Me pide iniciar sesión otra vez | Pasaron 12 horas sin uso o 7 días desde el login | Volver a entrar |
| No recuerdo la contraseña del admin | Está en la configuración del servidor | Revisar `ADMIN_PASS` en Render → Environment |
| La app no carga datos y el aviso de respaldo falla | La base de datos pudo pausarse | Entrar a Supabase y pulsar **Resume project** |
| Eliminé algo por error | Las eliminaciones no se deshacen | Recuperarlo desde un respaldo (sección 12) |
