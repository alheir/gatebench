# GateBench — Definición de la placa

**Estado:** borrador de diseño, v0.2
**Contexto:** 25.17 Electrónica II — ITBA
**Objetivo:** placa base ("carrier") para la UPduino v3.1 que permita hacer prácticas de lógica
combinacional, secuencial, FSM y Verilog sin protoboard ni cableado, y que sirva además como
estación de trabajo de FPGA para proyectos más ambiciosos.

> Este documento es la fuente de verdad de las **definiciones** de la placa. El mapeo de pines
> concreto vive en `docs/pinout.csv`; el manual de usuario en `docs/manual/`.

---

## 1. La restricción que define todo el diseño

La UPduino v3.1 saca **32 GPIO** a los headers. Nada más.

Los 3 pines del driver RGB (`led_red` / `led_green` / `led_blue`, IO 39/40/41) **no son I/O
disponible**: están cableados al LED RGB que la UPduino ya trae soldado. Están espejados en el
header, así que cualquier cosa que cuelgues ahí prende en paralelo con el LED de la placa y se
reparte la corriente constante del driver. No los cuentes.

Los 4 pines SPI (`spi_sck` / `spi_mosi` / `spi_miso` / `spi_ssn`) van a la flash y al FT232H.
El FTDI los maneja durante la programación. **Tampoco los uses.**

Presupuesto real: **32 pines**.

El Basys 3 usa unos 70 I/O del Artix-7 para lo que tiene. Su lista de periféricos es una buena
referencia de *qué* poner, pero su arquitectura (todo en paralelo, un pin por señal) no es
replicable acá.

**Consecuencia arquitectónica:** la placa es *serial-first*. El I/O lento (switches, LEDs,
display, teclado) va detrás de cadenas de shift registers; los pines directos se reservan para
lo que tiene requisitos de tiempo real (VGA, clock, WS2812, UART).

Y eso no es una concesión: es el contenido de la materia. El TP1 define un canal serie, el TP2
lo implementa con 74HC595, y los finales piden shift registers con carga paralela y periféricos
SPI. La placa enseña serializando.

---

## 2. Presupuesto de pines (32/32)

| # | Bloque | Pines | Señales |
|---|--------|-------|---------|
| 1 | Clock | 1 | `CLK_12M` → `gpio_35` |
| 2 | VGA 3-3-3 | 11 | R[2:0], G[2:0], B[2:0], HS, VS |
| 3 | Cadena de scan I/O | 5 | `SER_OUT`, `SCLK`, `RCLK`, `SH/LD`, `SER_IN` |
| 4 | Bus SPI | 3 | `SCK`, `MOSI`, `MISO` |
| 5 | CS del DAC R-2R | 1 | `CS_DAC` |
| 6 | CS del ADC | 1 | `CS_ADC` |
| 7 | CS del Pmod de expansión | 1 | `CS_EXT` |
| 8 | UART | 2 | `TX`, `RX` |
| 9 | I²C | 2 | `SDA`, `SCL` |
| 10 | WS2812 | 1 | `WS_DIN` |
| 11 | Pulsador directo | 1 | `BTN0` (reset / clock manual) |
| 12 | GPIO libre | 3 | `IO0`, `IO1`, `IO2` a header |
| | **Total** | **32** | |

**Clock (1 pin).** La UPduino trae el oscilador de 12 MHz en el pin 41 del header; el jumper R16
lo lleva a `gpio_20`, pero `gpio_35` es el pin ideal para clock porque permite ubicar el PLL al
lado. En la placa: puente (0 Ω o solder jumper) de pin 41 a `gpio_35`, con 33 Ω en serie para
amortiguar el stub. **Dejar R16 abierto** en la UPduino, así `gpio_20` queda libre.
*Verificar contra el esquemático oficial en `tinyvision-ai-inc/UPduino-v3.0` antes de rutear.*

**GPIO libre.** Los 3 pines de reserva salen de haber elegido VGA 3-3-3 en vez de 4-4-4. Son el
único margen que tiene el diseño: cualquier periférico que aparezca en revisión sale de ahí.

