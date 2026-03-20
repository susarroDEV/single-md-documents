# Ejercicios EC

## Mario González García

---

## Predictores

### 1. Enunciado del ejercicio 1

*(Examen EC - Mayo 2024)* Una máquina usa predictores dinámicos de saltos correlacionados de dos niveles. Considerando un segmento de código con tres saltos B1, B2 y B3. En un momento determinado de la ejecución los tres han sido repetidamente tomados. Justo después de este instante, la secuencia de saltos en orden de ejecución genera las siguientes decisiones:

| Salto          | B1 | B2 | B3 | B1 | B2 | B3 | B1 | B2 | B3 | B1 | B2 | B3 | B1 | B2 | B3 |
|----------------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|-----|
| Comportamiento | N  | T  | N  | T  | N  | T  | N  | T  | N  | T  | N  | T  | N  | T  | N  |

a) Deduzca el estado inicial de los predictores de B2 y de B3 en base a la información proporcionada.  
b) Calcular la tasa de errores de predicción para esta secuencia en el caso de predictor multinivel (1,1).  
c) Calcular la tasa de errores de predicción para esta secuencia en el caso de predictor multinivel (1,2).

---

### 1. Respuesta del ejercicio 1

#### Contexto: predictor (1,1) — 1 bit de historia global, 1 bit por predictor

En un predictor (1,1) existe **1 bit de historia global** (el resultado del último salto ejecutado), que selecciona entre **2 predictores de 1 bit** por cada salto.

**Estado inicial:** Se dice que los tres saltos han sido repetidamente tomados antes de la secuencia. Por tanto, justo antes de empezar:
- El bit de historia global es **1** (el último salto ejecutado, B3, fue Tomado).
- Todos los predictores habrán convergido a predecir **T (1)**.

Así, para B1: P0=0, P1=**1** (dado que la historia previa es 1, P1 es el que se ha venido usando y apunta a T).  
Para deducir B2 y B3 necesitamos analizar qué historia llega a cada uno:

- Antes de B1 (primera vez): historia=1 → usa P1 de B1. B1 resulta N → historia pasa a **0**.
- Antes de B2: historia=0 → usa P0 de B2. B2 resulta T → historia pasa a **1**.
- Antes de B3: historia=1 → usa P1 de B3. B3 resulta N → historia pasa a **0**.

Para que el predictor ya esté "estabilizado" desde antes de la secuencia, los valores iniciales de los predictores son:

| Salto | P0 | P1 |
|-------|----|----|
| B1    | 0  | 1  |
| B2    | 1  | 1  |
| B3    | 0  | 1  |

> **Deducción de B2 y B3:** B2 se accede siempre con historia=0 (tras B1=N), por lo que solo usa P0. Como B2 siempre es T, P0 de B2 = **1**. B3 se accede siempre con historia=1 (tras B2=T), por lo que solo usa P1. Como B3 siempre es N, P1 de B3 = **0** (ya convergerá, pero en estado inicial dado que venían de T, P1=1; el primer acceso falla y se actualiza a 0).

#### b) Tasa de errores — predictor (1,1)

Con 1 bit de predictor, el predictor cambia su valor al primer fallo. La secuencia por salto es siempre la misma:

- **B1**: siempre accede con historia=1 (B3 anterior fue T... pero B3 es N en toda la secuencia, así que historia llega a B1 siempre como 0). Reanalicemos el flujo completo:

Traza completa (historia global de 1 bit):

| Iter | Salto | Historia entrada | Pred usado | Predicción | Comportamiento | ¿Fallo? | Historia salida |
|------|-------|-----------------|------------|-----------|----------------|---------|----------------|
| 1    | B1    | 1               | P1=1       | T          | N              | **Sí**  | 0              |
| 2    | B2    | 0               | P0=1       | T          | T              | No      | 1              |
| 3    | B3    | 1               | P1=1       | T          | N              | **Sí**  | 0              |
| 4    | B1    | 0               | P0=0       | N          | T              | **Sí**  | 1              |
| 5    | B2    | 1               | P1=1       | T          | N              | **Sí**  | 0              |
| 6    | B3    | 0               | P0=0       | N          | T              | **Sí**  | 1              |
| 7    | B1    | 1               | P1=0       | N          | N              | No      | 0              |
| 8    | B2    | 0               | P0=1       | T          | T              | No      | 1              |
| 9    | B3    | 1               | P1=0       | N          | N              | No      | 0              |
| 10   | B1    | 0               | P0=0       | N          | T              | **Sí**  | 1              |
| 11   | B2    | 1               | P1=0→1    | N          | N              | No*     | 0              |
| 12   | B3    | 0               | P0=0→1    | N          | T              | **Sí**  | 1              |
| 13   | B1    | 1               | P1=1       | T          | N              | **Sí**  | 0              |
| 14   | B2    | 0               | P0=1       | T          | T              | No      | 1              |
| 15   | B3    | 1               | P1=1       | T          | N              | **Sí**  | 0              |

