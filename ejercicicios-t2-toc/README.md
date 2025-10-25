# Ejercicios TOC

## Mario González García

---

## Lección 2 – Análisis de hardware

### 1. Sobre el circuito de la imagen

![Circuito del ejercicio 1](./public/ej1.png)

#### ¿Podría este circuito funcionar correctamente a 250 MHz?

Veamos la condición de setup:

$$
T_{\text{clk}} = \frac{1}{250 MHz} = 4ns
$$

$$
\text{Máxima Suma lógica} = MUX + ADD + MUL = (0,5 + 2,75 + 2,75)ns = 6ns
$$

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,1 + 6 + 0,15) ns \leq (4 + (0,3 - 0,2))ns \; ? \to 6,25ns \leq 4,1ns  \; ?
$$

Al ser $6,25 > 4,1$ se observa que el circuito **no podrá** funcionar a $250 MHz$.

#### Indicar el rango de retardos de MUL(as) para que el circuito funcione a 250 MHz

Dejemos a MUL como incógnita:

$$
\text{Máxima Suma lógica} = MUX + ADD + MUL = (0,5 + 2,75 + MUL)ns = (3.25 + MUL) ns
$$

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,1 + 3,25 + MUL + 0,15) ns \leq (4 + (0,3 - 0,2))ns \; \to (3,5 + MUL) ns \leq 4,1ns \to MUL \leq 0,6ns
$$

Cualquier valor inferior a $0,6ns$ satisfacerá la condición de setup

#### ¿Si $MUL(as) = 2,60ns$ puede segmentarse el circuito para que el circuito funcione a $250MHz$?

$$
\text{Máxima Suma lógica} = MUX + ADD + MUL = (0,5 + 2,75 + 2,6)ns = 5,85ns
$$

Y esto, seguirá incumpliendo la condición de setup. Pero si segmentamos tras el ADDER:

$$
\text{Máxima Suma lógica} = MUX + ADD + MUL = (0,5 + 2,75)ns = 3,25ns
$$

Por tanto, si cumpliría la condición de setup:

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,1+3,25 + 0,15)ns \leq (4,2)ns \to 3,5ns \leq 4,1ns
$$

Idem, para la segunda etapa

$$
(0,1 + 2,6 + 0,15)ns \leq (4)ns \to 2,85ns \leq 4,1ns
$$

Al insertar el pipeline después del ADD estamos añadiendo una etapa de pipeline nueva en esa ruta. Para mantener la alineación temporal en el registro 3, cualquier ruta alternativa que llegue a este y no por el nuevo registro también debe recibir un registro intermedio equivalente. En otras palabras: todas las señales que entran al registro 3 deben estar pipelineadas con la misma cantidad de etapas para este circuito.

#### ¿Se viola condición de hold con la nueva segmentación?

Al haberse asegurado la alineación con el pipeline y otra segmentación, no existe una violación de la condición de hold, cumpliendo la inecuación:

$$
t_{clk \to q}^{min} + t_{logica}^{min} \geq t_{hold} + skew
$$

### 2. Sobre el circuito de la imagen

![Circuito del ejercicio 2](./public/ej2_1.png)

Tiempos y retardos:

![Parámetros del ejercicio 2](./public/ej2_2.png)

#### ¿Podría este circuito funcionar correctamente a 100 MHz?

Existen dos rutas:

* **Ruta 1**: R_num1 : MUL : Normalize : Round : Check : MUX : R_mult
* **Ruta 2**: R_num2 : Especial : MUX : R_mult

La primera es la ruta crítica asi que calculemos si es funcional físicamente. Veamos la condición de setup:

$$
T_{\text{clk}} = \frac{1}{100 MHz} = 10ns
$$

$$
\text{Máxima Suma lógica} = MUL + Normalize + Round + Check + MUX = (5,75 + 2,5 + 1,25+ 0,5+ 0,25)ns = 10,25ns
$$

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,15 + 10,25 + 0,2) ns \leq (10 + (0,5 - 0,1))ns \; ? \to 10,6ns \leq 10,4ns  \; ?
$$

Al ser $10,6 > 10,4$ se observa que el circuito **no podrá** funcionar a $100 MHz$. Implícitamente la ruta crítica es la calculada.

#### Indicar el rango de frecuencias en las que el circuito SÍ puede funcionar

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq \frac{1}{freq} + skew
$$

$$
freq \leq \frac{1}{t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} - skew}
$$

$$
freq \leq \frac{1}{(0,15  + 10,25 + 0,2 - 0,4)ns} \leq \frac{1}{10,2ns} = 98,04 MHz
$$

Por tanto, la frecuencia máxima, la mayor a la que podrá funcionar, es de $98,04MHz$.

#### Indicar la mejora arquitectónica para que pueda funcionar a la frecuencias deseada

Para solventar la violación de setup, añadiremos un registro de segmentación tras el MUL del camino crítico de tal modo que este se divide en dos etapas:

##### Etapa 1 para setup

R_num1 : MUL : R_P

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,15 + 5,75 + 0,2)ns \leq (10 + 0,1)ns \to 6,1ns \leq 10,1ns
$$