---

## 3. Qué va soldado y qué va en conector

**Regla general:** todo lo que la materia evalúa va soldado (para que se pueda usar en clase sin
armar nada); todo lo que amplía el alcance va en conector estándar.

### 3.1 Soldado a la placa

| Bloque | Detalle | Pines FPGA |
|---|---|---|
| 16 switches SPST + pull-down 10 K | entran a la cadena 165 | 0 |
| 5 pulsadores (U/D/L/R/C) | cadena 165 | 0 |
| DIP 2 bits (selector de clock lento) | cadena 165 | 0 |
| Encoder rotativo (A/B/SW) | cadena 165 | 0 |
| 16 LEDs + 330 Ω | cadena 595 | 0 |
| Display 4 dígitos 7 seg, ánodo común | cadena 595 | 0 |
| 4×74HC595 + 4×74HC165 | las cadenas | 5 |
| DAC R-2R 8 bits (74HC595 + escalera + opamp) | bus SPI | 1 (CS) |
| ADC MCP3204 (4 canales, 12 bit) | bus SPI | 1 (CS) |
| Potenciómetro 10 K | → ADC CH0 | 0 |
| 8× WS2812B + 74AHCT125 | | 1 |
| VGA: escaleras + DB15 | | 11 |
| CP2102N + USB-C (UART) | | 2 |
| EEPROM 24LC + bus I²C | | 2 |
| Pulsador con RC + 74HC14 bypasseable | | 1 |
| Alimentación (ver §6) | | 0 |

### 3.2 Conectores

| Conector | Formato | Para qué | Pines FPGA extra |
|---|---|---|---|
| Zócalo UPduino | 2×24 torneado, 2.54 mm | la placa madre; **no soldar la UPduino** | — |
| Pmod tipo 2 (SPI) | 1×6 | microSD, MAX7221 externo, módulos varios | 1 (`CS_EXT`) |
| Qwiic / STEMMA QT | JST-SH 4 pines | sensores I²C del ecosistema Sparkfun/Adafruit | 0 |
| **Teclado matricial 4×4** | **1×8, paso 2.54 mm** | **teclado de membrana comercial o la placa del TP3** | 0 |
| Extensión de cadena serie | 1×6: `SER_OUT`, `SCLK`, `RCLK`, `SER_RET`, 3V3, GND | la placa del TP2 se **encadena** como extensión natural | 0 |
| Puntas / analizador lógico | 2×5, GND alternado | osciloscopio y LA | 0 |
| Salida WS2812 | JST 3 pines + 5 V | extender tira externa | 0 |
| Entrada analógica externa | 2 pines o jack | → ADC CH1 | 0 |
| GPIO libre | 1×4 (3 señales + GND) | reserva de diseño | 3 |
| Salida DAC | jack 3.5 mm y/o BNC | osciloscopio, parlante activo | 0 |
| Alimentación | USB-C | ver §6 | 0 |

---

## 4. Las cadenas de shift registers

### 4.1 Salida — 4×74HC595 = 32 bits

| Bits | Destino |
|---|---|
| 31..16 | 16 LEDs |
| 15..8 | segmentos A..G + DP |
| 7..4 | ánodos de los 4 dígitos (vía PNP/P-MOS) |
| 3..0 | columnas del teclado matricial (al conector) |

### 4.2 Entrada — 4×74HC165 = 32 bits

| Bits | Origen |
|---|---|
| 31..16 | 16 switches |
| 15..11 | 5 pulsadores |
| 10..7 | filas del teclado matricial (del conector) |
| 6..5 | encoder A/B |
| 4 | pulsador del encoder |
| 3..2 | DIP selector de clock lento |
| 1..0 | reserva (test points) |

### 4.3 Pines y timing

`SCLK` es **compartido** entre las dos cadenas: el 595 desplaza en flanco ascendente de SRCLK y
el 165 desplaza en flanco ascendente de CLK con SH/LD alto. Es full-duplex, igual que SPI.

