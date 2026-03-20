# Ejercicios EC

## Mario González García

---

## Segmentación avanzada

---

### 1. DLX 5 etapas con FP ADD (lat=2), FP MUL (lat=5), Int ALU (lat=1)

**Características del procesador:**
- Leer y escribir registro en el mismo ciclo (write-then-read en WB)
- Cortocircuito (forwarding) habilitado
- Detección de riesgos LDE y paradas en etapa ID
- Riesgos estructurales de memoria: espera en última etapa de cada UF
- Riesgos EDE: se resuelven mediante inhibición de escritura de la instrucción anterior (`a`)
- FP ADD: latencia 2, segmentada | FP MUL: latencia 5, segmentada | Int ALU: latencia 1, no segmentada

**Código:**
```
I1: flw    f10, 0(x1)
I2: fmul.s f4,  f0, f10
I3: flw    f12, 0(x2)
I4: fadd.s f2,  f12, f4
I5: flw    f4,  8(x1)
I6: fmul.s f12, f4,  f12
I7: flw    f12, 16(x1)
```

---

#### a) Diagrama instrucción-tiempo y cortocircuitos

**Notación de etapas:** IF | ID | EX | MEM | WB  
Para FP MUL (lat=5, segmentada): EX1 | EX2 | EX3 | EX4 | EX5  
Para FP ADD (lat=2, segmentada): EX1 | EX2

```
Ciclo:   1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18

I1 flw   IF   ID   EX   MEM  WB
I2 fmul  .    IF   ID** ID** EX1  EX2  EX3  EX4  EX5  MEM  WB
I3 flw   .    .    IF   IF** IF** ID   EX   MEM  WB
I4 fadd  .    .    .    .    .    IF   ID** ID** EX1  EX2  MEM  WB
I5 flw   .    .    .    .    .    .    IF   IF** IF** ID   EX   MEM  WB
I6 fmul  .    .    .    .    .    .    .    .    .    IF   ID** ID** EX1  EX2  EX3  EX4  EX5  MEM  WB
I7 flw   .    .    .    .    .    .    .    .    .    .    IF   IF** IF** ID ...
```

> **Nota:** Los `**` indican ciclos de parada (stall). Las instrucciones posteriores se detienen en IF o ID hasta que se resuelve el riesgo.

**Análisis de riesgos y paradas:**

**I1 → I2 (RAW sobre f10):**
- I1 es `flw` (load): escribe f10 al final de MEM (ciclo 4) / WB (ciclo 5).
- I2 necesita f10 en EX1. Sin paradas, I2 entraría en EX en ciclo 4, pero f10 no está disponible hasta fin de ciclo 4 (MEM de I1).
- Con forwarding desde la salida de MEM de I1 hacia la entrada de EX1 de I2, se necesita que I1 esté en MEM cuando I2 entre en EX1.
- I2 en ID (ciclo 3) detecta el riesgo LDE (load-use): I1 aún no ha salido de EX. Se insertan **2 paradas** (ciclos 4 y 5 en ID), de modo que I2 entra en EX1 en ciclo 6 cuando I1 ya está en WB → forwarding WB→EX1.

> **Causa:** Riesgo LDE entre I1 (flw f10) e I2 (fmul usa f10). **2 paradas.**

**I2 → I4 (RAW sobre f4):**
- I2 (fmul, lat=5): EX1–EX5 en ciclos 6–10, MEM en ciclo 11, WB en ciclo 12 (con inhibición de escritura si hay EDE, sino WB normal).
- I4 necesita f4 (resultado de I2). I4 entra en ID en ciclo 16 (después de las paradas de I3).  
- I4 necesita f4 en EX1. I2 escribe f4 en WB (ciclo 12). Con forwarding, disponible tras ciclo 11 (MEM). I4 entraría en EX1 en ciclo 17 → forwarding desde WB de I2 (ciclo 12) a EX de I4 (ciclo... ver abajo).

**I3 → I4 (RAW sobre f12):**
- I3 es `flw f12` que entra en ID en ciclo 6 (después de las paradas de I2). EX en ciclo 7, MEM en ciclo 8, WB en ciclo 9.
- I4 necesita f12. I4 detecta riesgo LDE respecto a I3 → **2 paradas** en ID (ciclos 10–11), entra en EX1 en ciclo 12.
- En ese momento I2 tiene f4 disponible en MEM (ciclo 11) o WB (ciclo 12): forwarding posible ✓

> **Causa:** Riesgo LDE entre I3 (flw f12) e I4 (fadd usa f12). **2 paradas.**  
> **Cortocircuito:** Forwarding de f4 desde MEM/WB de I2 hacia EX de I4.

**I5 → I6 (RAW sobre f4):**
- I5 es `flw f4` que escribe en f4. I6 usa f4. Mismo patrón LDE → **2 paradas.**

**I6 → I7 (riesgo estructural sobre f12):**
- I6 escribe en f12, I7 también escribe en f12 (flw). Si ambas coinciden en WB → riesgo EDE.
- Se resuelve por inhibición de escritura de I6 (la instrucción anterior). I7 escribe el valor correcto.

**Resumen de cortocircuitos realizados:**
| Cortocircuito | Desde | Hacia | Registro |
|---|---|---|---|
| MEM→EX | I1 (MEM, c4) | I2 (EX1, c6) | f10 |
| MEM→EX | I3 (MEM, c8) | I4 (EX1, c12) | f12 |
| MEM→EX | I2 (MEM, c11) | I4 (EX1, c12) | f4 |
| MEM→EX | I5 (MEM, c…) | I6 (EX1, c…) | f4 |

---

#### b) Cálculo del CPI

Contamos las instrucciones y las paradas:

| Instrucción | Paradas asociadas |
|---|---|
| I1 flw f10 | 0 |
| I2 fmul (LDE con I1) | 2 paradas |
| I3 flw f12 | 0 |
| I4 fadd (LDE con I3; RAW con I2 resuelto por forwarding) | 2 paradas |
| I5 flw f4 | 0 |
| I6 fmul (LDE con I5) | 2 paradas |
| I7 flw f12 | 0 |

**Total instrucciones:** 7  
**Total paradas:** 2 + 2 + 2 = 6  
**Ciclos totales** = 7 + (5 - 1) etapas de pipeline inicial + 6 paradas = 7 + 4 + 6 = **17 ciclos** (aprox., contando desde que entra I1 hasta que sale I7 de WB)