Se cumple la condición de setup.

##### Etapa 2 para setup

R_P : Normalize : Round : Check : MUX : R_mult

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,15 + 2,5 + 1,25 + 0,5 + 0,25 + 0,2)ns \leq (10 + 0,3)ns \to 4,85ns \leq 10,3ns
$$

Se cumple la condición de setup.

#### Condición de hold

El camino de Especial sigue siendo el más rapido por lo que comprobamos la restricción de hold para este:

$$
t_{clk \to q}^{min} + t_{logica}^{min} \geq t_{hold} + skew
$$

##### Etapa 1 para hold

R_num2 : Especial : R_P

$$
(0,15 + 3,75)ns \geq (0,1 + 0,1)ns \to 3,9ns \geq 0,2ns
$$

Se cumple la condición de hold.

##### Etapa 2 para hold

R_P : MUX : R_mult

$$
(0,15 + 0,25)ns \geq (0,1 + 0,3)ns \to 0,4ns \geq 0,4ns
$$

Se cumple la condición de hold.

Como se ha calculado, las condiciones se cumplen en cada camino y cada etapa. Como se puede apreciar, se ha considerado añadir un R_P en el camino no crítico ya que si no desicronizaríamos los datos del circuito, dejando este de ser funcional.

### 3. Sobre el circuito de la imagen

![Circuito del ejercicio 3](./public/ej3_1.png)

Tiempos y retardos:

![Parámetros del ejercicio 3](./public/ej3_2.png)

#### ¿Podría este circuito funcionar correctamente a $500 MHz$? ¿A qué rangos de frecuencia puede ser funcional?

$$
T_{clk} = \frac{1}{freq} = \frac{1}{500MHz} = 2 \times 10^{-9}s = 2ns
$$

Identifiquemos el camino crítico (el otro camino sería el de realimentacion de REG1):

Reg1 - DEC - ADD - Reg2 / Reg3

Evaluemos la condición de setup:

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
(0,15 + 0,6 + 1,4 + 0,2)ns \leq (2ns - (0,15 - 0,2))ns \; \to 2,35ns \leq 1,95ns ?
$$

Al violarse la condición de setup el circuito no será funcional físicamente. Ahora encontremos la frecuencia máxima:

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq \frac{1}{freq} + skew
$$

$$
freq \leq \frac{1}{t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} - skew}
$$

$$
freq \leq \frac{1}{(0,15  + 2 + 0,2 + 0,05)ns} \leq \frac{1}{2,4ns} = 416,67 MHz
$$

Por lo que el circuito solo funcionará a menos o a $416,67 MHz$.

Igualmente evaluaremos la condición de hold:

$$
t_{clk \to q}^{min} + t_{logica}^{min} \geq t_{hold} + skew
$$

$$
0,15ns + 1,1ns \geq 0,1ns + 0ns \to 1,25ns \geq 0,1ns
$$

Se verifica la condición de hold

#### ¿A qué rango del decodificador funcionará el circuito a $500 MHz$?

Veamos que valores son necesario para que no se viole la condición de setup:

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
t_{clk \to q}^{max} + t_{logica}^{max}(\text{sin DEC}) + DEC + t_{setup} \leq T_{clk} + skew
$$

$$
DEC \leq T_{clk} + skew - t_{clk \to q}^{max} - t_{logica}^{max}(\text{sin DEC}) - t_{setup}
$$

$$
DEC \leq 2ns - 0,05ns - 0,15ns - 1,4ns - 0,2ns = 0,2ns
$$

Por tanto, si solo alteramos el retardo del decodificador, para que el circuito funcione deberá ser como máximo de 0,2ns de retardo.

#### Indicar si es posible funcionar a la frecuencia deseada usando pipelining

Para solventar mediante segmentación añadiremos 4 registros a las salidas del decodificador. Evaluamos ahora la condición de setup en las dos etapas.

##### Etapa 1 (setup): Decodificador

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
0,15ns + 0,6ns + 0,2ns \leq 2ns + (0,3 - 0,2)ns \to 0,95ns \leq 2,1ns
$$

Se verifica la condición de setup en la primera etapa.

##### Etapa 2 (setup): Sumador

$$
t_{clk \to q}^{max} + t_{logica}^{max} + t_{setup} \leq T_{clk} + skew
$$

$$
0,15ns + 1,4ns + 0,2ns \leq 2ns + (0,1 - 0,3)ns \to 1,75ns \leq 1,8ns
$$

Se verifica la condición de setup en la segunda etapa.

Ahora bien, el camino más corto ya no es el de retroalimentación (el cual no se puede segmentar por ser de feedback), es la primera nueva etapa, evaluemos la condición de hold:

##### Etapa 1 (hold): Decodificador

$$
t_{clk \to q}^{min} + t_{logica}^{min} \geq t_{hold} + skew
$$

$$
0,15ns + 0,6ns \geq 0,1ns + (0,3ns - 0,1ns) \to 0,85ns \geq 0,2ns
$$

Se verifica la condición de hold en el camino más corto, y, por ende, el circuito es funcional.