- `SER_OUT`, `SCLK`, `RCLK` (latch de salida)
- `SH/LD`, `SER_IN`
- `/OE` del 595 atado a masa (o a `RCLK` invertido si aparece ghosting)

74HC a 3.3 V anda cómodo hasta ~5 MHz. Una vuelta completa de 32 bits a 2 MHz son 16 µs, o sea
~62 kHz de refresco de la cadena. Con 4 dígitos eso da 15 kHz por dígito, dos órdenes de
magnitud por encima de los 60 Hz de trama que se necesitan.

Familia: **74HC595 / 74HC165**, no LVC. Mantiene continuidad con la familia que ya se usa en el
TP1 y las hojas de datos ya están en el material de la materia.

### 4.4 BSP: la contrapartida obligatoria

El costo de usabilidad de las cadenas se paga con software, no con hardware:

```verilog
module gb_board_io (
    input  wire        clk,
    input  wire        rst,
    // interfaz paralela hacia el usuario
    output wire [15:0] sw,
    output wire [4:0]  btn,
    output wire [3:0]  key_row,
    input  wire [15:0] led,
    input  wire [7:0]  seg,
    input  wire [3:0]  digit_sel,
    input  wire [3:0]  key_col,
    // pines físicos
    output wire ser_out, sclk, rclk, shld,
    input  wire ser_in
);
```

Quien recién arranca escribe `assign led = sw;` y funciona. Quien quiere entender abre el módulo
y ve la cadena. Esto es tan importante como el PCB.

---

## 5. Decisiones de bloque

### 5.1 VGA: 3-3-3

512 colores, más que suficiente para lo que entra en la SPRAM del UP5K, y libera 3 pines que van
al header de GPIO libre. El Basys 3 usa 4 bits por canal (escalera 4K/2K/1K/510 Ω por canal,
hoja 2 de su esquemático); acá se usa el mismo principio con un resistor menos por canal.

Los footprints del cuarto resistor por canal **quedan puestos y sin poblar**, por si en una
revisión futura se liberan pines.

Escalera binaria terminando en los 75 Ω del monitor. Calcular los valores para que el fondo de
escala dé 0.7 V sobre 75 Ω es un ejercicio de guía directo.

### 5.2 Display: 4 dígitos multiplexados, sin MAX7219

El Basys 3 también usa 4 dígitos (hoja 2 de su esquemático: `DISP1`, un KW4-281ASB con ánodos
A1–A4 y señales AN0–AN3). El TP3 pide mínimo 4. Coincide.

