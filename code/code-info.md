<div align="center">

# KEPLER RC — FIRMWARE

</div>

---

## Descripción

Firmware de control radiocontrolado para el minisumo **Kepler**. Lee un tren **PPM** de 8 canales proveniente de un receptor Radiolink **R8FM**, lo decodifica por interrupción, aplica una cadena de acondicionamiento (zona muerta, expo, suavizado) y lo convierte en dos salidas PWM diferenciales hacia los drivers TB6612FNG.

El sistema es **RC en su estado actual**: no hay lectura de sensores ni lógica autónoma implementada, aunque el hardware ya reserva las entradas para sensores de línea. 

---

## Receptor R8FM

- **Modelo:** Radiolink R8FM
- **Transmisor:** Radiolink T8S
- **Página oficial:** https://www.radiolink.com/r8fm
- **Canales:** 8
- **Salida usada:** PPM (una sola línea de señal)

### Conexión

| Pin del receptor | Destino en la PCB | Nota |
| :--- | :--- | :--- |
| Señal PPM | `RECEPTOR` → `GPIO21` | Configurado con `INPUT_PULLUP` |
| VCC | `RECEPTOR` → 3.3 V | — |
| GND | `RECEPTOR` → GND | — |


## Protocolo PPM

PPM (*Pulse Position Modulation*) transmite los 8 canales **secuencialmente en un solo cable**, en lugar de un cable por canal. Cada canal se codifica como el tiempo entre dos flancos consecutivos.

```
 ┌─┐   ┌─┐    ┌─┐  ┌─┐        ┌─┐   ┌─┐            ┌─┐
 │ │   │ │    │ │  │ │        │ │   │ │            │ │
─┘ └───┘ └────┘ └──┘ └── ... ─┘ └───┘ └────────────┘ └──
   │←CH1→│←CH2 →│←CH3→│       │←CH8→│←── SYNC ────→│
    1-2ms  1-2ms  1-2ms                  > 4 ms
   │←──────────── TRAMA ~20-22 ms ──────────────→│
```

| Elemento | Valor | Constante |
| :--- | :--- | :--- |
| Canales por trama | 8 | `CHANNELS` |
| Ancho por canal | 1000 – 2000 µs | — |
| Centro nominal | ~1500 µs | — |
| Umbral de sincronía | > 4000 µs | `FRAME_GAP` |
| Periodo de trama | ~20 – 22 ms | — |

### Decodificación

La lectura se hace por **interrupción en flanco de bajada** sobre `GPIO21`. En cada flanco se mide el tiempo transcurrido desde el anterior:

- Si ese intervalo **supera `FRAME_GAP`** → es el hueco de sincronía, se reinicia el índice de canal a 0.
- Si no → el intervalo es el valor de un canal, se guarda y se avanza el índice.

La ISR está marcada con `IRAM_ATTR` para ejecutarse desde RAM interna, lo cual es obligatorio en ESP32 cuando la interrupción puede dispararse durante operaciones de flash.

---

## Mapa de canales

| Canal | Índice en buffer | Control físico en el T8S | Uso en el firmware |
| ---: | :---: | :--- | :--- |
| **1** | `[0]` | Joystick derecho — Izq/Der | ✅ **Giro** |
| 2 | `[1]` | Joystick derecho — Arriba/Abajo | ⬜ Libre |
| **3** | `[2]` | Joystick izquierdo — Arriba/Abajo | ✅ **Acelerador** |
| 4 | `[3]` | Joystick izquierdo — Izq/Der | ⬜ Libre |
| **5** | `[4]` | Gatillo derecho | ✅ **Giro forzado** izq/der |
| **6** | `[5]` | Botón derecho | ✅ **Giro automático 90°** |
| 7 | `[6]` | Gatillo izquierdo | ⬜ Libre |
| 8 | `[7]` | Perilla / *round* | ⬜ Libre |

---

## ESP32-C3 Beetle

<img width="1280" height="686" alt="Pinout del Beetle ESP32-C3" src="https://github.com/user-attachments/assets/c4fe582c-7a3d-44e6-b954-cd28e80abf04" />

### Asignación de pines usada por el firmware

