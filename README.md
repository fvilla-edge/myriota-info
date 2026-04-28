## 1) Que hace este sample

El demo:

- obtiene ubicacion GNSS,
- obtiene temperatura interna y voltaje de bateria,
- arma un payload binario fijo,
- agenda uplinks periodicos por HyperPulse NTN,
- procesa downlinks por URC,
- permite configurar runtime y persistir configuracion (`cfg period`, `cfg gnss_fix`).


## 2) Estructura del sample y rol de archivos

### Nucleo

- `src/main.c`: secuencia de arranque, init, GNSS inicial, arranque de scheduler.
- `src/periodic_uplink.c`: worker periodico, construccion de payload y envio.
- `src/modem.c`: wrappers de comandos AT hacia la libreria HyperPulse.
- `src/modem_urc.c`: callback de URCs y dispatch por tipo (`#MLOCATION`, `#MRECV`).
- `src/app_gnss.c`: requests de ubicacion y cola de fixes GNSS.
- `src/configuration.c`: settings persistentes + comandos shell `cfg`.
- `src/hardware_controls.c`: LEDs, console on/off y ahorro energetico por hardware.

### Configuracion y build

- `CMakeLists.txt`: fuentes compiladas del sample.
- `prj.conf`: features de Zephyr/NCS habilitadas (shell, settings, littlefs, logs, PM).
- `sysbuild.conf`: politica de merge de hex.
- `pm_static.yml`: layout fijo de particiones para HyperPulse y littlefs.
- `boards/*.overlay` y `boards/*.conf`: ajustes por board.

### Herramientas de decodificacion

- `parser.js`: parser para backend/TagoIO.
- `unpack.py`: parser CLI local en Python.

## 3) Flujo de ejecucion (runtime) paso a paso

## Arranque (`main.c`)

1. Imprime version.
2. Verifica que `hyperpulse_lib` este inicializada.
3. Ejecuta `config_init()`.
4. Arranca shell serial (`shell_start(...)`).
5. Ejecuta `hardware_control_init()`.
6. Señaliza init con blink de `LED_1`.
7. Espera fix GNSS valido (`app_gnss_wait_for_valid_fix()`).
8. Habilita notificaciones downlink (`AT#MRECV=1`).
9. Arranca scheduler periodico (`periodic_uplink_init_and_start()`).
10. Señaliza ready con doble blink de `LED_1`.

## GNSS y URCs

- La app solicita location via `modem_request_location(...)`.
- El modem responde por URC `#MLOCATION`.
- `modem_urc.c` parsea esa URC y empuja el fix en `k_msgq` de `app_gnss`.
- `app_gnss_get_fix(...)` consume ese fix.

## Uplink periodico

El worker:

1. toma timestamp de inicio,
2. destella LED de actividad,
3. completa el payload (`sequence`, `time`, `lat`, `lon`, `elev`, `temp`, `vbat`),
4. loggea payload y hex,
5. envia con `AT#MSEND`,
6. reprograma proximo ciclo en base a `period` y tiempo transcurrido.

## 4) Payload binario (contrato critico)

Estructura actual:

- `uint32_t sequence_number`
- `uint32_t time`
- `int32_t latitude` (escala `1e-7`)
- `int32_t longitude` (escala `1e-7`)
- `int16_t elevation_m`
- `int8_t temperature_celsius`
- `uint16_t battery_voltage_mv`

Formato little-endian packed.  
`unpack.py` usa `MESSAGE_FORMAT = "<IIiihbH"`, por lo que debe mantenerse sincronizado con el C.

## 5) Configuracion runtime y persistencia

En `configuration.c`:

- namespace settings: `myriota/*`,
- defaults: `period=3600`, `gnss_fix=false`,
- persiste en littlefs (`/edge/settings`),
- expone shell:
  - `cfg period` / `cfg period <segundos>`
  - `cfg gnss_fix` / `cfg gnss_fix <0|1>`

Comportamiento:

- `period > 0`: scheduler activo.
- `period == 0`: scheduler detenido.
- `gnss_fix=1`: intenta fix nuevo para cada mensaje (mas consumo).
- `gnss_fix=0`: puede reutilizar ultimo fix (menos consumo).

## 6) Parametros de tuning mas importantes

- `cfg period`: frecuencia de uplink (impacta consumo y granularidad).
- `cfg gnss_fix`: frescura de posicion vs bateria.
- timeouts GNSS en `app_gnss.c`:
  - `LOCATION_REQUEST_TIMEOUT_SECS`
  - `LOCATION_UPDATE_TIMEOUT_SECS`
- limites en `modem.h`:
  - `MODEM_UPLINK_MSG_SIZE`
  - tamanos de buffers AT
- `prj.conf`: nivel de logs/shell (impacto energetico y visibilidad).
- `pm_static.yml`: particiones fijas (cambios requieren cuidado/provisioning coherente).

## 7) Donde tocar segun el objetivo

- **Agregar/editar campos del payload**: `src/periodic_uplink.c` + `parser.js` + `unpack.py`.
- **Nueva configuracion runtime persistente**: `src/configuration.c`.
- **Cambiar politica GNSS o timeouts**: `src/app_gnss.c`.
- **Procesar nuevas URC**: `src/modem_urc.c`.
- **Modificar señales LED/ahorro de energia/hardware**: `src/hardware_controls.c` + overlays de board.

---