El MAX7219/MAX7221 hace la multiplexación, el decode BCD y el control de corriente por vos. Eso
es exactamente lo que el TP2 y el TP3 **evalúan** ("corriente y frecuencia del display
multiplexado"). Ponerlo onboard le saca el ejercicio al alumno.

El MAX7221 (que es el que tiene timing compatible con SPI y slew limitado, a diferencia del
7219) queda disponible como módulo externo colgado del Pmod SPI, para quien quiera 8 dígitos en
un proyecto. Necesita 4–5.5 V, así que ese módulo se lleva su propio level shifter.

Dimensionamiento, siguiendo lo que hace el Basys 3 (100 Ω por segmento, PNP en el ánodo con
2.2 K de base):

- Pico por segmento: (3.3 − 2.0) / 100 ≈ 13 mA
- Con duty 25%: promedio 3.3 mA por segmento
- Pico por dígito con los 8 segmentos: ~104 mA a través del PNP

Un MMBT3906 (200 mA) queda justo. **Usar MMBT2907A o un P-MOS pequeño**, que además cae menos.

### 5.3 Teclado matricial: sólo conector

Header 1×8 (4 columnas + 4 filas) a paso 2.54 mm, compatible con los teclados de membrana
comerciales de 4×4, que traen justamente un flex de 8 vías con ese paso.

Ventajas sobre montarlo onboard: no ocupa área de PCB, permite usar el teclado del TP3 sin
modificaciones, y el teclado de membrana es reemplazable y barato.

Las columnas salen de los bits 3..0 de la cadena 595 y las filas entran por los bits 10..7 de la
cadena 165. Costo en pines de FPGA: cero.

**A verificar en el esquemático:** los teclados de membrana no traen pull-ups. La cadena 165
necesita pull-downs de 10 K en las cuatro filas, en la placa.

### 5.4 DAC de propósito general

Los finales piden textualmente *"una FPGA y un DAC paralelo... señal triangular de 100 kHz de
8 bits signados"*. Un R-2R de 8 bits directo se come 8 pines, así que:

> **74HC595 dedicado + escalera R-2R de 8 bits + buffer con opamp (MCP6002 / TLV9062) + salida a
> jack 3.5 mm y BNC.** Cuelga del bus SPI con su propio CS (que además hace de RCLK). Actualizar
> son 8 clocks: a 5 MHz eso es 1.6 µs, o sea ~600 kSps. Sobra para el triangular de 100 kHz.

Es un DAC SPI hecho con lógica discreta. Didácticamente vale más que un chip.

**Sobre los bits:** un R-2R de 8 bits con resistores al 1% **no garantiza monotonicidad**. La
precisión depende del apareamiento de relaciones, no del valor absoluto. Opciones: poblar con
0.1% (en 0603 no es caro) → 8 bits reales; o poblar con 1% y asumir ~6 bits efectivos.

En cualquiera de los dos casos, **medir INL y DNL de esa escalera es un TP de laboratorio
buenísimo** y justifica solo el bloque.

### 5.5 ADC: MCP3204, 4 canales

El potenciómetro no consume la entrada del ADC:

| Canal | Entrada |
|---|---|
| CH0 | potenciómetro onboard 10 K |
| CH1 | entrada externa por conector (10 K serie + clamp a rieles) |
| CH2 | LDR o termistor onboard |
| CH3 | **realimentación de la salida del DAC R-2R** |

CH3 es la joya: la FPGA puede medir su propio DAC y calcular INL/DNL sin instrumental externo.
Autotest de fábrica y ejercicio de laboratorio en el mismo cable.

### 5.6 Clock lento seleccionable

No hace falta hardware: la UPduino tiene 12 MHz y la división se hace adentro. El bloque físico
es sólo un **DIP de 2 bits + un pulsador**, ambos en la cadena 165, costo cero en pines.

| DIP | Clock del usuario |
|---|---|
| 00 | manual (un flanco por pulsación, con antirrebote) |
| 01 | 1 Hz |
| 10 | 10 Hz |
| 11 | 12 MHz (libre) |

Poder ejecutar una FSM paso a paso mirando los LEDs es *la* herramienta pedagógica de lógica
secuencial. El Basys 3 no lo tiene y se extraña en Guías 3, 4 y 5.

### 5.7 Bloque de antirrebote comparativo

Un pulsador con RC + 74HC14 y un jumper para bypassearlo. Con el osciloscopio se ve el rebote
crudo contra el limpio; con la FPGA se hace el antirrebote digital y se comparan.

Cierra dos ejercicios diferidos del proyecto de guías: el de Schmitt trigger de Guía 0 §4
(V_T+ / V_T− / histéresis desde hoja de datos) y el integrador de debounce + RC de Guía 3/4,
ahora con hardware real en la mano.

### 5.8 WS2812

8 en línea + conector para extender tira externa. Necesita:

- Level shifter 3.3 → 5 V (74AHCT125 o SN74LVC1T45): obligatorio, no opcional
- 330 Ω en serie en DIN
- 470–1000 µF de bulk en el riel de 5 V

El protocolo NRZ de 800 kbps es un ejercicio de timing y FSM excelente, y visualmente paga.

### 5.9 UART

CP2102N o CH340 + USB-C propio. Consola serie de la FPGA a la PC. Es el periférico que más
proyectos habilita y el mejor mecanismo de debug que van a tener.

**No intentar multiplexar el FT232H de la UPduino:** está en modo MPSSE para programar.

### 5.10 Descartados

| Bloque | Por qué |
|---|---|
| PS/2 | muerto; el Basys 3 lo tiene sólo por legado, y vía un PIC |
| HDMI / DVI | el UP5K no llega a TMDS de forma confiable. VGA es la decisión correcta |
| Micrófono PDM + CIC | hermoso para el UP5K pero fuera del alcance de 25.17. Queda el header |
| MAX7219 onboard | tapa exactamente lo que evalúan TP2 y TP3 (ver §5.2) |
| Teclado matricial onboard | ocupa área y no aporta sobre el conector (ver §5.3) |
| LED RGB adicional | los pines ya están comprometidos con el de la UPduino |

---

## 6. Alimentación

**Dato duro:** la UPduino ofrece 5 V / 3.3 V / GND para alimentar tu proyecto, pero **por debajo
de 200 mA**. Con 16 LEDs, un display multiplexado y 8 WS2812 (hasta ~60 mA cada uno en blanco
pleno) te pasás varias veces. La placa tiene alimentación propia.

### 6.1 Regulador: LDO, y por lo tanto entrada de 5 V únicamente

Se eligió **LDO** sobre buck: es más silencioso para el ADC y el DAC R-2R (sin ripple de
conmutación acoplado al riel analógico), más barato, y no necesita inductor ni layout cuidado.

**Consecuencia directa e inevitable:** con LDO, la entrada **tiene que ser 5 V**. Un jack de
9 V disiparía (9 − 3.3) × 0.4 A ≈ 2.3 W en el regulador, que es inviable en encapsulado SMD.
Con 5 V la caída es (5 − 3.3) × 0.4 A ≈ 0.7 W, manejable en SOT-223 o DPAK con pour de cobre.

Por eso: **entrada única por USB-C a 5 V. Sin jack de alimentación.** Menos formas de que un
alumno queme la placa.

```
USB-C (5 V)
  → PTC + protección de polaridad inversa (P-MOSFET)
  → riel 5 V   (WS2812, level shifters)
  → LDO ≥1 A, SOT-223/DPAK con pour → riel 3.3 V   (74HC, DAC, ADC, periféricos)
```

**Presupuesto estimado del riel de 3.3 V:** 16 LEDs (~80 mA), display multiplexado (~100 mA
pico, ~26 mA promedio), 8×74HC (~10 mA), CP2102N (~20 mA), DAC + ADC + opamp (~30 mA). Total
~250–400 mA según qué esté prendido. Con 1 A de LDO hay margen de sobra.

### 6.2 Reglas de dirección de energía

1. **Nunca alimentar el pin de 3.3 V de la UPduino** desde el LDO de la placa. Estarías
   empujando contra la salida de su LDO onboard.
2. En uso normal la UPduino se alimenta de su propio USB (que igual necesita para programar), y
   con la placa comparte sólo GND.
3. Para demo standalone (bitstream en flash, sin PC), inyectar 5 V al pin 8 de la UPduino
   **a través de un ORing**: dos Schottky, o un ideal-diode tipo LM66100 / TPS2116. Así conviven
   las dos fuentes sin pelearse.
4. Dejar el pin VIO en 3.3 V. Existe un tutorial oficial para alimentar los bancos a otra
   tensión, pero no exponerlo en esta placa.

### 6.3 Agregados

- Llave de encendido general
- LED indicador por riel (5 V y 3.3 V)
- **Shunt de sensado + test points en el riel de 3.3 V**, para que puedan medir cuánto consume su
  diseño. Conecta directo con los criterios eléctricos que ya se evalúan en el TP3
- Bulk de 100 µF por riel, 100 nF local por integrado

---

## 7. Protección: qué sí y qué es overkill

Bufferear todo con '245 es overkill: sumás tpd, sumás control de dirección (que no tenés pines
para manejar) y sumás BOM. En orden de valor real:

| Medida | Veredicto |
|---|---|
| **Resistencia serie en todo pin que sale de la placa** | Siempre. Es lo que hace el Basys 3: 200 Ω en Pmods, 100 Ω en segmentos, 330 Ω en LEDs, 100 Ω en HS/VS. Costo cero |
| **Array de clamp / ESD en conectores expuestos** | Sí. PRTR5V0U2X o similar en VGA, Pmod, entrada analógica, salida WS2812, teclado |
| **Level shifter donde hay cambio de dominio** | Obligatorio (WS2812, MAX7221 externo). No es protección, es funcionalidad |
| **Driver donde hay corriente** | Sí: PNP/P-MOS en los ánodos del display. Los LEDs los maneja el 595, la FPGA no ve esa corriente nunca |
| **Buffer en el VGA** | No. La escalera resistiva ya aísla |
| **Buffer bidireccional genérico en todo** | No. Overkill |

**La regla, que además sirve como criterio para darle a los alumnos:** resistencia serie siempre,
clamp en conectores expuestos, buffer sólo cuando hay cambio de tensión de dominio o corriente
que la FPGA no puede dar.

---

## 8. Mecánica y fabricación

- **Zócalos torneados 2×24** para la UPduino (el TP3 exige zócalo, no soldar), con tope mecánico
  o marca de orientación bien visible. **Verificar el paso entre filas contra el mecánico oficial
  de KiCad del repo, no estimarlo.**