**CPI = Ciclos / Instrucciones = (7 + 6) / 7 = 13/7 ≈ 1.86 ciclos/instrucción**

> El CPI exacto depende de la primera instrucción hasta la última en WB. Con 7 instrucciones, 6 stalls, y el pipeline de 5 etapas:  
> Ciclos = 5 + (7 - 1) + 6 = 5 + 6 + 6 = **17 ciclos**  
> **CPI = 17/7 ≈ 2.43** (contando ciclos de relleno del pipeline)

> Si calculamos CPI solo como penalizaciones por encima del ideal:  
> **CPI = 1 + (paradas / instrucciones) = 1 + 6/7 ≈ 1.86**

---

### 2. DLX 7 etapas con salto retardado y FP ADD (lat=4), FP MUL (lat=5), Int ALU (lat=1)

**Características:**
- Memoria de datos e instrucciones segmentadas en 2 etapas → pipeline de 7 etapas: IF1, IF2, ID, EX, MEM1, MEM2, WB
- Write-then-read en el mismo ciclo
- Cortocircuito habilitado
- Detección de riesgos (estructurales y LDE) en etapa ID
- EDE: inhibición de escritura
- Saltos resueltos en ID → **branch delay slot de 1 instrucción**
- Aritmética y store pueden coexistir en MEM y WB
- FP ADD: lat=4, segmentada | FP MUL: lat=5, segmentada | Int ALU: lat=1, no segmentada

**Código del bucle:**
```
loop: flw    f2,  0(x1)       ; I1
      fmul.s f4,  f2, f0      ; I2
      flw    f6,  0(x2)       ; I3
      fadd.s f6,  f4, f6      ; I4
      fsw    f6,  0(x2)       ; I5
      addi   x1,  x1, 8       ; I6
      slti   x3,  x1, done    ; I7
      bnez   x3,  loop        ; I8
      addi   x2,  x2, 8       ; I9 (delay slot)
```

**Datos:** `done = 0x1008`, `x1 = 0x0100 = 256` al inicio.

---

#### a) Cálculo del CPI

**Número de iteraciones:**  
El bucle continúa mientras `x1 < done = 0x1008 = 4104`.  
Cada iteración incrementa x1 en 8.  
Número de iteraciones = (4104 - 256) / 8 = 3848 / 8 = **481 iteraciones**

**Análisis de riesgos por iteración:**

Pipeline de 7 etapas: IF1, IF2, ID, EX[1..n], MEM1, MEM2, WB

Para FP MUL (lat=5): EX1–EX5 | Para FP ADD (lat=4): EX1–EX4

**I1 → I2 (RAW f2, load-use):**
- flw tiene etapas: IF1, IF2, ID, EX, MEM1, MEM2, WB. Escribe f2 tras MEM2.
- fmul necesita f2 en EX1. Sin paradas, entraría en EX demasiado pronto.
- Forwarding MEM2→EX1: I2 debe entrar en EX1 cuando I1 está en MEM2.
- I1 en MEM1 en c6, MEM2 en c7. I2 sin paradas entraría en EX en c5.
- Se necesitan **2 paradas** para retrasar I2 (entra en EX1 en c7).

**I2 → I4 (RAW f4, fmul→fadd):**
- fmul (lat=5): EX1–EX5 en ciclos 7–11, MEM1 c12, MEM2 c13, WB c14.
- fadd necesita f4 en EX1. Con forwarding desde MEM1 de fmul.

**I3 → I4 (RAW f6, flw→fadd):**
- I3 (flw) entra después de las paradas. Si hay riesgo LDE con I4, más paradas.
- I4 (fadd, lat=4): debe esperar tanto a f4 (de fmul) como a f6 (de flw I3).
- El cuello de botella es f4 de fmul.s (más lento). Paradas hasta que fmul haya completado suficientes etapas EX para forwarding.
- fmul termina EX5 en ciclo 11 → forwarding EX5→EX1 de fadd → fadd puede entrar en EX1 en ciclo 12.
- I3 termina MEM2 en el ciclo suficiente para forwarding a fadd en EX1 c12 → si I3 entra en MEM2 antes de c12, se puede hacer forwarding ✓.
- Paradas de I4 respecto a I2: **múltiples paradas** hasta que fmul complete sus etapas EX.

**I4 → I5 (RAW f6, fadd→fsw):**
- fadd (lat=4): si entra en EX1 en ciclo c, termina EX4 en c+3. fsw necesita f6 en MEM1.
- Con forwarding y la característica de que aritmética y store pueden coexistir en MEM/WB → posiblemente sin paradas adicionales.

**I6 → I7 (RAW x1, addi→slti):**
- addi (Int ALU, lat=1, no segmentada): EX en un ciclo, resultado en MEM1.
- slti necesita x1. Con forwarding MEM→EX de slti → posiblemente **1 parada** si no llega a tiempo.

**I7 → I8 (RAW x3, slti→bnez):**
- Salto resuelto en ID. bnez necesita x3 para evaluarlo en ID, pero slti escribe x3 tras EX. Riesgo de datos de control → **paradas** hasta que x3 esté disponible.

**Delay slot:** La instrucción I9 (`addi x2, x2, 8`) en el delay slot se ejecuta siempre, tanto si el salto se toma como si no. Esto aprovecha el hueco del salto.

**Estimación de paradas por iteración:**

| Riesgo | Paradas |
|---|---|
| I1→I2 (load-use f2) | 2 |
| I2→I4 (fmul→fadd, f4) | ~4 (esperar EX de fmul) |
| I3→I4 (flw→fadd, f6) | resuelto por el retraso de I2→I4 |
| I7→I8 (slti→bnez, x3) | 1–2 |
| I8 (branch, delay slot cubierto) | 0 (delay slot = I9) |

**Total paradas por iteración ≈ 8**  
**Instrucciones por iteración = 9**  
**Ciclos por iteración ≈ 9 + 8 = 17**  
**CPI ≈ 17/9 ≈ 1.89**

---

#### b) Reordenación del código para CPI mínimo

El objetivo es rellenar los huecos de parada con instrucciones útiles:

```assembly
loop:
      flw    f2,  0(x1)        ; I1: carga f2
      flw    f6,  0(x2)        ; I2: movida aquí (era I3) — rellena stall de I1→fmul
      addi   x1,  x1, 8        ; I3: movida aquí (era addi x1) — rellena otro hueco
      fmul.s f4,  f2, f0       ; I4: ahora f2 lleva 2 ciclos desde flw → menos stalls
      slti   x3,  x1, done     ; I5: usa el x1 ya actualizado
      fadd.s f6,  f4, f6       ; I6: f4 disponible tras fmul; f6 de I2
      bnez   x3,  loop         ; I7: salto — x3 disponible de slti
      fsw    f6,  0(x2)        ; I8: delay slot — f6 de fadd; escribe en 0(x2) viejo
      addi   x2,  x2, 8        ; I9: actualiza x2 (ya se usó la dirección en fsw)
```

> **Nota:** El `fsw` en el delay slot usa la dirección `0(x2)` antes de actualizar x2, lo cual es correcto porque queremos escribir en la posición original. Luego `addi x2` actualiza el puntero para la siguiente iteración.

**Beneficios de la reordenación:**
- Se eliminan las 2 paradas de LDE entre `flw f2` y `fmul` al interponer dos instrucciones independientes.
- La instrucción `slti` permite que x3 esté listo antes del `bnez`.
- El delay slot es ocupado por el `fsw`, eliminando un ciclo de penalización del salto.
- Las paradas de `fmul→fadd` se reducen al solapar la ejecución con instrucciones enteras.

**CPI estimado tras reordenación ≈ 1.1–1.3** (dependiendo de la latencia exacta fmul→fadd con forwarding)

---

### 3. FIR Filter — DLX con FP ADD/SUB (lat=2), FP MUL (lat=3), Int ALU (lat=1)

**Características:**
- x1=0, x3=&filter[0], x4=&input[0], x5=&out[0]; x10=4, x11=100
- Write-then-read en mismo ciclo
- Forwarding habilitado
- Detección de riesgos en ID
- EDE: paradas hasta que la instrucción lanzada entre en etapa MEM
- Dos instrucciones no pueden estar simultáneamente en MEM ni en WB (riesgo estructural)
- Saltos resueltos en ID, **sin delay slot** (se cancela la siguiente instrucción si el salto se toma)
- FP ADD/SUB: lat=2, segmentada | FP MUL: lat=3, segmentada | Int ALU: lat=1, no segmentada

**Código del bucle interno:**
```
loop_k:
  I1:  flw    f3,  0(x3)
  I2:  flw    f4,  0(x4)
  I3:  fmul.s f6,  f3, f4
  I4:  flw    f5,  0(x5)
  I5:  fadd.s f5,  f6, f5
  I6:  fsw    f5,  0(x5)
  I7:  addi   x3,  x3, 4
  I8:  addi   x4,  x4, 4
  I9:  addi   x2,  x2, 1
  I10: blt    x2,  x10, loop_k
  I11: addi   x5,  x5, 4
  I12: subi   x3,  x3, 16
  I13: subi   x4,  x4, 12
  I14: addi   x1,  x1, 1
  I15: blt    x1,  x11, loop_n
```

---

#### a) Diagrama instrucción-tiempo: 25 ciclos con k=3, n=0

Empezamos con `k=3` (última iteración del bucle interno antes de blt no tomado).

Pipeline: IF | ID | EX[1..n] | MEM | WB  
FP MUL: EX1, EX2, EX3 | FP ADD: EX1, EX2

```
Ciclo:   1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18   19   20   21   22   23   24   25

I1  flw    IF   ID   EX   MEM  WB
I2  flw    .    IF   ID   EX   MEM  WB
I3  fmul   .    .    IF   ID** ID** EX1  EX2  EX3  MEM  WB
    [pausa: LDE I1→I3 (f3) y I2→I3 (f4) — 2 stalls en ID]
I4  flw    .    .    .    IF   IF** IF** ID   EX   MEM  WB
    [arrastrada por los stalls de I3]
I5  fadd   .    .    .    .    .    .    IF   ID** ID** EX1  EX2  MEM  WB
    [LDE: I3→I5 (f6): fmul termina EX3 c8, I5 debe entrar en EX1 tras c8 → entra c9 ok con forwarding]
    [LDE: I4→I5 (f5): flw termina MEM c9, I5 entra EX1 c9 → forwarding MEM→EX1 c9 ✓]
    [1 stall extra para esperar f6 de fmul]
I6  fsw    .    .    .    .    .    .    .    IF   IF** ID   EX   MEM  WB
    [LDE: I5→I6 (f5): fadd termina EX2 c11, fsw necesita f5 en MEM. Con forward EX2→MEM ok si no coinciden]
    [Riesgo estructural: I5 en MEM (c12) e I6 en MEM (c12) — conflicto → 1 stall en I6]
I6  fsw    .    .    .    .    .    .    .    .    .    .    IF   ID   EX  MEM  WB
I7  addi   .    .    .    .    .    .    .    .    IF   ID   EX   MEM  WB
I8  addi   .    .    .    .    .    .    .    .    .    IF   ID   EX   MEM  WB
I9  addi   .    .    .    .    .    .    .    .    .    .    IF   ID   EX   MEM  WB
I10 blt    .    .    .    .    .    .    .    .    .    .    .    IF   ID** EX?  ...
    [I9→I10 RAW x2: addi escribe x2 en WB c14, blt necesita x2 en ID c13 → 1 stall]
I10 blt    .    .    .    .    .    .    .    .    .    .    .    .    IF   ID   ...
```

> La rama `blt x2, x10, loop_k` con k=3 y x2 que ha llegado a 3 (o 4 dependiendo del inicio): si x2 < 4 → salta; en la **última iteración k=3** el salto **no se toma** (x2=4 no es < 4).

```
Ciclo:   1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18   19   20   21   22   23   24   25

I1  flw    IF   ID   EX   MEM  WB
I2  flw         IF   ID   EX   MEM  WB
I3  fmul             IF   ID   s    s    EX1  EX2  EX3  MEM  WB         [2 stalls]
I4  flw                   IF   s    s    ID   EX   MEM  WB
I5  fadd                            IF   IF   IF   ID   s    EX1  EX2  MEM  WB  [1 stall por f6]
I6  fsw                                            IF   IF   ID   s    EX   MEM  WB  [1 stall estructural]
I7  addi                                                      IF   ID   EX   MEM  WB
I8  addi                                                           IF   ID   EX   MEM  WB
I9  addi                                                                IF   ID   EX   MEM  WB
I10 blt                                                                      IF   ID   [resolución]
I11 addi                                                                          IF   ID  EX  MEM  WB
```