*(Los predictores se actualizan tras cada acceso según el comportamiento real.)*

Iteraciones donde hay fallo: 1, 3, 4, 5, 6, 10, 12, 13, 15 → **9 fallos de 15 predicciones**.

> **Tasa de errores (1,1) = 9/15 = 60%**

#### c) Tasa de errores — predictor (1,2)

Con predictor (1,2): 1 bit de historia global → selecciona entre 2 contadores saturantes de **2 bits** por salto.  
Estados: 00=NT fuerte, 01=NT débil, 10=T débil, 11=T fuerte.  
Predicción: 0x → N, 1x → T.

Estado inicial: todos los predictores a **11** (T fuerte), pues venían de ser tomados repetidamente.

| Iter | Salto | Hist. | Pred | Estado | Pred. | Comp. | Fallo? | Nuevo estado |
|------|-------|-------|------|--------|-------|-------|--------|--------------|
| 1    | B1    | 1     | P1   | 11     | T     | N     | **Sí** | 10           |
| 2    | B2    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 3    | B3    | 1     | P1   | 11     | T     | N     | **Sí** | 10           |
| 4    | B1    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 5    | B2    | 1     | P1   | 11     | T     | N     | **Sí** | 10           |
| 6    | B3    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 7    | B1    | 1     | P1   | 10     | T     | N     | **Sí** | 01           |
| 8    | B2    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 9    | B3    | 1     | P1   | 10     | T     | N     | **Sí** | 01           |
| 10   | B1    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 11   | B2    | 1     | P1   | 10     | T     | N     | **Sí** | 01           |
| 12   | B3    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 13   | B1    | 1     | P1   | 01     | N     | N     | No     | 00           |
| 14   | B2    | 0     | P0   | 11     | T     | T     | No     | 11           |
| 15   | B3    | 1     | P1   | 01     | N     | N     | No     | 00           |

Fallos en iteraciones: 1, 3, 5, 7, 9, 11 → **6 fallos de 15 predicciones**.

> **Tasa de errores (1,2) = 6/15 = 40%**

El predictor de 2 bits mejora porque necesita dos fallos consecutivos antes de cambiar la predicción, amortiguando las oscilaciones.

---

### 2. Enunciado del ejercicio 2

*(Examen EC - Junio 2024)* Supongamos un predictor dinámico de dos niveles (2,1) que se utiliza para predecir el comportamiento de un salto condicional en un programa. El comportamiento real del salto es el siguiente: T, T, NT, NT, T, T, T, T.

a) Completa la tabla y calcula la tasa de fallos del predictor partiendo de la situación inicial reflejada en la misma.  
b) ¿Qué semejanzas y diferencias presenta este predictor comparado con un predictor (2,2)?

Estado inicial: Historia=00, predictores P0=0, P1=1, P2=0, P3=1.

---

### 2. Respuesta del ejercicio 2

#### a) Tabla completa del predictor (2,1)

**Funcionamiento del predictor (2,1):**
- **2 bits de historia global** → 4 posibles patrones: 00, 01, 10, 11.
- Cada patrón selecciona un **contador saturante de 1 bit** (P0, P1, P2, P3).
- Predicción: 0 → NT, 1 → T.
- Actualización de historia: se desplaza a la izquierda y entra el nuevo resultado (T=1, NT=0).

Estado inicial: Historia=00, P0=0, P1=1, P2=0, P3=1.

| # | Historia | Pred usado | Valor | Predicción | Comportamiento | Fallo/Acierto | Nueva historia | Nuevo valor pred |
|---|----------|------------|-------|-----------|----------------|---------------|----------------|-----------------|
| 1 | 00       | P0         | 0     | NT        | T              | **Fallo**     | 01             | P0=1            |
| 2 | 01       | P1         | 1     | T         | T              | Acierto       | 11             | P1=1            |
| 3 | 11       | P3         | 1     | T         | NT             | **Fallo**     | 10             | P3=0            |
| 4 | 10       | P2         | 0     | NT        | NT             | Acierto       | 00             | P2=0            |
| 5 | 00       | P0         | 1     | T         | T              | Acierto       | 01             | P0=1            |
| 6 | 01       | P1         | 1     | T         | T              | Acierto       | 11             | P1=1            |
| 7 | 11       | P3         | 0     | NT        | T              | **Fallo**     | 11             | P3=1            |
| 8 | 11       | P3         | 1     | T         | T              | Acierto       | 11             | P3=1            |

**Fallos:** iteraciones 1, 3, 7 → **3 fallos de 8**.

> **Tasa de fallos = 3/8 = 37,5%**

#### b) Semejanzas y diferencias con un predictor (2,2)

