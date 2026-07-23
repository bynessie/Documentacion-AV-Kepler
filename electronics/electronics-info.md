# Documentación Técnica — PCB "KEPLER RC VOL.1"
### Subsistema de electrónica — Robot Minisumo RC / semi-autónomo

---

## 1. Identificación del subsistema

| Campo | Valor |
|---|---|
| **Nombre de la placa** | KEPLER RC VOL.1 |
| **Proyecto** | Robot Minisumo RC (Kepler) |
| **Subsistema** | Electrónica de control y potencia |
| **Organización** | Artemis Robotics |
| **Diseñador** | Marian Zaes (@marian_zaes) |
| **Herramienta EDA** | EasyEDA / LCEDA |
| **Revisión de esquemático** | 1.0 |
| **Fecha de esquemático** | 2026-06-17 |
| **Hojas** | 1/1 |
| **Capas PCB** | 2 (según render) |
| **Acabado / máscara** | Máscara azul, serigrafía blanca |

---

## 2. Descripción general

La PCB Kepler es una **placa madre de dos capas** que integra en un solo módulo la alimentación, el control y la etapa de potencia de un robot minisumo. Está construida alrededor de un **DFRobot Beetle ESP32-C3** montado en zócalo (headers hembra) y **dos módulos TB6612FNG**, uno por motor, con sus dos canales internos puestos en paralelo para duplicar la capacidad de corriente.

El robot opera por **radiocontrol** (receptor PWM de 3 canales) y reserva dos entradas para **sensores de línea QTR**, que permiten protección autónoma de borde del dohyo. Es decir, arquitectónicamente la placa es **RC con asistencia autónoma de borde**, no un minisumo autónomo completo (ver §9, hallazgo H-07).

### 2.1 Diagrama de bloques

```
        LiPo 3S 450mAh 11.1V
                │
           [ BORNERA1 ]  ← entrada de batería (2 pines, paso 5.08 mm)
                │
           [  SW1  ]  ← switch deslizante 3 pines (ON/OFF general)
                │
         ┌──────┴───────── net "12.1V" (VM) ──────┐
         │                                        │
    [ C1 220nF ]                          ┌───────┴────────┐
         │                                │                │
    [ U4  L7805 ]  ── 5V ──┐          [ U2 TB6612 ]   [ U3 TB6612 ]
         │                 │           (Motor A)       (Motor B)
    [ C2 100nF ]      [ SWESP ]            │                │
         │            (jumper)         [ MOTORA ]      [ MOTORB ]
    [ R1 220Ω ]            │            JST 2P          JST 2P
         │                 │
    [ LED1 5mm ]      [ Beetle ESP32-C3 ]
     (power ON)        DFR0868, VIN=5V
                             │
                        3.3V interno
                             │
         ┌───────────┬───────┴────┬───────────┬──────────┐
    [RECEPTOR]    [ CH2 ]      [ CH3 ]    [ QTR1 ]   [ QTR2 ]
     HX 3P         XH 3P        XH 3P      XH 3P      XH 3P
    (CH1 RC)      (RC)         (RC)      (línea)    (línea)
```

---

## 3. Etapa de potencia (alimentación)

| Ref. | Componente | Función |
|---|---|---|
| BORNERA1 | `CONN-TH_2P-P5.00` | Entrada de batería. Nets: `12V` y `GND` |
| SW1 | Switch deslizante 3 pines | Corte general de alimentación. Salida: net `12.1V` |
| C1 | 220 nF cerámico | Desacople en la entrada del regulador |
| U4 | **L7805** (TO-220) | Regulador lineal fijo 5 V, hasta 1.5 A. Pines IN / GND / VCC serigrafiados en la placa |
| C2 | 100 nF cerámico | Desacople en la salida del regulador |
| R1 + LED1 | 220 Ω + LED rojo 5 mm | Indicador de "power ON" del riel de 5 V |
| SWESP | Header macho 1×2 + jumper | Puente que conecta el riel de 5 V al `VIN` del Beetle |