**Stalls en 25 ciclos: 2 (I3) + 1 (I5) + 1 (I6) + 1 (I10) = 5 stalls**

---

#### b) Estimación de ciclos totales y CPI

**Estructura del código:**
- Bucle externo: 100 iteraciones (n = 0..99)
- Bucle interno: 4 iteraciones (k = 0..3)
- Total iteraciones bucle interno: 100 × 4 = 400

**Ciclos por iteración del bucle interno (I1–I10):**
- Instrucciones: 10
- Stalls: 2 (LDE fmul) + 1 (LDE fadd espera fmul) + 1 (estructural fsw/fadd) + 1 (RAW blt/addi) = **5 stalls**
- Ciclos por iteración ≈ **15 ciclos**

**Iteraciones del bucle externo (I11–I15), ejecutadas 100 veces (una por cada n):**
- Instrucciones: 5 (I11–I15)
- `blt x1, x11, loop_n`: 1 stall por RAW x1 (addi→blt)
- Ciclos por ejecución del cuerpo externo ≈ **7 ciclos**

**Ciclos totales:**
```
Ciclos_interno = 400 × 15 = 6000
Ciclos_externo = 100 × 7  =  700
Pipeline drain (últimas etapas) ≈ 4

Total ≈ 6000 + 700 + 4 = 6704 ciclos
```

**Total instrucciones ejecutadas:**
```
Bucle interno: 400 × 10 = 4000
Bucle externo: 100 × 5  =  500
Total = 4500 instrucciones
```

**CPI = 6704 / 4500 ≈ 1.49 ciclos/instrucción**

---

#### c) Adaptación con delay slot (1 instrucción)

Con delay slot, la instrucción **después** del salto siempre se ejecuta. Hay que asegurarse de poner una instrucción útil o una NOP.

**Para `blt x2, x10, loop_k`:**
La instrucción siguiente (I11: `addi x5, x5, 4`) solo debe ejecutarse cuando el salto **no** se toma (última iteración). Si ponemos I11 en el delay slot, se ejecutaría también cuando el salto se toma → **incorrecto**.

→ Hay que poner una **NOP** en el delay slot (o una instrucción que sea segura en ambos casos, como `addi x3, x3, 4` o `addi x4, x4, 4` si todavía no se han actualizado).

**Para `blt x1, x11, loop_n`:**
La instrucción después es el inicio de loop_n o la que sigue a done. Podemos mover al delay slot una instrucción independiente.

**¿Se obtiene beneficio?**
En este código concreto, el delay slot de `blt loop_k` **no puede ser aprovechado** con una instrucción útil de forma segura (I11 no debe ejecutarse en cada iteración). Por tanto, se colocaría una NOP → **no hay beneficio neto** para el bucle interno.

Para el salto del bucle externo, podría aprovecharse con una instrucción de reset de puntero. Beneficio marginal.

**Conclusión:** El delay slot no proporciona beneficio significativo en este código tal como está escrito.

---

#### d) Mejora mediante reordenación e instrucciones

**Técnica: Loop unrolling × 4 (desenrollado del bucle interno)**

Al desenrollar el bucle interno 4 veces, eliminamos el overhead del bucle y podemos solapar mejor las latencias:

```assembly
; Bucle externo (sin bucle interno — desenrollado completo)
loop_n:
      flw    f3,  0(x3)          ; k=0: carga filter[0]
      flw    f4,  0(x4)          ; k=0: carga input[n+0]
      flw    f7,  4(x3)          ; k=1: carga filter[1]   ← adelantada
      fmul.s f6,  f3, f4         ; k=0: filter[0]*input[n]
      flw    f8,  4(x4)          ; k=1: carga input[n+1]  ← rellena stall de fmul k=0
      flw    f9,  8(x3)          ; k=2: carga filter[2]
      fmul.s f10, f7, f8         ; k=1: filter[1]*input[n+1]
      flw    f11, 8(x4)          ; k=2: carga input[n+2]
      flw    f12, 12(x3)         ; k=3: carga filter[3]
      fmul.s f13, f9, f11        ; k=2
      flw    f14, 12(x4)         ; k=3: carga input[n+3]
      flw    f5,  0(x5)          ; carga out[n]
      fmul.s f15, f12, f14       ; k=3
      fadd.s f6,  f6,  f10       ; suma parcial k=0+k=1
      fadd.s f13, f13, f15       ; suma parcial k=2+k=3
      fadd.s f6,  f6,  f13       ; suma total
      fadd.s f5,  f5,  f6        ; out[n] += suma
      fsw    f5,  0(x5)          ; almacena out[n]
      addi   x3,  x3,  16        ; avanza filter (4 elementos × 4 bytes)
      subi   x3,  x3,  16        ; vuelve a filter[0]  ← equivalente a reset
      addi   x4,  x4,  4         ; avanza input[n] → input[n+1]
      addi   x5,  x5,  4         ; avanza out
      addi   x1,  x1,  1
      blt    x1,  x11, loop_n
```

> **Nota:** Con desenrollado ×4, el bucle interno desaparece. Se eliminan: 4 saltos del bucle interno, 4 instrucciones `addi x2`, y `blt` redundantes. Las 4 multiplicaciones se solapan con las cargas → reducción significativa de stalls.

**Estimación de ciclos con desenrollado ×4:**
- Instrucciones por iteración (bucle externo): ~24 instrucciones
- Stalls estimados: ~6–8 (los stalls de fmul→fadd se solapan con cargas inter-multiplicaciones)
- Ciclos por iteración: ~30–32
- Total: 100 × 31 = **3100 ciclos**
- Total instrucciones: 100 × 24 = **2400**
- **CPI ≈ 3100 / 2400 ≈ 1.29** (mejora notable respecto a ~1.49 sin desenrollar)

---

## Problemas Opcionales

---

### Opcional 1. DLX con FP ADD/SUB (lat=2, no seg.), FP MUL (lat=3, no seg.), FP DIV (lat=5, no seg.), Int ALU (lat=1)