- 4 capas con plano de masa sólido. Con VGA a 25 MHz y clock de 12 MHz dos capas con pour anda,
  pero la diferencia de precio en JLCPCB ya no justifica el riesgo.
- SMD ensamblado en fábrica (escaleras R-2R apareadas, level shifters, CP2102N); through-hole
  para switches, pulsadores y LEDs, que es lo que se rompe y se reemplaza.
- Serigrafía con el nombre de cada señal al lado de cada conector, como hace bien el Basys 3.
  `GATEBENCH rev.A` + fecha + QR al repo. Patas de goma (el Basys 3 tiene F1–F4 para eso).

---

## 9. Entregables de software

El PCB es la mitad del trabajo. Lo que hace que la placa se use:

- `gatebench.pcf` completo y verificado, generado desde `docs/pinout.csv`
- BSP en Verilog con prefijo `gb_`: `gb_board_io`, `gb_vga_timing`, `gb_uart`, `gb_ws2812`,
  `gb_dac_r2r`, `gb_adc_mcp3204`, `gb_keypad_scan`, `gb_slow_clock`
- Ejemplos que compilan y andan: blink, barras VGA, contador en 7 segmentos, echo UART, arcoíris
  WS2812, generador de funciones (DAC), autotest DAC→ADC
- Manual de referencia estilo Basys 3: mapa de pines, decisiones eléctricas justificadas,
  cálculos de corriente. Ese doc es material de guía reutilizable

---

## 10. Registro de decisiones

| # | Decisión | Elegido | Razón |
|---|---|---|---|
| 1 | Profundidad de color VGA | **3-3-3** | 512 colores alcanzan para el UP5K; libera 3 GPIO de reserva. Footprints del 4º resistor puestos sin poblar |
| 2 | Teclado matricial | **Sólo conector 1×8** | Compatible con membranas comerciales 4×4 y con la placa del TP3. No ocupa área |
| 3 | Cantidad de dígitos | **4** | Mínimo del TP3, y coincide con el Basys 3 (KW4-281ASB, ánodos A1–A4) |
| 4 | Regulador 3.3 V | **LDO** | Más silencioso para ADC/DAC, más barato, sin inductor. **Fuerza entrada de 5 V únicamente** |

### Pendiente

- **Referencias a mirar antes de congelar el esquemático:** el **iCEBreaker** (1BitSquared) es el
  análogo directo, un UP5K educativo resuelto con Pmods. El **ULX3S** muestra hasta dónde llega la
  ambición de "workstation".
- **Nombre:** `gatebench` es provisorio hasta verificar colisiones (GitHub, Tindie, Crowd Supply,
  INPI). Reversible hasta que salgan los gerbers con la serigrafía.

---

## 11. Licencia

Licencia única para todo el repositorio: **CERN-OHL-P-2.0**. Ver `LICENSE` y §"Licencia" del
`README.md`.
