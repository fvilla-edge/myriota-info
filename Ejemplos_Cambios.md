# Ejemplos de cambios


## Ejemplo A: agregar al payload un campo de humedad (sensor)

Supuesto: se tiene un sensor de humedad en Zephyr (driver `sensor`) y querés enviar **humedad relativa en %** como `int8_t` (0–100), o `int16_t` si preferís décimas.

### A1) Cambio en C (`src/periodic_uplink.c`)

**Idea:** extender `struct uplink_message_t` y rellenar el campo en `populate_uplink_message()`.

```diff
 struct uplink_message_t {
 	uint32_t sequence_number;
 	uint32_t time;
 	int32_t latitude;
 	int32_t longitude;
 	int16_t elevation_m;
 	int8_t temperature_celsius;
 	uint16_t battery_voltage_mv;
+	int8_t humidity_percent; /* 0..100, o INT8_MIN si no hay dato */
 } __attribute__((packed));
```

En `populate_uplink_message()` (después de temperatura o al final), **leer el sensor** y asignar:

```diff
+	/* Pseudocódigo: reemplazar por tu driver real (sensor_channel_get, etc.) */
+	int8_t rh = INT8_MIN;
+	if (leer_humedad_del_sensor(&rh) == 0) {
+		msg->humidity_percent = rh;
+	} else {
+		msg->humidity_percent = INT8_MIN;
+	}
```

**Notas:**

- El `sizeof(msg)` ya incluye el nuevo campo: `modem_schedule_uplink_message((const char *)&msg, sizeof(msg))` sigue siendo correcto si no se cambia la firma.
- Verificar que el tamaño total siga siendo `<= MODEM_UPLINK_MSG_SIZE` (definido en `include/modem.h`).

### A2) Cambio en Python (`unpack.py`)

Actualizar el formato `struct` para reflejar el nuevo byte al final:

```diff
-MESSAGE_FORMAT = "<IIiihbH"
+MESSAGE_FORMAT = "<IIiihbHb"   /* ejemplo: ... uint16_t vbat, int8_t rh */
 MESSAGE_SIZE = struct.calcsize(MESSAGE_FORMAT)
```

Y en el `unpack(...)`:

```diff
     (
         sequence_number,
         time,
         latitude,
         longitude,
         elevation,
         temperature,
         battery_voltage,
+        humidity_percent,
     ) = struct.unpack(MESSAGE_FORMAT, data[:MESSAGE_SIZE])
```

En el diccionario JSON de salida, agregar la clave nueva.

### A3) Cambio en TagIO (`parser.js`)

Agregar lectura por offset (el offset depende del orden exacto del struct). Si el nuevo campo va **al final** tras `battery_voltage` (2 bytes LE), el offset sería **21** (0-based: byte 21).

```diff
         data.push({ variable: 'battery_voltage', value: buffer.readUInt16LE(19), unit : "mV" });
+        data.push({ variable: 'humidity_percent', value: buffer.readInt8(21), unit : "%" });
```

**Importante:** si se usa `readInt16LE` para batería en tu copia, los offsets cambian. Siempre recalcular offsets con el struct final.

---

## Ejemplo B: nuevo comando `cfg` persistente (habilitar humedad en uplink)

Objetivo: `cfg humidity <0|1>` para no leer el sensor si no hace falta.

### B1) `src/configuration.c`

1. **Nueva key** en settings, por ejemplo `"humidity_en"` bajo `myriota/`.
2. **Variable estática** `static bool humidity_enabled = true;` + mutex si querés simetría con el resto.
3. En `myriota_settings_set(...)`, agregás branch para cargar desde flash al boot.
4. **Getter** `config_get_humidity_enabled(void)` (podés declararlo en `include/configuration.h`).
5. **Setter** que guarde con `settings_save_one` y actualice la variable en RAM.
6. **Comando shell** nuevo dentro de `sub_cfg`:

```diff
 SHELL_STATIC_SUBCMD_SET_CREATE(sub_cfg,
 	SHELL_CMD(period, NULL, "...", sh_config_period),
 	SHELL_CMD(gnss_fix, NULL, "...", sh_config_gnss_fix),
+	SHELL_CMD(humidity, NULL, "Get/set humidity reporting (0=off,1=on)", sh_config_humidity),
 	SHELL_SUBCMD_SET_END
 );
```

Uso esperado:

- `cfg humidity` → imprime `enabled`/`disabled`
- `cfg humidity 0|1` → persiste y responde `OK`

### B2) Consumir la config en uplink

En `populate_uplink_message()`:

```diff
+	if (!config_get_humidity_enabled()) {
+		msg->humidity_percent = INT8_MIN;
+		return; /* o saltar solo el bloque de sensor */
+	}
```

---

## Ejemplo C: nueva URC `#MSENSOR:` (simulación de telemetría del modem)

Supuesto: el modem manda una línea tipo:

`#MSENSOR: rh=63,temp=22`

### C1) `src/modem_urc.c`

1. Crear `static void process_urc_sensor(const char *const response)`.
2. Parsear con `sscanf` o una función dedicada (validar prefijo).
3. Guardar en variables globales estáticas o en una pequeña estructura accesible por `periodic_uplink.c` (mejor: API en `sensor_cache.h/.c` si crece).

Registro en la tabla:

```diff
 const process_urc_response_entry_t urc_response_handlers[] = {
 	{"#MLOCATION:", process_urc_location},
 	{"#MRECV:", process_urc_message_received},
+	{"#MSENSOR:", process_urc_sensor},
 };
```

### C2) (Opcional) helper en `modem.c`

Si el parseo es largo, mover la lógica a:

- `int modem_parse_sensor_urc(const char *response, sensor_urc_t *out);`

y declarar en `modem.h`.

### C3) Uso en payload

En `populate_uplink_message()`, en vez de leer I2C, podés preferir el último valor recibido por URC si está “fresco” (con timestamp local `k_uptime_get()`).

---

## Resumen de archivos por ejemplo

| Objetivo | Archivos típicos |
|----------|-------------------|
| Campo nuevo en payload | `src/periodic_uplink.c`, `unpack.py`, `parser.js`, (`README.md`) |
| Nuevo `cfg` persistente | `src/configuration.c`, `include/configuration.h`, consumidor (`periodic_uplink.c`) |
| Nueva URC | `src/modem_urc.c`, (`src/modem.c` / `modem.h` si factorizás parseo) |