**Características:**
- Write-then-read en mismo ciclo
- Forwarding habilitado
- Saltos en ID → se cancela siguiente instrucción si salto tomado
- Detección de riesgos en ID
- EDE: parada hasta que instrucción lanzada entra en MEM
- x1=4, x2=4 inicialmente
- Dos instrucciones no pueden coincidir en MEM ni WB
- Todas las UF FP: **no segmentadas**

**Código:**
```
L0:
  I1:  fadd.s  f2,  f4, f0
  I2:  fsw     f2,  0(x1)
  I3:  fdiv.s  f4,  f4, f0
  I4:  fadd.s  f4,  f0, f2
  I5:  fmul.s  f2,  f2, f4
  I6:  flw     f4,  0(x1)
  I7:  fadd.s  f0,  f2, f4
  I8:  fadd.s  f8,  f6, f8
  I9:  subi    x2,  x2, 1
  I10: addi    x1,  x1, 1
  I11: bnez    x2,  L0
  I12: fadd.s  f0,  f8, f2
  I13: fadd.s  f2,  f0, f8
```

**UF no segmentadas → ocupación exclusiva durante toda la latencia (estructural).**

---

#### a) Diagrama instrucción-tiempo y cortocircuitos

Dado que las UF FP no están segmentadas, una nueva instrucción FP no puede usar la misma UF hasta que la anterior termine completamente.

```
Etapas base: IF | ID | EX | MEM | WB
FP ADD (lat=2, no seg): EX ocupa 2 ciclos  → MEM en ciclo IF+3+2=...
FP MUL (lat=3, no seg): EX ocupa 3 ciclos
FP DIV (lat=5, no seg): EX ocupa 5 ciclos
```

**Análisis de riesgos:**

**I1 (fadd f2) → I2 (fsw f2): RAW f2**
- fadd no segmentada (lat=2): entra EX en c3, termina EX en c4, MEM en c5, WB en c6.
- fsw necesita f2 en MEM. Sin paradas, fsw entraría en MEM en c5 (misma que I1 → riesgo estructural). + RAW f2.
- Forwarding EX→MEM no aplica si no seg. Se necesita que I1 haya terminado EX antes de que I2 use f2.
- EDE: parada en I2 hasta que I1 entre en MEM → I2 en ID espera hasta c4 → I2 entra MEM en c6.
- **1–2 paradas** en I2.

**I1 (fadd f2) → I4 (fadd f4←f2): RAW f2, además I3 ocupa la UF fadd**
- I3 (fdiv f4, lat=5, no seg): entra EX en ciclo posterior a I2. UF fdiv no impide fadd.
- I4 (fadd): necesita f2 (de I1) y f4 (de I3). El cuello de botella es f4 de fdiv.
- fdiv (lat=5): si entra EX en c5, termina EX en c9. I4 no puede entrar hasta c10 (EDE: espera a que fdiv entre en MEM en c10).
- **Muchas paradas en I4** (esperando fdiv).

**I4 → I5 (RAW f4 y f2→fmul): EDE**
- Además, I3 (fdiv) aún puede estar en EX cuando I5 intenta entrar.
- fmul (lat=3, no seg): también riesgo estructural si otra instrucción MUL en vuelo (no hay otra aquí).
- Paradas de I5 hasta que f4 (de I4/fadd) y f2 estén disponibles.

**I5 → I7 (RAW f2→fadd f0): y I6 (flw f4) → I7 (RAW f4)**
- fmul (lat=3): si entra EX en c_x, termina en c_x+2.
- flw: escribe f4 tras MEM.
- I7 espera al más lento: fmul o flw.

**I8 (fadd f8): riesgo estructural con UF fadd si otra fadd en vuelo**
- Si I7 (fadd) no ha liberado la UF fadd, I8 debe esperar.
- **1–2 paradas** estructurales.

**I9, I10 (enteras, lat=1):** sin conflictos significativos entre sí.

**I11 (bnez x2):**
- x2 actualizado por I9 (subi). Si hay RAW → parada.
- Salto resuelto en ID: si tomado, se cancela I12 (burbuja).

**En la primera iteración, x2=4 → subi → x2=3 ≠ 0 → salto tomado → I12 cancelada (burbuja).**

**Cortocircuitos realizados:**
| Cortocircuito | Desde | Hacia | Registro |
|---|---|---|---|
| MEM→EX | I1 (fadd, MEM) | I2 (fsw, EX/MEM) | f2 |
| MEM→EX | I1 (fadd) | I4 (fadd) | f2 |
| MEM→EX | I3 (fdiv, MEM) | I4 (fadd) | f4 |
| MEM→EX | I4 (fadd) | I5 (fmul) | f4 |
| MEM→EX | I5 (fmul) | I7 (fadd) | f2 |
| MEM→EX | I6 (flw) | I7 (fadd) | f4 |

---

#### b) CPI de la ejecución completa

Con x2=4 inicialmente y subi x2, x2, 1 en cada iteración → 4 iteraciones del bucle (x2=4,3,2,1 → cuando x2=0 no salta).

**Stalls estimados por iteración (primera iteración como referencia):**

| Riesgo | Stalls |
|---|---|
| I1→I2 (EDE fadd→fsw) | 1 |
| I2→I3 estructural (MEM) | 1 |
| I3→I4 (EDE fdiv→fadd, esperando fdiv) | ~4 |
| I4→I5 (EDE fadd→fmul) | 1 |
| I5→I6 estructural (MEM) | posible 1 |
| I5→I7 (EDE fmul→fadd, f2) | ~2 |
| I6→I7 (LDE flw→fadd, f4) | resuelto por espera de I5 |
| I7→I8 estructural (UF fadd) | 1 |
| I9→I11 (RAW x2 subi→bnez) | 1 |
| I11 (salto tomado, burbuja) | 1 |

**Total stalls por iteración ≈ 13**  
**Instrucciones por iteración = 11** (I1–I11; I12–I13 solo en la última)  
**Ciclos por iteración ≈ 24**

**Iteraciones del bucle:** 4  
**Ciclos bucle:** 4 × 24 = 96  
**Instrucciones finales (I12, I13):** ~4 ciclos adicionales con 1 RAW  
**Total ≈ 100 ciclos**

**Total instrucciones:** 4 × 11 + 2 = 46  
**CPI = 100 / 46 ≈ 2.17 ciclos/instrucción**

---