### 3.1 Rieles del sistema

| Riel | Origen | Cargas |
|---|---|---|
| **VM (~11.1–12.6 V)** | Batería directa, después de SW1 | `VM` de U2 y U3 (motores) |
| **5 V** | L7805 desde VM | `VIN` del Beetle ESP32-C3 (vía jumper SWESP), LED indicador |
| **3.3 V** | LDO interno del Beetle | `VCC` y `STBY` de U2/U3, receptor RC, sensores QTR |

### 3.2 Función del jumper SWESP

El puente permite **desconectar el riel de 5 V del `VIN` del Beetle**. Con el jumper retirado, el ESP32-C3 puede alimentarse y programarse solo por USB-C sin energizar la etapa de potencia, y sin riesgo de contraalimentar el L7805 desde el puerto USB. Es un detalle de diseño acertado y conviene documentarlo en la serigrafía de la siguiente revisión (p. ej. `USB ONLY / BATT`).

---

## 4. Unidad de control

| Ref. | Componente |
|---|---|
| U1 | **DFRobot Beetle ESP32-C3** (DFR0868) |

**Características relevantes:** SoC ESP32-C3 (RISC-V single-core), Wi-Fi 802.11 b/g/n + BLE 5, 4 MB Flash, 400 kB SRAM, USB-C nativo (CDC), LDO 3.3 V a bordo, lógica 3.3 V **no tolerante a 5 V**.

El módulo va montado sobre **headers hembra de 2.54 mm**, lo que permite reemplazarlo sin desoldar. En el render se aprecia el conector USB-C accesible desde la cara superior.

---

## 5. Etapa de potencia motriz

| Ref. | Componente | Motor |
|---|---|---|
| U2 | Módulo TB6612FNG (Toshiba) | Motor A |
| U3 | Módulo TB6612FNG (Toshiba) | Motor B |

### 5.1 Topología: canales en paralelo

Cada driver TB6612FNG contiene dos puentes H independientes (A y B). En este diseño **ambos canales de cada driver se conectan en paralelo para manejar un solo motor**:

| Señal del driver | Conexión en U2 (Motor A) | Conexión en U3 (Motor B) |
|---|---|---|
| `AO1` + `BO1` | net `MA1` | net `MB1` |
| `AO2` + `BO2` | net `MA2` | net `MB2` |
| `AIN1` + `BIN1` | net `AIN1` | net `BIN1` |
| `AIN2` + `BIN2` | net `AIN2` | net `BIN2` |
| `PWMA` + `PWMB` | net `PWMA` | net `PWMB` |
| `STBY` | 3.3 V (fijo) | 3.3 V (fijo) |
| `VM` | 12.1 V | 12.1 V |
| `VCC` | 3.3 V | 3.3 V |

**Efecto:** la corriente continua disponible por motor pasa de 1.2 A a **≈ 2.4 A**, y el pico de ≈ 3.2 A a ≈ 6.4 A. Es la razón por la que se usan dos drivers en lugar de uno.

### 5.2 Motores

2× **JSUMO Core DC Motor**: 6 V nominal, 400 RPM, 120 mA en vacío, **3.2 A en stall**, 21 g c/u.

---

## 6. Mapa de pines (GPIO)

Tomado de la serigrafía de la placa y confirmado con el esquemático:

| Función | GPIO | Bloque | Conector / destino | Notas |
|---|---|---|---|---|
| `PWMA` | **5** | Driver A (U2) | — | Va a `PWMA` y `PWMB` de U2 |
| `AIN1` | **4** | Driver A (U2) | — | Va a `AIN1` y `BIN1` de U2 |
| `AIN2` | **6** | Driver A (U2) | — | Va a `AIN2` y `BIN2` de U2 |
| `PWMB` | **2** | Driver B (U3) | — | ⚠️ Pin de *strapping* |
| `BIN1` | **8** | Driver B (U3) | — | ⚠️ Pin de *strapping* |
| `BIN2` | **9** | Driver B (U3) | — | ⚠️ Pin de *strapping* / BOOT |
| `CH1` (RECEPTOR) | **21** | Radiocontrol | `RECEPTOR` (HX PM2.54 3P) | UART0 TX por defecto |
| `CH2` | **0** | Radiocontrol | `CH2` (JST-XH 3P) | ADC1_CH0 |
| `CH3` | **7** | Radiocontrol | `CH3` (JST-XH 3P) | — |
| `QTR1` | **1** | Sensor de línea | `QTR1` (JST-XH 3P) | ADC1_CH1 |
| `QTR2` | **20** | Sensor de línea | `QTR2` (JST-XH 3P) | UART0 RX por defecto |

**GPIO libres en el Beetle tras este diseño: solo IO3 e IO10.** La placa está prácticamente al límite de E/S disponibles (ver §9, H-07).

---

## 7. Conectores e interfaces

| Designador | Tipo | Pines | Señales | Uso |
|---|---|---|---|---|
| `BORNERA1` | Bornera de tornillo, paso 5.08 mm | 2 | `12V`, `GND` | Entrada de batería |
| `MOTORA` | JST-XH 2.54 mm | 2 | `MA1`, `MA2` | Motor izquierdo/derecho |
| `MOTORB` | JST-XH 2.54 mm | 2 | `MB1`, `MB2` | Motor opuesto |
| `RECEPTOR` | HX PM2.54-1×3P WC-Y | 3 | `GND`, `3.3V`, `CH1` | Canal 1 del receptor RC |
| `CH2` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, `CH2` | Canal 2 RC |
| `CH3` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, `CH3` | Canal 3 RC |
| `QTR1` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, señal | Sensor de línea 1 |
| `QTR2` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, señal | Sensor de línea 2 |
| `SWESP` | Header macho 1×2 + jumper | 2 | 5 V ↔ VIN | Selector de alimentación del MCU |
| `SW1` | Switch deslizante | 3 | — | Encendido general |
| `USB-C` | En el módulo Beetle | — | — | Programación / depuración |

Los conectores de señal están alineados en el borde inferior de la placa en el orden: **MOTORA · QTR1 · CH2 · CH3 · QTR2 · MOTORB**.

---

## 8. Lista de materiales (BOM)

### 8.1 Electrónica (subsistema PCB)

| # | Descripción | Cant. | Unit. (MXN) | Total (MXN) |
|---|---|---|---|---|
| 1 | Módulo driver dual TB6612FNG | 2 | $88.56 | $442.80 |
| 2 | Radiolink T8S 2.4G + receptor R8EF | 1 | $1,080.46 | $1,080.46 |
| 3 | Beetle ESP32-C3 (DFR0868) | 1 | $215.00 | $215.00 |
| 4 | Capacitor cerámico 220 nF (224) | 1 | $2.00 | $2.00 |
| 5 | Capacitor cerámico 100 nF (104) | 1 | $2.00 | $2.00 |
| 6 | Tira de pines hembra 1×40 | 3 | $6.00 | $18.00 |
| 7 | Tira de pines macho 1×40 | 1 | $6.00 | $6.00 |
| 8 | Resistencia 220 Ω 1/4 W | 1 | $1.00 | $1.00 |
| 9 | Conector JST 2P (macho+hembra) | 2 | $2.00 | $4.00 |
| 10 | LED 5 mm | 1 | $2.00 | $2.00 |
| 11 | Switch deslizante 3 pines | 1 | $3.50 | $3.50 |
| 12 | Bornera de tornillo 2P | 1 | $7.00 | $7.00 |
| 13 | Conector XT30 (par) | 2 | $10.00 | $20.00 |
| 14 | Jumper 2.54 mm | 1 | $1.50 | $1.50 |
| 15 | WS2812B NeoPixel (opcional) | 2 | $15.00 | $30.00 |
| 16 | Regulador L7805 TO-220 | 1 | $9.00 | $9.00 |
| | **Subtotal electrónica** | | | **$1,844.26** |

