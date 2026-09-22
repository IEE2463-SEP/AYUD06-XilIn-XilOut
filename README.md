# AYUD06 · XilIn / XilOut

> Lectura y escritura directa de registros del SoC: las funciones `Xil_In32()` y `Xil_Out32()` reemplazan a los drivers de Xilinx para hablar con los periféricos AXI de un MicroBlaze, primero sobre un AXI GPIO y después sobre un IP-Core propio con un mapa de registros definido por uno mismo.

Ayudante a cargo: **Pablo Uribe Arredondo** — parred@uc.cl

Esta ayudantía se divide en dos partes:

| | Qué es | Cuándo se hace |
| :--- | :--- | :--- |
| **Actividades previas** | Un *Block Design* con un MicroBlaze, un IP-Core propio por AXI Lite y un AXI GPIO con los switches y el LED RGB, más el proyecto de Vitis que lo maneja con `Xil_In32()`, `Xil_Out32()` y las funciones del AXI GPIO. | **Antes**, en su casa. |
| **Ejercicio propuesto** | Botar los drivers `XGpio_*` y escribir los propios, y después reemplazar el AXI GPIO por un IP-Core propio con mapa de registros `CTRL`, `DUTY`, `SW` e `ID`. | **Durante** la ayudantía. |

---

## 🎥 Antes de la ayudantía

Debes llegar a la sesión con las **actividades previas ya desarrolladas y funcionando en la tarjeta**: un proyecto de Vivado con un procesador **MicroBlaze** en el *Block Design*, los IP-Cores que necesita para funcionar (*Run Block Automation* y *Run Connection Automation*), un IP-Core propio conectado por **AXI Lite** que contiene la lógica PWM, y un **AXI GPIO** con los switches y el LED RGB conectados. Sobre ese hardware, el proyecto de Vitis con su parte de *hardware* y *software*, escrito con `Xil_In32()`, `Xil_Out32()` y las funciones del AXI GPIO. Todo el detalle, incluido el diagrama del *Block Design* final, está en el [enunciado](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/AYUD06-XilIn-XilOut.pdf).

Estas actividades están resueltas paso a paso en este video, grabado el año **2023 por el ex ayudante del curso Cristóbal Vásquez**:

[![Video de la ayudantía 06](https://img.youtube.com/vi/iRbIIK3E60Y/hqdefault.jpg)](https://youtu.be/iRbIIK3E60Y)

**Apoyarse en el video es opcional.** Puede seguirlo completo, usarlo sólo para destrabar un punto puntual, o resolver las actividades por su cuenta: eso lo decide usted. Lo que no es opcional es llegar con el diseño andando, porque el tiempo de la ayudantía se destinará por completo al ejercicio propuesto.

---

## 📂 Material

| Archivo | Descripción |
| :--- | :--- |
| [AYUD06-XilIn-XilOut.pdf](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/AYUD06-XilIn-XilOut.pdf) | Enunciado de la ayudantía: actividades previas, ejercicio propuesto, diagrama del *Block Design* y mapa de registros del IP-Core pedido. |
| [main.c](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/main.c) | Código en C tal como queda al terminar las actividades previas: escribe los cuatro comparadores del IP-Core con `Xil_Out32()`, lee los switches con `Xil_In32()` y deja a la vista las llamadas `XGpio_*` equivalentes. Es el punto de partida del ejercicio propuesto. |
| [LED_DRIVER_v1_0.vhd](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/LED_DRIVER_v1_0.vhd) | Fuente de mayor jerarquía del IP-Core AXI Lite: expone `clk` y la salida `leds_out` de 4 bits, e instancia la interfaz AXI. |
| [LED_DRIVER_v1_0_S00_AXI.vhd](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/LED_DRIVER_v1_0_S00_AXI.vhd) | Fuente de menor jerarquía: la interfaz AXI Lite con sus cuatro registros esclavos (`slv_reg0` a `slv_reg3`), cada uno comparado contra un contador diente de sierra para generar el PWM de un LED. |
| [Zybo-Z7-Master.xdc](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/Zybo-Z7-Master.xdc) | Constraints de la tarjeta, con el reloj, el reset, los 4 switches, los 4 LEDs y el LED RGB habilitados. |
| [AYU06.zip](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/AYU06.zip) | Proyecto completo de la ayudantía: el proyecto de Vivado, el IP-Core en `ip_repo/` y el *workspace* de Vitis. |
| [AY06_vitis_export.zip](https://github.com/IEE2463-SEP/AYUD06-XilIn-XilOut/blob/HEAD/AY06_vitis_export.zip) | Exportación del *workspace* de Vitis (`AYU06_HW`, `AYU06_SW` y `AYU06_SW_system`), con el `.xsa`, el *bitstream* y el `main.c`, para importar sin descargar el proyecto completo. |

---

## 🧪 Durante la ayudantía

El **ejercicio propuesto** es el trabajo de la sesión, y fue propuesto y desarrollado por el ayudante de este semestre, **Pablo Uribe Arredondo**. Se parte del proyecto de las actividades previas, que todavía se apoya en los drivers que vienen hechos, y se llega a un periférico propio manejado enteramente a punta de lecturas y escrituras de registros:

**1. Driver propio para el AXI GPIO.** Eliminar todas las llamadas a funciones `XGpio_*` —junto con el `#include "xgpio.h"`— y reemplazarlas por funciones propias escritas sólo con `Xil_In32()` y `Xil_Out32()`, que permitan configurar la dirección de cada canal, escribir en los LEDs RGB y leer los switches. Para esto se usa el mapa de registros del AXI GPIO descrito en su guía de producto (PG144) y la dirección base asignada en el *Address Editor* de Vivado.

**2. IP-Core propio con mapa de registros definido.** Reemplazar el AXI GPIO por un IP-Core AXI4-Lite creado manualmente (*Create and Package New IP*) que controle el LED RGB mediante PWM y lea los switches, respetando este mapa de registros:

| Offset | Registro | Acceso | Descripción |
| :--- | :--- | :--- | :--- |
| `0x00` | `CTRL` | L/E | Bit 0 (`EN`): 1 habilita la salida PWM, 0 apaga el LED. |
| `0x04` | `DUTY` | L/E | Ciclo de trabajo de 0 a 255 por color: `[7:0]` rojo, `[15:8]` verde, `[23:16]` azul. |
| `0x08` | `SW` | L | Estado actual de los switches. Los bits sin switch asociado se leen como 0. Las escrituras se ignoran. |
| `0x0C` | `ID` | L | Valor constante `0x24632026`. |

> ⚠️ La plantilla que genera Vivado deja los cuatro registros como lectura/escritura: **debe modificar la lógica de lectura para que `SW` e `ID` queden de sólo lectura**. El período del PWM queda a su criterio, siempre que el LED no parpadee a simple vista.

El programa en Vitis debe comunicarse con el IP-Core **únicamente** mediante `Xil_In32()` y `Xil_Out32()`, y cumplir con lo siguiente:

- Al iniciar, leer `ID` y comprobar que vale `0x24632026`. Así se verifica que la dirección base usada es la correcta.
- Luego, escribir `0xFFFFFFFF` en `SW` y leerlo de vuelta. Si se lee el mismo valor, la escritura tuvo efecto y el registro no es de sólo lectura.
- Si alguna de las dos comprobaciones falla, el programa debe detenerse sin entrar al funcionamiento normal, dejando el LED apagado.
- Si ambas comprobaciones pasan, leer continuamente `SW` y usar los cuatro switches menos significativos como índice de una tabla de 16 colores definida en el programa. El índice 0 deja `EN` en 0 (LED apagado); cualquier otro índice escribe el color correspondiente en `DUTY` y deja `EN` en 1.

> 💡 Se sugiere, aunque no es obligatorio, usar la UART o el depurador de Vitis para revisar los valores leídos en cada comprobación, ya que facilita encontrar errores. La UART se verá en detalle en una ayudantía posterior.

### 📤 Entrega y bonificación

El desarrollo del ejercicio propuesto se sube a **Canvas el mismo día de la ayudantía, hasta las 14:50**. Entregarlo dentro de plazo otorga **una décima (+0,1)** en la nota del **Proyecto 1 y 2. Si, solo para esta ayudantía, el ejercicio propuesto otorga una décima a cada proyecto**.

---

<sub>IEE2463 · Sistemas Electrónicos Programables</sub>