### Opcional 2. DLX con FP ADD/SUB (lat=3, seg.), FP MUL (lat=4, seg.), FP DIV (lat=5, no seg.), Int ALU (lat=1)

**Código (x3=3 inicialmente, bucle 3 iteraciones):**
```
      addi x3, x0, 3
L1:
  I1:  fsub.s  f2,  f6, f8
  I2:  fsub.s  f4,  f8, f6
  I3:  fsw     f4,  0(x3)
  I4:  fdiv.s  f2,  f4, f8
  I5:  fadd.s  f2,  f8, f8
  I6:  subi    x3,  x3, 1
  I7:  fdiv.s  f6,  f4, f8
  I8:  fmul.s  f4,  f2, f6
  I9:  fsub.s  f10, f2, f6
  I10: flw     f4,  0(x3)
  I11: fadd.s  f0,  f4, f2
  I12: bnez    x3,  L1
  I13: fmul.s  f4,  f2, f2   ; fuera del bucle
```

---

#### a) Diagrama instrucción-tiempo (primera iteración)

**Análisis de riesgos clave:**

**I1→I2 (estructural UF fsub):**
- fsub (lat=3, seg): I1 en EX1–EX3. I2 puede entrar en EX1 en el siguiente ciclo (segmentada → no bloquea). Sin stall estructural.

**I2→I3 (RAW f4, fsub→fsw):**
- fsub (lat=3): I2 EX1–EX3 en c4–c6, MEM c7. fsw necesita f4 en MEM.
- Con forwarding EX3→MEM de fsw → si I3 entra en MEM en c7 y I2 termina EX3 en c6: forwarding posible. ✓

**I2→I4 (RAW f4, fsub→fdiv):**
- fdiv necesita f4. fsub termina EX3 → forwarding al inicio de EX de fdiv.
- Sin stall si forwarding disponible. ✓

**I4 vs I7 (ambos fdiv, riesgo estructural):**
- fdiv no segmentada (lat=5): I4 ocupa la UF fdiv durante 5 ciclos.
- I7 no puede entrar en fdiv hasta que I4 la libere → **stall estructural** hasta que I4 termine EX5.

**I4→I5 (RAW f2, fdiv→fadd):**
- fdiv (lat=5): I4 termina EX en c(4+4)+5-1=... (según cuándo entre).
- I5 (fadd, lat=3, seg): necesita f2 de I4. EDE: parada hasta que I4 entre en MEM.
- **Muchas paradas** (fdiv lenta).

**I5→I8 (RAW f2, fadd→fmul):**
- I8 también necesita f2. Espera a que fadd termine.

**I7→I8 (RAW f6, fdiv→fmul):**
- fmul necesita f6 de I7. Si I7 no ha terminado, más stalls.

**I10→I11 (LDE flw→fadd):**
- flw escribe f4, fadd necesita f4. Riesgo LDE → paradas.

**I12 (bnez x3):**
- x3 actualizado por I6 (subi). Con forwarding o 1 stall según latencia.

**Stalls estimados primera iteración:**

| Riesgo | Stalls |
|---|---|
| I4→I5 (EDE fdiv→fadd, f2) | ~4 |
| I4 vs I7 (estructural fdiv) | ~4 |
| I7→I8 (EDE fdiv→fmul, f6) | ~2 |
| I5→I9 (RAW fadd→fsub, f2) | 0 (solapado) |
| I10→I11 (LDE flw→fadd, f4) | 2 |
| I12 RAW x3 | 1 |

**Total stalls primera iteración ≈ 13**

---

#### b) Ciclos totales de ejecución completa

**Iteraciones del bucle:** x3 = 3, 2, 1 → 3 iteraciones (bnez toma el salto con x3≠0; cuando x3=0 no salta).

**Ciclos por iteración:** 12 instrucciones + ~13 stalls = **~25 ciclos/iteración**

- Los stalls de fdiv (lat=5, no seg) dominan → similar en todas las iteraciones.
- **3 iteraciones × 25 ciclos = 75 ciclos del bucle**

**Instrucción final (I13, fmul fuera del bucle):**
- Stalls por RAW f2 (de I11 fadd de la última iteración): ~2 stalls
- ~5 ciclos adicionales

**Total ≈ 75 + 5 + 4 (drain) = ~84 ciclos**

**Total instrucciones:** 3 × 12 + 1 = **37 instrucciones**  
**CPI = 84 / 37 ≈ 2.27**

---

### Opcional 3. DLX con delay slot, FP ADD (lat=4, no seg.), FP MUL (lat=5, no seg.), Int ALU (lat=1)

**Código:**
```
loop:
  I1: flw    f2,  0(x1)
  I2: fmul.s f4,  f2, f0
  I3: flw    f6,  0(x2)
  I4: fadd.s f6,  f4, f6
  I5: fsw    f6,  0(x2)
  I6: addi   x1,  x1, 8
  I7: addi   x2,  x2, 8
  I8: slti   x3,  x1, done
  I9: bnez   x3,  loop
done: add x1, x4, 5   ; delay slot
```

**Datos:** done=0x1008, x1=0x0100=256 al inicio.  
Coexistencia de store y aritmética en etapas MEM y WB permitida.  
UF no segmentadas → exclusivas durante su latencia.  
Delay slot de 1 instrucción.

---

#### a) Diagrama primera iteración y cortocircuitos

**Riesgos principales:**

**I1→I2 (LDE flw→fmul, f2):**
- flw termina MEM → forwarding a EX de fmul. Pero fmul necesita f2 al inicio de EX.
- Riesgo LDE → **2 paradas** (fmul espera que flw complete MEM para forwarding).

**I2→I4 (RAW fmul→fadd, f4):**
- fmul (lat=5, no seg): ocupa EX durante 5 ciclos.
- fadd necesita f4. EDE: espera hasta que fmul entre en MEM.
- **~4 paradas** esperando fmul.

**I3→I4 (LDE flw→fadd, f6):**
- Si I3 termina antes que fmul, la espera por f4 (de fmul) cubre también la espera por f6. Sin stalls adicionales.

**I4→I5 (RAW fadd→fsw, f6):**
- fadd (lat=4, no seg): ocupa EX 4 ciclos. fsw necesita f6.
- EDE + coexistencia MEM/WB permitida → forwarding posible desde MEM de fadd a MEM de fsw.
- **~3 paradas** esperando fadd.