### 8.2 Otros subsistemas (referencia)

| # | Descripción | Cant. | Total (MXN) | Subsistema |
|---|---|---|---|---|
| 17 | JSUMO Core DC Motor 6 V 400 RPM | 2 | $550.76 | Chasis |
| 18 | Placa de latón 7×5×0.75 cm | 1 | $80.00 | Chasis |
| 19 | LiPo TATTU 3S 450 mAh 75C | 1 | $477.21 | Energía |
| | **Subtotal** | | **$1,107.97** | |

**Suma de todas las líneas: $2,952.23 MXN.** El BOM entregado indica un total de **$2,283.61 MXN** — hay una diferencia de $668.62 (ver §9, H-08).

---

## 9. Revisión de diseño — hallazgos

Ordenados por severidad. Estos puntos son para la **Rev 1.1 / Vol. 2**; la placa actual es funcional con los ajustes indicados.

### 🔴 Críticos

**H-01 — Tensión de motor muy por encima del nominal**
Los motores JSUMO Core son de **6 V nominales** y el riel `VM` es la batería directa (**11.1 V nominal, 12.6 V a plena carga**). A duty 100 % los motores trabajan a más del doble de su tensión de diseño.
→ *Acción:* limitar el duty de PWM por firmware a **≤ 50 %** como tope absoluto, o alimentar `VM` desde un buck de 6–7.4 V. Documentar el límite como constante en el código (`#define PWM_MAX 128`).

**H-02 — Receptor RC alimentado a 3.3 V**
Los conectores `RECEPTOR`, `CH2` y `CH3` entregan **3.3 V**. Los receptores Radiolink (incluido el R8EF) especifican típicamente **4.8–6 V**; a 3.3 V es probable que no arranque o que se comporte de forma errática.
→ *Acción:* verificar el rango de alimentación del R8EF. Si requiere 5 V, alimentar el receptor desde el riel de 5 V. **Importante:** en ese caso las salidas PWM del receptor serán de 5 V y el ESP32-C3 **no es tolerante a 5 V** — hay que añadir un divisor resistivo (10 kΩ/20 kΩ) o un level shifter en cada canal.

**H-03 — Corriente de stall superior a la capacidad continua del driver**
Stall del motor = **3.2 A**; capacidad continua con canales en paralelo ≈ **2.4 A**. En minisumo el motor pasa buena parte del combate cerca de stall (empuje).
→ *Acción:* el TB6612FNG tiene protección térmica y de sobrecorriente, así que no se destruye, pero **se apagará en pleno combate**. Mitigación inmediata: el límite de duty de H-01 reduce la corriente media. Para Vol. 2, migrar a drivers de mayor corriente (DRV8871, TB67H450FNG, o puente H discreto con MOSFETs).

### 🟠 Importantes

**H-04 — Uso de pines de strapping del ESP32-C3**
`PWMB`→GPIO2, `BIN1`→GPIO8 y `BIN2`→GPIO9 son los tres pines de arranque del ESP32-C3. GPIO8 debe estar en alto al arranque y GPIO9 en bajo fuerza el modo descarga.
→ *Acción:* verificar en el datasheet del TB6612FNG si sus entradas tienen pull-down interno. Si lo tienen, añadir pull-ups de 10 kΩ a 3.3 V en GPIO8 y GPIO9, o reasignar. Si la placa a veces "no arranca" o entra en modo bootloader sola, esta es la causa.

