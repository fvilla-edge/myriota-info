# Playbook de modificaciones (`samples/demo`)

Si se toca payload, actualizar siempre parsers (`parser.js`, `unpack.py`).

---

## Receta 1: Agregar un campo nuevo al payload

Objetivo ejemplo: agregar `gnss_accuracy` al mensaje uplink.

### Archivos a tocar

- `src/periodic_uplink.c`
- `parser.js`
- `unpack.py`
- opcional: `README.md` (tabla de payload)

### Paso 1 - Extender struct en C

En `src/periodic_uplink.c`, agregar el campo al final de `struct uplink_message_t`.

Recomendacion: agregar al final minimiza riesgo de romper offsets previos durante pruebas iniciales.

### Paso 2 - Poblar el nuevo campo

En `populate_uplink_message(...)`, setea el valor desde la fuente correcta:

- si viene del fix GNSS: `location.accuracy`,
- si no hay dato valido: valor sentinel (ej. `0xFF` o `-1` segun tipo).

### Paso 3 - Validar tamaño total

El mensaje cambia de longitud.  
Asegurar que:

- sigue siendo `<= MODEM_UPLINK_MSG_SIZE`,
- la impresion hex y `AT#MSEND=<len>` usan el nuevo `sizeof(msg)` correcto.

### Paso 4 - Actualizar `unpack.py`

En `unpack.py`:

- actualizar `MESSAGE_FORMAT` para incluir el nuevo tipo,
- extraer el campo nuevo en el unpack,
- agregar el valor al JSON de salida.

### Paso 5 - Actualizar `parser.js`

En `parser.js`:

- agregar lectura con offset correcto (`readInt8`, `readUInt8`, `readInt16LE`, etc.),
- agregar variable al array `data`.

### Paso 6 - Validacion minima

Checklist:

1. El log muestra payload con longitud nueva.
2. `unpack.py` decodifica sin error.
3. `parser.js` decodifica sin error.
4. Campo nuevo coincide con el valor esperado.

---

## Receta 2: Agregar un nuevo parametro de configuracion `cfg`

Objetivo ejemplo: `cfg temp_enable <0|1>` para habilitar/deshabilitar envio de temperatura.

### Archivos a tocar

- `src/configuration.c`
- `include/configuration.h` (si necesitas getter publico)
- `src/periodic_uplink.c` (consumir la config)

### Paso 1 - Definir clave y variable

En `configuration.c`:

- agregar sub-key settings (ej. `temp_enable`),
- agregar variable estatica con default (ej. `true`),
- agregar mutex si corresponde.

### Paso 2 - Persistencia settings

Actualizar:

- `myriota_settings_set(...)` para leer la nueva key,
- funcion setter para guardar con `settings_save_one(...)`,
- getter para lectura segura.

### Paso 3 - Comando shell

Agregar comando dentro de `SHELL_STATIC_SUBCMD_SET_CREATE(sub_cfg, ...)`:

- `cfg temp_enable`
- `cfg temp_enable <0|1>`

Con validacion de argumentos y respuesta `OK`.

### Paso 4 - Aplicar en runtime

En `periodic_uplink.c`, usar el getter:

- si esta deshabilitado, no leer temp del modem o mandar sentinel.

### Paso 5 - Validacion minima

1. `cfg temp_enable` muestra estado.
2. Cambia valor, responde `OK`.
3. Reboot y valor persiste.
4. Payload refleja habilitado/deshabilitado.

---

## Receta 3: Agregar soporte para una nueva URC

Objetivo ejemplo: procesar `#MALERT:`.

### Archivos a tocar

- `src/modem_urc.c`
- opcional: `src/modem.c` + `include/modem.h` (si parseo necesita helper dedicado)

### Paso 1 - Crear handler

En `modem_urc.c`, agregar funcion:

- parsea respuesta,
- valida formato,
- toma accion (log, actualizar estado, encolar evento, etc.).

### Paso 2 - Registrar en tabla de dispatch

Agregar entrada en `urc_response_handlers[]` con el prefijo exacto:

- `{"#MALERT:", process_urc_alert}`

### Paso 3 - Manejo robusto de errores

Si parse falla:

- no crashear,
- loggear error claro,
- retornar sin bloquear otros URCs.

### Paso 4 - Validacion minima

1. Inyectar/simular URC en formato valido.
2. Ver handler correcto en logs.
3. Simular URC invalida y verificar manejo de error sin side effects.

---

## Plantilla de pruebas rapidas por cada cambio


1. Build limpio (`--pristine`).
2. Flash y boot completo.
3. Verificar `cfg` y logs principales.
4. Capturar al menos un payload hex.
5. Decodificar con `unpack.py`.
6. Confirmar que no aparezcan errores AT repetidos.

---