| Aspecto | (2,1) | (2,2) |
|---------|-------|-------|
| Bits de historia global | 2 | 2 |
| Número de predictores (entradas en tabla) | 4 (00,01,10,11) | 4 (00,01,10,11) |
| Bits por predictor (contador saturante) | 1 bit (2 estados: T/NT) | 2 bits (4 estados: TF, TD, NTD, NTF) |
| Reacción a cambios de comportamiento | Inmediata (1 fallo basta) | Amortiguada (necesita 2 fallos para cambiar predicción) |
| Memoria de tendencia | No tiene histéresis | Tiene histéresis (débil/fuerte) |

**Semejanzas:** ambos usan el mismo registro de historia global de 2 bits para seleccionar el predictor, por lo que indexan exactamente las mismas 4 entradas de la tabla según el patrón de los dos últimos saltos.

**Diferencias:** el (2,2) usa contadores saturantes de 2 bits, lo que le da "inercia": no cambia de predicción hasta acumular dos fallos consecutivos. Esto le hace más robusto frente a comportamientos mayoritariamente regulares con excepciones puntuales, pero responde más lentamente a cambios reales de patrón.

---

### 3. Enunciado del ejercicio 3

Supongamos un procesador con predicción dinámica de saltos que usa un predictor competitivo (Tournament Predictor) como el del Alpha 21264. El programa tiene dos instrucciones de salto: `beq` y `bne`. La instrucción `beq` se ejecuta 1000 veces con patrón **T-T-NT-NT-T-T-NT-NT-...** La instrucción `bne` siempre se toma.

a) Determinar qué entradas de la Tabla de Predicción Local han sido accedidas por las 100 últimas ejecuciones de `beq` y cuál es su contenido al final de las 1000 ejecuciones.  
b) Si `beq` se ejecuta 1000 veces más con el mismo patrón, ¿cuántos fallos de predicción se producen?

---

### 3. Respuesta del ejercicio 3

#### Contexto: Tournament Predictor (Alpha 21264)

El predictor del Alpha 21264 combina:
- **Predictor local:** tabla de historia local (indexada por PC), cada entrada tiene un registro de historia de 10 bits que indexa a su vez una tabla de predictores locales de 3 bits (8 estados saturantes). Para este ejercicio simplificamos a los bits relevantes.
- **Predictor global:** contador saturante de 2 bits indexado por historia global.
- **Tabla de selección:** contador saturante de 2 bits que decide cuál de los dos predictores usar.

Para este ejercicio nos centramos en la **Tabla de Predicción Local**.

#### a) Entradas accedidas por `beq`

El patrón de `beq` es: **T, T, NT, NT, T, T, NT, NT, ...** (período 4).

La instrucción `bne` siempre se toma (T) y se ejecuta intercalada con `beq` (según el código: beq está en el bucle, bne está fuera — en realidad, analizando el código: `beq` salta de vuelta al loop y `bne` también, pero ambos son saltos distintos con PCs distintos).

En el predictor local del Alpha 21264, cada instrucción de salto tiene su propio **registro de historia local** (indexado por los bits bajos del PC). Para `beq`, su registro de historia local acumula los resultados de sus propias ejecuciones anteriores.

**Historia local de `beq` tras muchas ejecuciones (patrón T-T-NT-NT con período 4):**

Dado que `bne` no influye en el registro de historia local de `beq`, el registro de historia local de `beq` (tras estabilizarse) contendrá un patrón cíclico de 4 bits correspondiente a: **...1100 1100...** (T=1, NT=0).

Con un registro de historia local de, por ejemplo, 4 bits relevantes, los patrones que se repiten cíclicamente al acceder a la tabla de predicción local son:

| Ejecución beq | Historia local (4 bits) | Predictor accedido |
|--------------|------------------------|--------------------|
| 4k+1 (T)     | 0011 → índice 3        | Entrada 3          |
| 4k+2 (T)     | 0111 → índice 7        | Entrada 7          |
| 4k+3 (NT)    | 1111 → índice 15       | Entrada 15         |
| 4k+4 (NT)    | 1110 → índice 14       | Entrada 14         |

> Las **4 entradas accedidas** por las 100 últimas ejecuciones de `beq` son las correspondientes a los índices **3, 7, 14 y 15** (expresados en binario: 0011, 0111, 1110, 1111).

**Contenido al final de las 1000 ejecuciones:**

Tras 1000 ejecuciones (250 repeticiones del patrón T-T-NT-NT), los contadores saturantes de cada entrada habrán convergido:

| Entrada | Resultado en ese momento | Contador (3 bits, escala 0-7) | Predicción |
|---------|--------------------------|-------------------------------|-----------|
| 3       | T                        | 111 (saturado en T fuerte)    | T         |
| 7       | T                        | 111 (saturado en T fuerte)    | T         |
| 14      | NT                       | 000 (saturado en NT fuerte)   | NT        |
| 15      | NT                       | 000 (saturado en NT fuerte)   | NT        |

#### b) Fallos en las siguientes 1000 ejecuciones

