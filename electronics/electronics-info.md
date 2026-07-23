<div align="center">

#  KEPLER RC - PCB

</div>

---

## Identificación del subsistema

| Campo | Valor |
|---|---|
| **Nombre de la placa** | KEPLER RC VOL.1 |
| **Proyecto** | Robot Minisumo RC (Kepler) |
| **Subsistema** | Electrónica de control y potencia |
| **Organización** | Artemis Robotics |
| **Diseñador** | Marian Zaes (@marian_zaes) |
| **Herramienta EDA** | EasyEDA / LCEDA |
| **Capas PCB** | 2 |

---

## Descripción general

La PCB Kepler es una **placa madre de dos capas** que integra en un solo módulo la alimentación, el control y la etapa de potencia de un robot minisumo. Está construida alrededor de un **DFRobot Beetle ESP32-C3** montado en zócalo (headers hembra) y **dos módulos TB6612FNG**, uno por motor, con sus dos canales internos puestos en paralelo para duplicar la capacidad de corriente.

El robot opera por **radiocontrol** (receptor PWM de 3 canales) y reserva dos entradas para **sensores de línea QTR**, que permiten protección autónoma de borde del dohyo. Es decir, arquitectónicamente la placa es **RC con asistencia autónoma de borde**, no un minisumo autónomo completo.

### Diagrama de bloques

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

## Etapa de potencia (alimentación)

| Ref. | Componente | Función |
|---|---|---|
| BORNERA1 | `CONN-TH_2P-P5.00` | Entrada de batería. Nets: `12V` y `GND` |
| SW1 | Switch deslizante 3 pines | Corte general de alimentación. Salida: net `12.1V` |
| C1 | 220 nF cerámico | Desacople en la entrada del regulador |
| U4 | **L7805** (TO-220) | Regulador lineal fijo 5 V, hasta 1.5 A. Pines IN / GND / VCC serigrafiados en la placa |
| C2 | 100 nF cerámico | Desacople en la salida del regulador |
| R1 + LED1 | 220 Ω + LED rojo 5 mm | Indicador de "power ON" del riel de 5 V |
| SWESP | Header macho 1×2 + jumper | Puente que conecta el riel de 5 V al `VIN` del Beetle |

### Rieles del sistema

| Riel | Origen | Cargas |
|---|---|---|
| **VM (~11.1–12.6 V)** | Batería directa, después de SW1 | `VM` de U2 y U3 (motores) |
| **5 V** | L7805 desde VM | `VIN` del Beetle ESP32-C3 (vía jumper SWESP), LED indicador |
| **3.3 V** | LDO interno del Beetle | `VCC` y `STBY` de U2/U3, receptor RC, sensores QTR |

### Función del jumper SWESP

El puente permite **desconectar el riel de 5 V del `VIN` del Beetle**. Con el jumper retirado, el ESP32-C3 puede alimentarse y programarse solo por USB-C sin energizar la etapa de potencia, y sin riesgo de contraalimentar el L7805 desde el puerto USB. 

---

## Unidad de control

| Ref. | Componente |
|---|---|
| U1 | **DFRobot Beetle ESP32-C3** (DFR0868) |

**Características relevantes:** SoC ESP32-C3 (RISC-V single-core), Wi-Fi 802.11 b/g/n + BLE 5, 4 MB Flash, 400 kB SRAM, USB-C nativo (CDC), LDO 3.3 V a bordo, lógica 3.3 V **no tolerante a 5 V**.

El módulo va montado sobre **headers hembra de 2.54 mm**, lo que permite reemplazarlo sin desoldar. En el render se aprecia el conector USB-C accesible desde la cara superior.

---

## Etapa de potencia motriz

| Ref. | Componente | Motor |
|---|---|---|
| U2 | Módulo TB6612FNG (Toshiba) | Motor A |
| U3 | Módulo TB6612FNG (Toshiba) | Motor B |

### Topología: canales en paralelo

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

### Motores

2× **JSUMO Core DC Motor**: 6 V nominal, 400 RPM, 120 mA en vacío, **3.2 A en stall**, 21 g c/u.

---

## Mapa de pines (GPIO)

Tomado de la serigrafía de la placa y confirmado con el esquemático:

| Función | GPIO | Bloque | Conector / destino | Notas |
|---|---|---|---|---|
| `PWMA` | **5** | Driver A (U2) | — | Va a `PWMA` y `PWMB` de U2 |
| `AIN1` | **4** | Driver A (U2) | — | Va a `AIN1` y `BIN1` de U2 |
| `AIN2` | **6** | Driver A (U2) | — | Va a `AIN2` y `BIN2` de U2 |
| `PWMB` | **2** | Driver B (U3) | — | Va a `PWMA` y `PWMB` de U3 |
| `BIN1` | **8** | Driver B (U3) | — | Va a `AIN1` y `BIN1` de U3 |
| `BIN2` | **9** | Driver B (U3) | — | Va a `AIN2` y `BIN2` de U3 |
| `CH1` (RECEPTOR) | **21** | Radiocontrol | `RECEPTOR` (HX PM2.54 3P) | UART0 TX por defecto |
| `CH2` | **0** | Radiocontrol | `CH2` (JST-XH 3P) | Pin extra |
| `CH3` | **7** | Radiocontrol | `CH3` (JST-XH 3P) | Pin extra |
| `QTR1` | **1** | Sensor de línea | `QTR1` (JST-XH 3P) |  Pin extra |
| `QTR2` | **20** | Sensor de línea | `QTR2` (JST-XH 3P) |  Pin extra |

**GPIO libres en el Beetle tras este diseño: solo IO3 e IO10.** La placa está prácticamente al límite de E/S disponibles (ver §9, H-07).

---

## 7. Conectores e interfaces

| Designador | Tipo | Pines | Señales | Uso |
|---|---|---|---|---|
| `BORNERA1` | Bornera de tornillo, paso 5.08 mm | 2 | `12V`, `GND` | Entrada de batería |
| `MOTORA` | JST-XH 2.54 mm | 2 | `MA1`, `MA2` | Motor izquierdo |
| `MOTORB` | JST-XH 2.54 mm | 2 | `MB1`, `MB2` | Motor derecho |
| `RECEPTOR` | HX PM2.54-1×3P WC-Y | 3 | `GND`, `3.3V`, `CH1` | Canal 1 del receptor RC |
| `CH2` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, `CH2` | Canal 2 RC |
| `CH3` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, `CH3` | Canal 3 RC |
| `QTR1` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, señal | Sensor de línea 1 |
| `QTR2` | JST-XH 2.54 mm | 3 | `GND`, `3.3V`, señal | Sensor de línea 2 |
| `SWESP` | Header macho 1×2 + jumper | 2 | 5 V ↔ VIN | Selector de alimentación del MCU |
| `SW1` | Switch deslizante | 3 | — | Encendido general |
| `USB-C` | En el módulo Beetle | — | — | Programación / depuración |

---

## Lista de materiales (BOM)

### Electrónica (subsistema PCB)

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

---