**H-05 — Disipación del L7805**
Con 12.6 V de entrada y 5 V de salida, cada 100 mA consumidos disipan **0.76 W**. El Beetle con Wi-Fi activo consume 100–150 mA promedio y hasta ~350 mA en picos de TX → **0.8 a 2.6 W** en un TO-220 sin disipador (θ<sub>JA</sub> ≈ 60 °C/W).
→ *Acción:* si se usa Wi-Fi/BLE, es obligatorio disipador o —mejor— sustituir por un convertidor **buck** (MP1584, LM2596, TPS5430). Si el Wi-Fi permanece apagado y solo se usa RC, el L7805 con un pequeño disipador es viable.

**H-06 — Capacitancia de bulk insuficiente en VM**
C1 (220 nF) es demasiado pequeño para una entrada compartida con motores. Los picos de corriente de arranque/inversión provocarán caídas de tensión que pueden resetear el MCU.
→ *Acción:* añadir un **electrolítico de 220–470 µF / 25 V** en `VM`, físicamente cerca de los drivers, más 100 nF cerámico en paralelo.

**H-07 — Cobertura sensorial insuficiente para operación autónoma**
La placa tiene **2 entradas de línea (QTR) y 0 entradas para sensores de oponente**. Un minisumo autónomo típico necesita 3–5 sensores de distancia + 3–4 de línea. Además solo quedan **2 GPIO libres** (IO3, IO10) en el ESP32-C3.
→ *Acción:* para Vol. 2, dos caminos:
- **A)** Migrar a un MCU con más E/S (ESP32-S3, RP2040).
- **B)** Mantener el ESP32-C3 y pasar los sensores a **I²C** (VL53L0X/VL53L1X con multiplexor TCA9548A, o un expansor PCF8574 para los QTR digitales). Solo consume 2 pines y deja el resto libre.

### 🟡 Menores / mejoras

**H-08 — Inconsistencia aritmética en el BOM**
La suma de las columnas "Precio total" da **$2,952.23**, pero el total declarado es **$2,283.61**. Revisar la fórmula del total.

**H-09 — Componentes en el BOM que no aparecen en el esquemático**
Los **WS2812B** (ítem 15) y los **XT30** (ítem 13) no están en el esquemático ni en el render. Si el XT30 es el conector de la batería que entra a la bornera, conviene indicarlo como cable ensamblado; si los NeoPixel son para Vol. 2, marcarlos como "no montado / futuro".

**H-10 — Conteo de conectores incompleto en el BOM**
El BOM lista 2 pares de JST-2P, pero la placa lleva **2× JST-XH 2P (motores) + 4× JST-XH 3P (CH2, CH3, QTR1, QTR2) + 1× HX PM2.54 3P (receptor)**. Faltan también: la fabricación del PCB, los **sensores QTR** propiamente dichos, cable AWG 22–24, y separadores/tornillería.

**H-11 — `STBY` cableado permanentemente a 3.3 V**
El MCU no puede poner los drivers en alta impedancia. Para un paro de emergencia limpio (y por reglamento de minisumo, un apagado confiable), conviene llevar `STBY` a un GPIO.
→ *Acción:* en Vol. 2, unir ambos `STBY` a un solo GPIO libre (IO3 o IO10) — cuesta un pin y da un kill-switch por software.

**H-12 — Sin protección de polaridad inversa**
Una inversión de la batería en la bornera destruye los drivers y el regulador.
→ *Acción:* añadir un **MOSFET P-channel** en serie con VM (bajo drop, no penaliza como un diodo).

**H-13 — Restricción dimensional**
Reglamento minisumo estándar: **10 × 10 cm** máximo y **500 g**. El chasis de latón es de 7×5 cm; verificar que la PCB montada sobre él no exceda el perímetro y que el conjunto con LiPo y motores cumpla el peso.

---

## 10. Procedimiento de puesta en marcha (bring-up)

Ejecutar en este orden. **No conectar batería ni motores hasta el paso 4.**