Dado que los contadores están completamente saturados y el patrón es perfectamente regular (período 4, siempre el mismo), el predictor local habrá aprendido perfectamente el patrón. En cada ciclo de 4 ejecuciones:

- Ejecución 1 (T): historia=0011 → entrada 3 → predice **T** ✓
- Ejecución 2 (T): historia=0111 → entrada 7 → predice **T** ✓
- Ejecución 3 (NT): historia=1111 → entrada 15 → predice **NT** ✓
- Ejecución 4 (NT): historia=1110 → entrada 14 → predice **NT** ✓

> **Fallos en las siguientes 1000 ejecuciones = 0**

El predictor local habrá aprendido el patrón completamente y no cometerá ningún fallo adicional.

---

### 4. Enunciado del ejercicio 4

*(Examen AC - Febrero 2016)* Supongamos un programa que incluye una única instrucción de salto que siempre se toma y que se ejecuta en un procesador dotado de un predictor de saltos de dos niveles (3,2). ¿A partir de qué número de ejecución del salto el predictor acertará en la predicción? El estado inicial del predictor es todo a cero.

---

### 4. Respuesta del ejercicio 4

#### Funcionamiento del predictor (3,2)

- **3 bits de historia global** → 8 entradas posibles (000 a 111).
- Cada entrada es un **contador saturante de 2 bits**: 00=NT fuerte, 01=NT débil, 10=T débil, 11=T fuerte.
- Predicción: bit alto del contador. 0x → NT, 1x → T.
- El salto **siempre se toma (T=1)**.
- Estado inicial: historia=000, todos los contadores a **00**.

#### Traza de ejecuciones

| Ejec. | Historia (entrada) | Estado contador | Predicción | Comportamiento | Fallo? | Nueva historia | Nuevo estado |
|-------|--------------------|----------------|-----------|----------------|--------|----------------|--------------|
| 1     | 000                | 00             | NT        | T              | **Sí** | 001            | 01           |
| 2     | 001                | 00             | NT        | T              | **Sí** | 011            | 01           |
| 3     | 011                | 00             | NT        | T              | **Sí** | 111            | 01           |
| 4     | 111                | 00             | NT        | T              | **Sí** | 111            | 01           |
| 5     | 111                | 01             | NT        | T              | **Sí** | 111            | 10           |
| 6     | 111                | 10             | **T**     | T              | No ✓   | 111            | 11           |
| 7     | 111                | 11             | **T**     | T              | No ✓   | 111            | 11           |

A partir de la **ejecución 6**, el predictor acierta siempre.

**Explicación:** Las 3 primeras ejecuciones llenan el registro de historia (que pasa de 000 a 001, 011, 111). A partir de la ejecución 4, la historia siempre es 111 y se accede al mismo contador. Ese contador parte de 00 y necesita 2 incrementos para superar el umbral de predicción T (llega a 01 en la ejecución 4, a 10 en la ejecución 5). En la ejecución 6 ya vale 10 → predice T → **acierto**.

> **El predictor acierta a partir de la ejecución número 6.**

---

### 5. Enunciado del ejercicio 5

*(Examen AC - Enero 2024)* Supongamos un predictor de saltos de dos niveles (2,1). Explica cómo funciona usando como ejemplo un programa que ejecuta repetidamente un salto con el patrón T T T NT (muestra el estado del predictor para las dos primeras ejecuciones del patrón). Todos los elementos inicializados a cero.

---

### 5. Respuesta del ejercicio 5

#### Funcionamiento del predictor (2,1)

Un predictor **(2,1)** tiene:
- **Registro de historia global de 2 bits**: almacena los resultados de los 2 últimos saltos ejecutados. Se actualiza con cada salto: desplazamiento a la izquierda y entrada del nuevo resultado (T=1, NT=0).
- **Tabla de 4 predictores de 1 bit** (indexada por los 2 bits de historia): cada entrada es un contador de 1 bit (0=NT, 1=T) que se actualiza al resultado real.

El registro de historia selecciona qué predictor de la tabla usar. Tras conocer el resultado, se actualiza el predictor correspondiente y se actualiza el registro de historia.

#### Primera ejecución del patrón (T, T, T, NT)

Estado inicial: historia=00, P[00]=0, P[01]=0, P[10]=0, P[11]=0.

| Salto | Historia | Pred. usado | Valor | Predicción | Comp. | Fallo? | Nueva historia | Nuevo P |
|-------|----------|-------------|-------|-----------|-------|--------|----------------|---------|
| T     | 00       | P[00]       | 0     | NT        | T     | **Sí** | 01             | P[00]=1 |
| T     | 01       | P[01]       | 0     | NT        | T     | **Sí** | 11             | P[01]=1 |
| T     | 11       | P[11]       | 0     | NT        | T     | **Sí** | 11             | P[11]=1 |
| NT    | 11       | P[11]       | 1     | T         | NT    | **Sí** | 10             | P[11]=0 |