**I8→I9 (RAW slti→bnez, x3):**
- slti (Int ALU, lat=1): resultado disponible tras EX → forwarding.
- **1 stall** si no llega a tiempo.

**Delay slot:** La instrucción en el delay slot (`add x1, x4, 5`) se ejecuta siempre. En la última iteración (cuando bnez no toma el salto), esta instrucción ejecuta y sobrescribe x1. **Precaución: esto puede ser un bug en el código original si se diseñó sin el delay slot en mente.**

**Cortocircuitos:**
| Cortocircuito | Desde | Hacia | Registro |
|---|---|---|---|
| MEM→EX | I1 (flw, MEM) | I2 (fmul, EX1) | f2 |
| MEM→EX | I2 (fmul, MEM) | I4 (fadd, EX) | f4 |
| MEM→EX | I3 (flw, MEM) | I4 (fadd, EX) | f6 |
| MEM→MEM | I4 (fadd, MEM) | I5 (fsw, MEM) | f6 |

---

#### b) CPI con done=0x1008, x1=0x0100

**Número de iteraciones:**  
x1 avanza de 256 en pasos de 8. Condición: x1 < done=4104.  
Iteraciones = (4104 - 256) / 8 = **481 iteraciones**

**Stalls por iteración:**
| Riesgo | Stalls |
|---|---|
| I1→I2 LDE (f2) | 2 |
| I2→I4 EDE fmul (f4) | 4 |
| I4→I5 EDE fadd (f6) | 3 |
| I8→I9 RAW (x3) | 1 |
| I9 bnez (delay slot, tomado) | 0 (cubierto por delay slot) |

**Total stalls por iteración = 10**  
**Instrucciones por iteración = 9** (I1–I9, con I10=`add x1,x4,5` en delay slot ejecutada siempre)  
**Contando delay slot: 10 instrucciones por iteración**  
**Ciclos por iteración = 10 + 10 = 20**

**Ciclos totales = 481 × 20 + 4 (pipeline drain) ≈ 9624 ciclos**  
**Total instrucciones = 481 × 10 = 4810**  
**CPI = 9624 / 4810 ≈ 2.00 ciclos/instrucción**

---

### Opcional 4. DLX con FP ADD/SUB (lat=2, seg.), FP MUL (lat=3, seg.), FP ADD no seg. (lat=4), Int ALU (lat=1)

> ⚠️ El enunciado presenta **dos UF FP ADD**: una segmentada (lat=2) y una no segmentada (lat=4). Esto es inusual — se interpreta como que existen dos unidades ADD distintas.

**Código (x5=1000 inicial):**
```
Loop:
  I1:  fdiv.s  f0,  f4, f2
  I2:  fadd.s  f0,  f2, f6     ; usa UF ADD (lat=2 seg. o lat=4 no seg.)
  I3:  fdiv.s  f8,  f8, f2
  I4:  addi    x3,  x3, 1
  I5:  fadd.s  f2,  f6, f8
  I6:  fmul.s  f6,  f8, f0
  I7:  flw     f2,  0(x3)
  I8:  fsw     0(x5), f6
  I9:  fmul.s  f2,  f6, f8
  I10: fadd.s  f6,  f8, f0
  I11: fadd.s  f0,  f2, f2
  I12: subi    x5,  x5, 1
  I13: bnez    x5,  Loop
  I14: fadd.s  f4,  f2, f2
  I15: fsub.s  f6,  f0, f0
end:
  I16: subi    x3,  x3, 1
```

**Riesgos principales en la primera iteración:**

**I1 vs I3 (estructural fdiv, no segmentada — si solo hay 1 UF fdiv):**
- Hay 1 UF fdiv. I3 debe esperar a que I1 libere fdiv → **stalls estructurales** (~5 ciclos).

**I1→I2 (RAW f0, fdiv→fadd):**
- fdiv (lat=5 según tabla del opt.1, pero aquí no se especifica fdiv — en este problema la tabla es: FP ADD seg lat=2, FP MUL seg lat=3, FP ADD no seg lat=4). 
- No hay FP DIV en la tabla de este problema. Se asume que fdiv usa la UF FP ADD no segmentada (lat=4) o es una UF separada implícita. Con lat=4 no seg:
  - I1 EX ciclos 3–6, MEM c7, WB c8. I2 necesita f0 → EDE → **~4 stalls**.

**I3 vs I1 (estructural si comparten UF):**
- Si fdiv es una UF separada con 1 instancia → I3 espera a que I1 libere la UF.

**I5→I6 (RAW f8, fadd→fmul):**
- fadd (lat=2, seg): I5 EX1–EX2, forwarding a fmul EX1. Sin stalls si forwarding llega a tiempo.

**I6→I8 (RAW f6, fmul→fsw):**
- fmul (lat=3, seg): forwarding EX3→MEM del fsw. Sin stalls si solapado correctamente.

**I6→I9 (RAW f6, fmul→fmul2):**
- I9 necesita f6 (de I6). Si I6 aún está en EX, EDE → paradas.

**I9→I11 (RAW f2, fmul→fadd):**
- I11 necesita f2 de I9 (fmul, lat=3). Forwarding posible si timing correcto.

**I13 (bnez x5):**
- x5 actualizado por I12 (subi, lat=1). Con forwarding, sin stalls significativos.
- x5=1000 → tomado 1000 veces.

**Stalls estimados por iteración:**

| Riesgo | Stalls |
|---|---|
| I1→I2 (EDE fdiv→fadd) | 3–4 |
| I1 vs I3 (estructural fdiv) | 4 |
| I3→I5 (EDE fdiv→fadd, f8) | 2 |
| I6→I9 (EDE fmul→fmul, f6) | 2 |
| I8/I9 estructural (MEM) | 1 |
| I13 (salto, x5≠0 → burbuja) | 1 |

**Total stalls/iteración ≈ 14–15**

**CPI en régimen estacionario:**  
- 13 instrucciones (I1–I13) + 14 stalls = **~27 ciclos/iteración**
- **CPI ≈ 27/13 ≈ 2.08**

---

### Opcional 5. DLX con FP ADD (lat=2, no seg.), FP MUL (lat=5, no seg.), FP DIV (lat=10, no seg.), Int ALU (lat=1); Delay slot de 1 instrucción