| Señal | GPIO | Dirección | Periférico |
| :--- | :---: | :--- | :--- |
| `PPM_PIN` | 21 | Entrada | Interrupción externa (`FALLING`) |
| `PWMA` | 5 | Salida | LEDC canal 0 |
| `AIN1` | 4 | Salida | GPIO digital |
| `AIN2` | 6 | Salida | GPIO digital |
| `PWMB` | 2 | Salida | LEDC canal 1 |
| `BIN1` | 8 | Salida | GPIO digital |
| `BIN2` | 9 | Salida | GPIO digital |

---

## Cadena de procesamiento

Cada eje de control atraviesa cinco etapas antes de llegar al mezclador:

| # | Etapa | Entrada | Salida | Propósito |
| :---: | :--- | :--- | :--- | :--- |
| 1 | **Zona muerta** | 1000–2000 µs | — | Ignora el ruido del stick centrado. Ventana **1470–1530 µs** (±30 µs) |
| 2 | **Mapeo lineal** | µs | −255…+255 | Escala cada mitad del recorrido por separado |
| 3 | **Inversión** | −255…+255 | ∓255 | Corrige el sentido del acelerador (`INVERT_THROTTLE`) |
| 4 | **Curva expo** | −255…+255 | −255…+255 | Reduce sensibilidad cerca del centro, mantiene el tope |
| 5 | **Umbral bajo** | −255…+255 | −255…+255 | Elimina residuos < 10 y reexpande el rango |

### Sobre la curva expo

La fórmula usada es la exponencial cúbica ponderada estándar: mezcla una recta con una cúbica según el factor `e`.

| Factor | Comportamiento | Aplicado a |
| ---: | :--- | :--- |
| `0.0` | Respuesta lineal pura | — |
| `0.4` | Suavizado moderado | **Acelerador** |
| `0.6` | Suavizado fuerte | **Giro** |
| `1.0` | Cúbica pura, muy suave al centro | — |


### Mezclador diferencial

| Salida | Expresión conceptual |
| :--- | :--- |
| Motor A (izquierdo) | acelerador **−** (giro × ganancia) |
| Motor B (derecho) | acelerador **+** (giro × ganancia) |

---

## Parámetros configurables

| Constante | Valor | Unidad | Qué controla | Efecto de aumentarlo |
| :--- | ---: | :--- | :--- | :--- |
| `PPM_PIN` | 21 | GPIO | Entrada de señal | — |
| `CHANNELS` | 8 | — | Canales a decodificar | Debe coincidir con el TX |
| `FRAME_GAP` | 4000 | µs | Umbral de sincronía | Muy alto → no detecta trama · Muy bajo → falsos reinicios |
| `INVERT_THROTTLE` | −1 | — | Sentido del acelerador | Cambiar a `1` si va al revés |
| `deadMin` / `deadMax` | 1470 / 1530 | µs | Ancho de zona muerta | Más ancho → menos deriva, más recorrido perdido |
| Expo acelerador | 0.4 | — | Suavidad del avance | Más suave al centro |
| Expo giro | 0.6 | — | Suavidad del giro | Más suave al centro |
| Umbral bajo | 10 | /255 | Corte de residuo | Más alto → zona muerta efectiva mayor |
| `turnGain` | 1.2 | — | Agresividad del giro | Gira más rápido, satura antes |
| `TURN_90_TIME` | 300 | ms | Duración del giro automático | Ángulo mayor |
| Frecuencia PWM | 20000 | Hz | LEDC | Fuera del rango audible |
| Resolución PWM | 8 | bits | LEDC | Rango de salida 0–255 |
| Retardo de armado | 2000 | ms | Espera en el arranque | — |

### Guía rápida de calibración

| Síntoma | Parámetro a ajustar |
| :--- | :--- |
| El robot se mueve solo con los sticks centrados | Ampliar `deadMin` / `deadMax` |
| Avanza al revés | Cambiar `INVERT_THROTTLE` |
| Muy nervioso al centro | Subir el factor expo del eje afectado |
| Gira poco / gira demasiado | Ajustar `turnGain` |
| El giro automático no da 90° | Ajustar `TURN_90_TIME` |
| Motores chillan | Verificar frecuencia PWM ≥ 20 kHz |