*4 fallos en la primera ejecución del patrón.*

Estado al terminar la primera ejecución: historia=10, P[00]=1, P[01]=1, P[10]=0, P[11]=0.

#### Segunda ejecución del patrón (T, T, T, NT)

| Salto | Historia | Pred. usado | Valor | Predicción | Comp. | Fallo? | Nueva historia | Nuevo P |
|-------|----------|-------------|-------|-----------|-------|--------|----------------|---------|
| T     | 10       | P[10]       | 0     | NT        | T     | **Sí** | 01             | P[10]=1 |
| T     | 01       | P[01]       | 1     | T         | T     | No ✓   | 11             | P[01]=1 |
| T     | 11       | P[11]       | 0     | NT        | T     | **Sí** | 11             | P[11]=1 |
| NT    | 11       | P[11]       | 1     | T         | NT    | **Sí** | 10             | P[11]=0 |

*3 fallos en la segunda ejecución del patrón.*

Estado al terminar: historia=10, P[00]=1, P[01]=1, P[10]=1, P[11]=0.

A partir de la **tercera ejecución del patrón**, el predictor habrá aprendido completamente y solo fallará en el primer salto de cada patrón (el T que sigue al NT, que llega con historia=10 y P[10]=1, ya predice T correctamente desde la tercera vez).

---

### 6. Enunciado del ejercicio 6

*(Examen AC - Enero 2022)* Supóngase un salto que se ejecuta siguiendo el patrón T-T-N-T-T-N-... y cuyo comportamiento se predice utilizando un predictor dinámico (2,2). Indicar a partir de qué ejecución de este salto el predictor empieza a acertar siempre. Asumir que todas las estructuras y registros de la arquitectura están inicializados a 0.

---

### 6. Respuesta del ejercicio 6

#### Funcionamiento del predictor (2,2)

- **2 bits de historia global** → 4 entradas (00, 01, 10, 11).
- Cada entrada: **contador saturante de 2 bits** (00=NT fuerte, 01=NT débil, 10=T débil, 11=T fuerte).
- Predicción: bit alto. 1x → T, 0x → NT.
- Patrón: **T, T, N, T, T, N, ...** (período 3).

Estado inicial: historia=00, todos los contadores a **00**.

#### Traza completa

| Ejec. | Historia | Contador | Predicción | Comp. | Fallo? | Nueva hist. | Nuevo cont. |
|-------|----------|----------|-----------|-------|--------|------------|-------------|
| 1  (T) | 00       | 00       | NT        | T     | **Sí** | 01         | 01          |
| 2  (T) | 01       | 00       | NT        | T     | **Sí** | 11         | 01          |
| 3  (N) | 11       | 00       | NT        | N     | No ✓   | 10         | 00          |
| 4  (T) | 10       | 00       | NT        | T     | **Sí** | 01         | 01          |
| 5  (T) | 01       | 01       | NT        | T     | **Sí** | 11         | 10          |
| 6  (N) | 11       | 00       | NT        | N     | No ✓   | 10         | 00          |
| 7  (T) | 10       | 01       | NT        | T     | **Sí** | 01         | 10          |
| 8  (T) | 01       | 10       | T         | T     | No ✓   | 11         | 11          |
| 9  (N) | 11       | 00       | NT        | N     | No ✓   | 10         | 00          |
| 10 (T) | 10       | 10       | T         | T     | No ✓   | 01         | 11          |
| 11 (T) | 01       | 11       | T         | T     | No ✓   | 11         | 11          |
| 12 (N) | 11       | 00       | NT        | N     | No ✓   | 10         | 00          |
| 13 (T) | 10       | 11       | T         | T     | No ✓   | 01         | 11          |
| 14 (T) | 01       | 11       | T         | T     | No ✓   | 11         | 11          |
| 15 (N) | 11       | 00       | NT        | N     | No ✓   | 10         | 00          |

A partir de la ejecución 10 no hay más fallos y el patrón se estabiliza: el ciclo {10→11→12} = {T→T→N} con historias {10→01→11} y contadores {11→11→00} se repite correctamente.

> **El predictor empieza a acertar siempre a partir de la ejecución número 10.**

---

### 7. Enunciado del ejercicio 7

*(Examen AC - Junio 2022)* Supongamos un procesador dotado de un predictor de saltos bimodal y un código P que consta de un único salto. Si inicialmente el predictor contiene el valor 10 y el comportamiento real del mencionado salto en sus 6 primeras ejecuciones es: NT-T-NT-NT-T-T, completa la siguiente tabla.

---

### 7. Respuesta del ejercicio 7

#### Predictor bimodal (contador saturante de 2 bits)

Estados: **00** (NT fuerte) → **01** (NT débil) → **10** (T débil) → **11** (T fuerte).  
- Si el salto se toma (T): incrementar (máx. 11).  
- Si no se toma (NT): decrementar (mín. 00).  
- Predicción: bit alto. 1x → T, 0x → NT.