**Código:**
```
loop:
  I1:  flw     f6,  0(x2)
  I2:  fmul.s  f8,  f6, f0
  I3:  addi    x2,  x2, 1
  I4:  flw     f2,  0(x2)
  I5:  fdiv.s  f8,  f2, f8
  I6:  fsw     f8,  0(x2)
  I7:  fmul.s  f8,  f2, f0
  I8:  fadd.s  f8,  f4, f6
  I9:  addi    x3,  x3, 8
  I10: slti    x4,  x3, done
  I11: beqz    x4,  loop      ; salta si x4=0 (x3 >= done), es decir, NO salta si x3 < done
  I12: (delay slot)
done: add x1, x2, 5
```

> **Nota:** `beqz x4, loop` salta a loop si x4=0, lo que ocurre cuando slti devuelve 0, es decir cuando x3 ≥ done → salta al loop cuando la condición de fin NO se cumple. El bucle continúa mientras x3 < done.

---

#### a) CPI suponiendo que el bucle se ejecuta muchas veces

**Análisis de riesgos:**

**I1→I2 (LDE flw→fmul, f6):**
- flw escribe f6 tras MEM. fmul necesita f6. EDE con UF no seg → **2 paradas** (fmul en ID espera a que flw entre en MEM).

**I2→I5 (RAW fmul→fdiv, f8):**
- fmul (lat=5, no seg): ocupa EX durante 5 ciclos. fdiv necesita f8.
- EDE: fdiv espera en ID hasta que fmul entre en MEM → **~4 paradas** mínimo.
- Además, I4 (flw) también escribe en f2 que usa fdiv → si flw termina después de fmul, más espera.

**I4→I5 (LDE flw→fdiv, f2):**
- Si I4 en MEM y I5 en EX simultáneamente con forwarding → 1–2 paradas adicionales.

**I5→I6 (RAW fdiv→fsw, f8):**
- fdiv (lat=10, no seg): ocupa EX 10 ciclos. fsw necesita f8 en MEM.
- EDE: fsw espera en ID hasta que fdiv entre en MEM → **~9 paradas** (cuello de botella principal).

**I5→I7 (RAW fdiv→fmul2, f8):**
- I7 también necesita f8. Debe esperar a que fdiv termine. **~9 paradas** (solapado con espera de I6).

**I7→I8 (RAW fmul→fadd, f8):**
- fmul (lat=5, no seg): I8 espera → **~4 paradas**.
- Pero si I8 ya está retrasada por la espera de I5 (fdiv), puede estar solapado.

**I10→I11 (RAW slti→beqz, x4):**
- slti (lat=1): forwarding → sin stalls o **1 stall**.

**I11 (beqz, delay slot):**
- Delay slot: instrucción siguiente a beqz siempre se ejecuta.
- Hay que colocar una instrucción útil o NOP en el delay slot.
- En la imagen original no se especifica qué va en el delay slot; se asume NOP si no se reordena → **0 penalización** (delay slot = 1 ciclo ya pagado por la instrucción del DS).

**Total stalls/iteración estimados:**

| Riesgo | Stalls |
|---|---|
| I1→I2 LDE (f6) | 2 |
| I2→I5 EDE fmul→fdiv (f8) | 4 |
| I4→I5 LDE (f2) | 0 (solapado) |
| I5→I6 EDE fdiv→fsw (f8) | 9 |
| I5→I7 EDE fdiv→fmul2 (f8) | 0 (solapado con I6) |
| I7→I8 EDE fmul→fadd (f8) | 4 |
| I10→I11 RAW (x4) | 1 |
| delay slot (NOP) | 0 |

**Total stalls/iteración ≈ 20**  
**Instrucciones por iteración = 11** (I1–I11 + 1 delay slot = 12 efectivos)

**Ciclos por iteración = 12 + 20 = 32**  
**CPI = 32/12 ≈ 2.67** (o 32/11 ≈ 2.91 sin contar el delay slot como instrucción útil)

---

#### b) Inhibición de escritura para reducir penalizaciones EDE

**Mecanismo:** Cuando hay un riesgo EDE entre instrucción A (que escribe registro R) e instrucción B (que lee R), en lugar de detener B en ID hasta que A entre en MEM, se deja que **A continue pero se inhibe su escritura** en el registro destino si B ya ha leído el valor correcto por forwarding desde una etapa anterior.

**Aplicación práctica:**

En el contexto de este procesador, cuando hay dos instrucciones A y B con riesgo EDE (A escribe f8 y B también escribe f8, o A escribe f8 y B lo lee muy pronto):

- **Sin inhibición:** B espera en ID hasta que A entre en MEM (stall largo).
- **Con inhibición de escritura:** si A es más reciente y "ganará" la escritura de todas formas, se puede dejar que A continue y que B continue también, inhibiendo la escritura de A (o de B) según cuál sea el valor correcto.

**En este código específico:**

El cuello de botella es `I5 (fdiv) → I6 (fsw)` y `I5 (fdiv) → I7 (fmul)`. Con inhibición:

- I7 (fmul escribe f8) comienza antes sin esperar a fdiv completamente; cuando fdiv termine, **se inhibe la escritura de fmul en f8** si fdiv es la instrucción "correcta" (la más reciente que escribe f8). Esto solo funciona si el resultado de fmul no se necesita después.
- Sin embargo, I8 (fadd) necesita f8 de I7 (fmul), no de fdiv. Por tanto **no podemos inhibir la escritura de fmul** sin perder el resultado necesario.

**Reducción de CPI con inhibición:**
- Los stalls de I5→I6 (9 stalls) se reducen: I6 puede lanzarse antes si se usa forwarding combinacional desde el registro de pipeline de fdiv.
- Potencial ahorro: ~4–5 stalls en los casos donde la instrucción posterior puede adelantarse.

**Nuevo CPI estimado con inhibición:**
- Stalls reducidos a ~12 (eliminando ~8 stalls de fdiv gracias a forwarding mejorado e inhibición)
- Ciclos/iteración ≈ 12 + 12 = 24
- **CPI ≈ 24/12 = 2.00**

> La reducción exacta depende de la implementación. La inhibición de escritura es especialmente útil cuando una instrucción lenta (fdiv, lat=10) produce un resultado que otra instrucción sobreescribirá inmediatamente, permitiendo lanzar las instrucciones posteriores sin esperar el resultado de fdiv cuando no lo necesitan.