| # | Paso | Criterio de aceptación |
|---|---|---|
| 1 | Inspección visual: cortos entre `12V`–`GND` y `5V`–`GND` con multímetro en continuidad | Sin continuidad |
| 2 | Con jumper SWESP **retirado**, alimentar solo por USB-C. Cargar un *blink* | El Beetle enumera y ejecuta |
| 3 | Alimentar la bornera con **fuente de banco a 12 V, límite de corriente 200 mA**, sin drivers ni motores | LED1 enciende. Medir salida del L7805: **4.9–5.1 V** |
| 4 | Colocar jumper SWESP. Verificar `VIN` y el riel de 3.3 V del Beetle | 5 V en VIN, 3.3 V estable |
| 5 | Insertar U2 y U3 (sin motores). Verificar `VM` y `VCC` en cada módulo | VM ≈ Vbat, VCC = 3.3 V |
| 6 | Prueba de motores **con duty limitado a 30 %**, un motor a la vez | Giro suave, sentido correcto en ambas direcciones |
| 7 | Medir temperatura del L7805 tras 2 min con Wi-Fi activo | < 70 °C, o instalar disipador |
| 8 | Conectar receptor RC y leer anchos de pulso de CH1/CH2/CH3 | 1000–2000 µs estables |
| 9 | Conectar QTR y calibrar sobre borde blanco/superficie negra | Diferencia clara entre ambas lecturas |
| 10 | Sustituir fuente de banco por LiPo 3S | Comportamiento idéntico al paso 6–9 |

### 10.1 Puntos de medición

| Nodo | Valor esperado | Dónde medir |
|---|---|---|
| `12V` / `VM` | 11.1–12.6 V | Bornera, pin `VM` de U2/U3 |
| `5V` | 4.90–5.10 V | Pin `VCC` del L7805 / header SWESP |
| `3.3V` | 3.25–3.35 V | Pin `VCC` de U2/U3, pin 3.3 V de cualquier conector |

---

## 11. Notas de firmware

- **PWM:** usar el periférico LEDC del ESP32-C3. Frecuencia recomendada **20 kHz** (fuera del rango audible; el TB6612FNG admite hasta 100 kHz). Resolución 8 bits.
- **Límite de duty:** definir `PWM_MAX = 128` (50 %) como constante global mientras `VM` sea la batería directa — ver H-01.
- **Lectura RC:** capturar los pulsos por interrupción de flanco (`attachInterrupt` en CHANGE) midiendo con `micros()`. Implementar *failsafe*: si no llega pulso válido en **> 100 ms**, poner ambos motores a cero.
- **Arranque seguro:** inicializar todos los pines de driver en `LOW` **antes** de habilitar la etapa de potencia.
- **Rutina de borde:** los QTR deben leerse en el bucle principal a alta frecuencia y su reacción tiene prioridad sobre el comando RC (retroceso + giro).
- **Wi-Fi:** desactivarlo (`WiFi.mode(WIFI_OFF)`) durante el combate — reduce el consumo y por tanto la disipación del L7805, y evita picos de corriente.

---

## 12. Estado y siguientes pasos

| Estado | Detalle |
|---|---|
| Esquemático | ✅ Rev 1.0 completo |
| Layout / PCB | ✅ Render 3D disponible |
| Fabricación | ⬜ Pendiente |
| Bring-up | ⬜ Pendiente |
| Firmware | ⬜ Pendiente |

**Antes de fabricar**, resolver como mínimo: **H-02** (tensión del receptor), **H-04** (pines de strapping) y **H-06** (capacitor de bulk). Los tres son cambios baratos en el esquemático y evitan un respin completo.

---

*Documento generado a partir del esquemático EasyEDA Rev 1.0 (2026-06-17), el render 3D de la PCB y el BOM de julio 2026. Los valores marcados como "verificar" requieren confirmación contra los datasheets de los componentes reales.*