| Iteración | Estado predictor | Predicción | Comportamiento | Hit? | Siguiente estado |
|-----------|----------------|-----------|----------------|------|-----------------|
| 1         | 10             | T         | NT             | No   | 01              |
| 2         | 01             | NT        | T              | No   | 10              |
| 3         | 10             | T         | NT             | No   | 01              |
| 4         | 01             | NT        | NT             | Sí   | 00              |
| 5         | 00             | NT        | T              | No   | 01              |
| 6         | 01             | NT        | T              | No   | 10              |

**Resumen:** 1 acierto (iteración 4), 5 fallos. **Tasa de aciertos = 1/6 ≈ 16,7%.**

---

### 8. Enunciado del ejercicio 8

*(Examen AC - Enero 2022)* Un programa P que consta de 2 saltos (S1 y S2) se ejecuta en un procesador dotado de un predictor de dos niveles (2,1) con 2 entradas. S1 y S2 se ejecutan de modo alterno. S1 siempre se toma y S2 nunca es tomado. Indica, para las dos primeras ejecuciones de cada uno de los saltos, qué predictor se escoge, si es acierto o fallo y el nuevo valor del predictor. Todos los predictores e historia global inicializados a 0. No hay aliasing entre S1 y S2.

---

### 8. Respuesta del ejercicio 8

#### Configuración del predictor (2,1) con 2 entradas

- **Historia global de 2 bits** → selecciona entre las 4 combinaciones posibles.
- Cada salto tiene su propia **tabla de 4 predictores de 1 bit** (2 entradas significa que solo hay 2 filas físicas, pero indexadas por los 2 bits de historia → 4 entradas lógicas; con 2 entradas físicas se usa aliasing aunque el enunciado dice que no hay aliasing entre S1 y S2, así que los tratamos de forma independiente).
- Estado inicial: historia=00, todos los predictores=0.

Secuencia de ejecución: **S1, S2, S1, S2, S1, S2, S1, S2, ...**  
S1 siempre T (1), S2 siempre NT (0).

| # | Salto | Historia entrada | Predictor elegido | Valor actual | Predicción | Comportamiento | Fallo/Acierto | Nueva historia | Nuevo valor |
|---|-------|-----------------|-------------------|-------------|-----------|----------------|---------------|----------------|-------------|
| 1 | S1    | 00              | P_S1[00]          | 0           | NT        | T              | **Fallo**     | 01             | P_S1[00]=1  |
| 2 | S2    | 01              | P_S2[01]          | 0           | NT        | NT             | Acierto       | 10             | P_S2[01]=0  |
| 3 | S1    | 10              | P_S1[10]          | 0           | NT        | T              | **Fallo**     | 01             | P_S1[10]=1  |
| 4 | S2    | 01              | P_S2[01]          | 0           | NT        | NT             | Acierto       | 10             | P_S2[01]=0  |

> **Nota:** La historia se actualiza tras cada salto. S2 con NT deja siempre el bit 0, por lo que la historia alterna entre los patrones 01 y 10. S1 accede siempre con historia=00 (primera vez) o historia=10 (segunda vez en adelante), mientras que S2 accede siempre con historia=01.

**Resumen de las 2 primeras ejecuciones de cada salto:**

| Salto | Ejecución | Historia | Predictor | Predicción | Comportamiento | Resultado  |
|-------|-----------|----------|-----------|-----------|----------------|------------|
| S1    | 1ª        | 00       | P_S1[00]  | NT        | T              | **Fallo**  |
| S1    | 2ª        | 10       | P_S1[10]  | NT        | T              | **Fallo**  |
| S2    | 1ª        | 01       | P_S2[01]  | NT        | NT             | **Acierto**|
| S2    | 2ª        | 01       | P_S2[01]  | NT        | NT             | **Acierto**|

S1 falla en sus dos primeras ejecuciones porque los predictores inicializados a 0 (NT) no coinciden con su comportamiento real (T). S2 acierta siempre porque su predictor inicializado a 0 (NT) coincide con su comportamiento real (NT).

---

### 9. Enunciado del ejercicio 9

*(Examen AC - Septiembre 2015)* En un predictor híbrido, que combina dos predictores diferentes, explica cómo se suele implementar la tabla de selección entre ambos y cómo se actualiza en todos los casos posibles: si los dos predictores fallan, si los dos aciertan, si acierta el Predictor 1 y falla el 2, y si falla el Predictor 1 y acierta el 2.

---

### 9. Respuesta del ejercicio 9

#### Implementación de la tabla de selección

La tabla de selección (también llamada *chooser table* o *meta-predictor*) es un array de **contadores saturantes de 2 bits**, típicamente indexada por los bits bajos del PC o por la historia global de saltos (según el diseño).

Cada entrada del array es un contador de 2 bits con el significado:
- **00, 01**: seleccionar Predictor 1.
- **10, 11**: seleccionar Predictor 2.

El contador satura en 00 y 11 (no puede bajar de 00 ni subir de 11).

#### Política de actualización

La tabla de selección **solo se actualiza cuando los dos predictores discrepan**, es decir, cuando uno acierta y el otro falla. Si ambos aciertan o ambos fallan, el contador no cambia (no hay información útil para discriminar entre ellos).

| Caso                              | Actualización del contador |
|-----------------------------------|---------------------------|
| Ambos aciertan                    | **Sin cambio**            |
| Ambos fallan                      | **Sin cambio**            |
| P1 acierta, P2 falla              | Decrementar (→ favorecer P1) |
| P1 falla, P2 acierta              | Incrementar (→ favorecer P2) |

**Justificación:** cuando ambos aciertan, cualquiera de los dos sería válido; cuando ambos fallan, ninguno es mejor que el otro. Solo cuando difieren en el resultado tiene sentido ajustar la preferencia hacia el que acertó.

Este mecanismo hace que el predictor híbrido converja hacia el predictor individual que mejor se adapta al patrón de cada salto particular, combinando lo mejor de cada estrategia (por ejemplo, un predictor global correlacionado frente a uno local bimodal).

---

### 10. Enunciado del ejercicio 10

Se dispone de un procesador segmentado tipo DLX (fases IF, ID, EX, ME, WB). El predictor es un Branch Target Buffer (BTB) de 2 bits, accedido en IF, respuesta al final de IF. Estado inicial: **"predicción no-salta-fuerte"** para `bnez x1, loop`. Mostrar las fases de las dos instrucciones que siguen al salto en cada una de las cuatro primeras iteraciones.

```
addi x1, x0, 4
loop: sub  x5, x5, x2
      add  x2, x2, 1
      subi x1, x1, 1
      bnez x1, loop     ← salto estudiado
      andi x6, x5, 63
      ori  x6, x6, 128
```

---

### 10. Respuesta del ejercicio 10

#### Análisis del comportamiento del salto

El salto `bnez x1, loop` se ejecuta con x1 = 4, 3, 2, 1 (decrementado por `subi`).
- Iteraciones 1, 2, 3: x1 ≠ 0 → salto **Tomado (T)**.
- Iteración 4: x1 = 0 → salto **No Tomado (NT)**.

**Estado del BTB:** contador de 2 bits, parte en **00 = "no-salta-fuerte"** (predicción NT).

Transiciones del contador: 00→01 si T, 00→00 si NT (satura). 01→10 si T, 01→00 si NT. 10→11 si T, 10→01 si NT. 11→11 si T, 11→10 si NT.

Las dos instrucciones **siguientes en secuencia** (si no se salta) son `andi` y `ori`.  
Las dos instrucciones **en destino del salto** (si se toma) son `sub` y `add` (primeras instrucciones de `loop`).

#### Tabla de ejecución (IF, ID en ciclos siguientes al salto)

**Iteración 1** — Predicción: NT (00) | Real: T → **Fallo**. Nuevo estado: 01.
- Se han buscado `andi` y `ori` (predicción NT). Al detectar el fallo en ID (fin de decodificación, condición conocida), se cancelan.
- Las instrucciones correctas (`sub`, `add` de loop) se cargan desde el inicio.

| Instrucción     | Predicción | Real | IF | ID | EX | ME | WB |
|-----------------|-----------|------|----|----|----|----|-----|
| andi x6,x5,63   | (buscada) | —    | ✓  | X  | X  | X  | X   |
| ori x6,x6,128   | (buscada) | —    | ✓  | X  | X  | X  | X   |

*(X = cancelada por fallo de predicción)*

**Iteración 2** — Predicción: NT (01 → aún predice NT, bit alto = 0) | Real: T → **Fallo**. Nuevo estado: 10.
- Misma situación: se buscan `andi` y `ori`, se cancelan.

| Instrucción     | Predicción | Real | IF | ID | EX | ME | WB |
|-----------------|-----------|------|----|----|----|----|-----|
| andi x6,x5,63   | (buscada) | —    | ✓  | X  | X  | X  | X   |
| ori x6,x6,128   | (buscada) | —    | ✓  | X  | X  | X  | X   |

**Iteración 3** — Predicción: T (10 → bit alto = 1) | Real: T → **Acierto**. Nuevo estado: 11.
- Se buscan correctamente `sub` y `add` (destino del salto).

| Instrucción     | Predicción | Real | IF | ID | EX | ME | WB |
|-----------------|-----------|------|----|----|----|----|-----|
| sub x5,x5,x2    | (buscada) | —    | ✓  | ✓  | ✓  | ✓  | ✓   |
| add x2,x2,1     | (buscada) | —    | ✓  | ✓  | ✓  | ✓  | ✓   |

*(Sin cancelaciones — predicción correcta)*

**Iteración 4** — Predicción: T (11) | Real: NT → **Fallo**. Nuevo estado: 10.
- Se buscan `sub` y `add` (loop), pero el salto no se toma. Se cancelan.

| Instrucción     | Predicción | Real | IF | ID | EX | ME | WB |
|-----------------|-----------|------|----|----|----|----|-----|
| sub x5,x5,x2    | (buscada) | —    | ✓  | X  | X  | X  | X   |
| add x2,x2,1     | (buscada) | —    | ✓  | X  | X  | X  | X   |

**Resumen:** El BTB tarda 2 iteraciones en aprender que el salto se toma. Las iteraciones 1 y 2 producen 2 instrucciones canceladas cada una (penalización de 2 ciclos por fallo). La iteración 3 es correcta. La iteración 4 (último paso, salto no tomado) vuelve a fallar.

---

### 11. Enunciado del ejercicio 11

Se quiere diseñar un procesador con predicción dinámica de saltos mediante una tabla de historia de saltos. La mayoría de los programas ejecutan bucles anidados a tres niveles. ¿Cuántos fallos se producirán con las dos alternativas estudiadas en clase, usando 1 bit (2 estados) y 2 bits (4 estados), si en ambos casos se parte de no saltar?

```
for i = 1 ... n
  for j = 1 ... n
    for k = 1 ... n
      ....
    end   ← salto k
  end     ← salto j
end       ← salto i
```

---

### 11. Respuesta del ejercicio 11

#### Análisis por nivel de bucle

Cada instrucción de salto de fin de bucle (`end`) se toma durante n-1 iteraciones y no se toma en la última (para salir del bucle). Al volver a entrar en el bucle (inicio de una nueva iteración del bucle exterior), la primera ejecución del salto vuelve a tomarse.

El patrón de cada salto es, por tanto: **T, T, ..., T (n-1 veces), NT, T, T, ..., T (n-1 veces), NT, ...**

#### Predictor de 1 bit

El predictor cambia su estado ante cualquier error. El patrón de fallos para un bucle que itera n veces seguido de salida:
- **Al entrar** (primera T después de una NT): el predictor estaba en NT (0) → **1 fallo**.
- **n-1 iteraciones tomadas**: el predictor pasa a T (1) → 0 fallos.
- **Al salir** (NT): el predictor estaba en T (1) → **1 fallo**, pasa a NT.

Por tanto, cada "pasada completa" del bucle produce **2 fallos** (uno al entrar, uno al salir), excepto la primera pasada que solo tiene 1 fallo adicional de entrada.

**Cálculo para bucles anidados a 3 niveles:**

| Bucle | Iteraciones del salto | Fallos con 1 bit |
|-------|----------------------|-----------------|
| k (más interno) | n veces, repetido n² veces | 2 fallos × n² pasadas = **2n²** fallos |
| j (medio)       | n veces, repetido n veces  | 2 fallos × n pasadas = **2n** fallos   |
| i (externo)     | n veces, 1 vez              | 2 fallos × 1 pasada = **2** fallos     |

> **Total con 1 bit ≈ 2n² + 2n + 2 fallos**

#### Predictor de 2 bits

Con el contador saturante de 2 bits, se necesitan 2 fallos consecutivos para cambiar la predicción. El patrón de fallos:
- **Al entrar** (primera T después de una NT): predictor en "NT débil" (01) → 1 fallo, pasa a "T débil" (10). Si n ≥ 2, la siguiente T → acierto (10 → 11).
- **n-1 iteraciones tomadas** (con el predictor en T fuerte desde la 2ª): 0 fallos.
- **Al salir** (NT): predictor en T fuerte (11) → 1 fallo, pasa a T débil (10). Si el bucle vuelve a ejecutarse, la primera T → acierto (10 → 11).

De nuevo: **2 fallos por pasada completa** (1 al entrar, 1 al salir), igual que con 1 bit para esta estructura de bucle.

| Bucle | Fallos con 2 bits |
|-------|------------------|
| k (más interno) | 2 × n² = **2n²** fallos |
| j (medio)       | 2 × n = **2n** fallos   |
| i (externo)     | 2 × 1 = **2** fallos    |

> **Total con 2 bits ≈ 2n² + 2n + 2 fallos**

#### Conclusión

Para bucles anidados con n > 2 iteraciones, **ambos predictores producen el mismo número de fallos** en esta estructura de bucle anidado. La ventaja del predictor de 2 bits se manifiesta en otros patrones (como saltos con comportamiento irregular o excepcional), donde su histéresis evita cambios de predicción por eventos aislados. Para bucles simples que siempre terminan con exactamente una NT al final de cada pasada, la diferencia entre 1 y 2 bits es mínima.

Si n = 1 (bucle de una sola iteración): con 1 bit, el predictor oscila en cada ejecución (todos fallos). Con 2 bits, tarda algo más en estabilizarse pero también falla frecuentemente. En este caso degenera y ambos rinden mal.
