# Ejercicios EC
## Mario González García
---
## Planificación Dinámica

---

### 1. Estado de la unidad de ejecución con Tomasulo (primer ejercicio)

**Enunciado:** Secuencia de instrucciones FP en un DLX con Tomasulo. Latencia de todas las UF = 6 ciclos. Solo la primera instrucción (`fld f6, x(x1)`) ha completado su ejecución (f6 = x). El resto de registros fi tienen valor `[Fi]`. No se puede escribir en CDB e iniciar en el mismo ciclo.

#### Análisis de dependencias

```
I1: fld    f6,  x(x1)    → escribe f6 = x         ✓ COMPLETADA
I2: fld    f2,  y(x1)    → escribe f2 = [y]        en ejecución (MEM)
I3: fmul.d f0,  f2, f4   → espera f2 (de I2)
I4: fsub.d f8,  f6, f2   → espera f2 (de I2); f6 ya disponible (= x)
I5: fdiv.d f6,  f0, f6   → espera f0 (de I3) y f6 disponible (=x)
I6: fadd.d f10, f0, f6   → espera f0 (de I3) y f6 (de I5)
I7: fadd.d f6,  f8, f2   → espera f8 (de I4) y f2 (de I2)
I8: fsd    f6,  z(x1)    → espera f6 (de I7)
```

#### Estado de las Estaciones de Reserva (momento en que todas están lanzadas)

| ER  | Operación | Qj        | Vj   | Qk        | Vk    | Busy |
|-----|-----------|-----------|------|-----------|-------|------|
| LD1 | LOAD      | —         | y(x1)| —        | —     | Sí   |
| MUL1| FMUL      | I2(f2)    | —    | —         | [F4]  | Sí   |
| ADD1| FSUB      | I2(f2)    | —    | —         | x     | Sí   |
| DIV1| FDIV      | I3(f0)    | —    | —         | x     | Sí   |
| ADD2| FADD      | I3(f0)    | —    | I5(f6)    | —     | Sí   |
| ADD3| FADD      | I4(f8)    | —    | I2(f2)    | —     | Sí   |
| ST1 | STORE     | I7(f6)    | —    | —         | —     | Sí   |

#### Estado del banco de registros (Register Result Status)

| Registro | Qi (etiqueta productora) |
|----------|--------------------------|
| f0       | MUL1 (I3)                |
| f2       | LD1  (I2)                |
| f6       | ADD3 (I7)                |
| f8       | ADD1 (I4)                |
| f10      | ADD2 (I6)                |
| f4       | —  ([F4])                |

**Observación clave:** I1 ya escribió f6 = x en el CDB y lo lanzó al banco de registros. Las instrucciones I4 e I5 usan ese valor directamente (Vk = x). I6, I7 e I8 esperan resultados intermedios aún no disponibles.

---

### 2. Tomasulo para AXPY en DLX a 800 MHz

**Enunciado:** Bucle AXPY con pipeline IF/ID/Issue/EX/WB. Latencias: Load/Store = 2 ciclos, Suma/Resta = 7 ciclos, Multiplicación/División = 10 ciclos. Se **puede** escribir en CDB e iniciar en el mismo ciclo. Delay slot = 1 instrucción.

```
bucle: fld    f2, x(x1)
       fmul.d f4, f2, f0
       fld    f6, y(x1)
       fadd.d f6, f4, f6
       fsd    f6, y(x1)
       bnez   x1, bucle
       subi   x1, x1, 8      ← delay slot (se ejecuta siempre)
```

#### Dependencias del bucle

- `fmul` espera `f2` de `fld x(x1)` → RAW
- `fadd` espera `f4` de `fmul` → RAW
- `fadd` espera `f6` de `fld y(x1)` → RAW
- `fsd` espera `f6` de `fadd` → RAW
- Entre iteraciones: `fld f2, x(x1+8)` independiente; `fld f6, y(x1+8)` independiente

#### Diagrama instrucción/tiempo — 2 primeras iteraciones

> Notación: IF=F, ID=D, IS=Is, EX=EX(c1..cn), WB=W

| Instrucción           | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  |10  |11  |12  |13  |14  |15  |16  |17  |18  |19  |20  |21  |22  |
|-----------------------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| **IT.1**              |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |
| fld f2, x(x1)        | F  | D  | Is | E1 | E2 | W  |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |
| fmul.d f4, f2, f0    |    | F  | D  | Is | *  | Is | E1 |... |E10 | W  |    |    |    |    |    |    |    |    |    |    |    |    |
| fld f6, y(x1)        |    |    | F  | D  | Is | E1 | E2 | W  |    |    |    |    |    |    |    |    |    |    |    |    |    |    |
| fadd.d f6, f4, f6    |    |    |    | F  | D  | Is | *  | *  | *  | Is | E1 |... | E7 | W  |    |    |    |    |    |    |    |    |
| fsd f6, y(x1)        |    |    |    |    | F  | D  | Is | *  | *  | *  | *  | *  | *  | Is | E1 | E2 | W  |    |    |    |    |    |
| bnez x1, bucle       |    |    |    |    |    | F  | D  | Is | W  |    |    |    |    |    |    |    |    |    |    |    |    |    |
| subi x1,x1,8 (slot)  |    |    |    |    |    |    | F  | D  | Is | W  |    |    |    |    |    |    |    |    |    |    |    |    |
| **IT.2**              |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |
| fld f2, x(x1)        |    |    |    |    |    |    |    | F  | D  | Is | E1 | E2 | W  |    |    |    |    |    |    |    |    |    |
| fmul.d f4, f2, f0    |    |    |    |    |    |    |    |    | F  | D  | Is | *  | Is | E1 |... |E10 | W  |    |    |    |    |    |
| fld f6, y(x1)        |    |    |    |    |    |    |    |    |    | F  | D  | Is | E1 | E2 | W  |    |    |    |    |    |    |    |
| fadd.d f6, f4, f6    |    |    |    |    |    |    |    |    |    |    | F  | D  | Is | *  | *  | *  | Is | E1 |... | E7 | W  |    |
| fsd f6, y(x1)        |    |    |    |    |    |    |    |    |    |    |    | F  | D  | Is | *  | *  | *  | *  | *  | *  | Is | E1 |
| bnez x1, bucle       |    |    |    |    |    |    |    |    |    |    |    |    | F  | D  | Is | W  |    |    |    |    |    |    |
| subi x1,x1,8 (slot)  |    |    |    |    |    |    |    |    |    |    |    |    |    | F  | D  | Is | W  |    |    |    |    |    |

> `*` = stall en Issue esperando operando. `Is` tras stalls = ciclo real de Issue.

#### Análisis del cuello de botella

El cuello de botella es la cadena crítica:
```
fld f2 (2 ciclos) → fmul (10 ciclos) → fadd (7 ciclos) → fsd (2 ciclos)
```
Total de latencia crítica por iteración: **Issue + 2 + 10 + 7 + 2 = ~21 ciclos entre iteraciones**

Sin embargo, con Tomasulo el `fld f6, y(x1)` de la segunda iteración se solapa con la multiplicación de la primera. El inicio de una nueva iteración ocurre cada **~14 ciclos** (dominado por `fmul` + `fadd`).

#### Velocidad en MFLOPS

Instrucciones FP por iteración: **2** (`fmul.d` y `fadd.d`)

Ciclos por iteración ≈ 14 ciclos (la segunda iteración empieza en ciclo 8, y la nueva en ciclo ~22)

```
Ciclos/iteración ≈ 14
Tiempo/iteración = 14 / 800×10⁶ = 17.5 ns
MFLOPS = (2 ops FP) / (17.5×10⁻⁹ s) = 114.3 MFLOPS
```

**Velocidad sostenida ≈ 114 MFLOPS**

---

### 3. Programa Y = Y + X con Tomasulo a 1 GHz

**Enunciado:** Vectores X e Y, máquina a 1 GHz. UF: Load (2 ciclos, 3 buf), Add (3 ciclos, 2 ER), Store (2 ciclos, 3 buf). CPI entero = 1, delay slot = 1. No segmentadas.

```
loop: fld    f2, X(x1)
      fld    f4, Y(x1)
      fadd.d f4, f2, f4
      fsd    Y(x1), f4
      bnez   x1, loop
      sub    x1, x1, 8     ← delay slot
```

#### Dependencias

- `fadd` RAW: espera `f2` (de 1er fld, 2 ciclos) y `f4` (de 2do fld, 2 ciclos)
- `fsd` RAW: espera `f4` (de `fadd`, 3 ciclos)

#### Diagrama instrucción/tiempo (1ª iteración)

| Instrucción       | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 |
|-------------------|----|----|----|----|----|----|----|----|----|-----|
| fld f2, X(x1)     | Is | E1 | E2 | W  |    |    |    |    |    |    |
| fld f4, Y(x1)     | Is | E1 | E2 | W  |    |    |    |    |    |    |
| fadd.d f4,f2,f4   | Is | *  | *  | Is | E1 | E2 | E3 | W  |    |    |
| fsd Y(x1), f4     | Is | *  | *  | *  | *  | *  | *  | Is | E1 | E2 |
| bnez x1, loop     |    |    |    |    |    |    |    |    | Is | W  |
| sub x1,x1,8       |    |    |    |    |    |    |    |    |    | Is |

> Las dos `fld` se emiten en ciclo 1 (Issue simultáneo si hay ER disponibles). `fadd` espera hasta ciclo 4 (cuando ambas cargas han escrito en ciclo 3→ disponible en 4, dado que **no** se puede escribir e iniciar en el mismo ciclo según el enunciado — revisando: *el enunciado del problema 3 no especifica esto*, se asume que **sí** se puede en el mismo ciclo, igual que el problema 2).

**Corrigiendo** (mismo ciclo = sí):
- fld f2 y fld f4: Issue en ciclos 1 y 2, EX en 2-3 y 3-4, WB en ciclo 3 y 4
- fadd: Issue en ciclo 5 (espera WB de ambas fld → ciclo 4), EX ciclos 5-7, WB ciclo 8
- fsd: Issue en ciclo 5, espera f4 hasta WB en ciclo 8 → EX ciclos 9-10, WB ciclo 11

**Ciclos por iteración ≈ 11 ciclos**

Para la siguiente iteración, los `fld` de la 2ª iteración pueden emitirse en ciclo 3 y 4 (registros independientes o mismos). El cuello de botella es la cadena `fld → fadd → fsd` = 2+3+2 = 7 ciclos de latencia pura, más 1 ciclo por WB = 8 ciclos de latencia crítica.

Con solapamiento entre iteraciones:

**Ciclos/iteración sostenida ≈ 8 ciclos**

#### Tiempo de ejecución (vectores de tamaño n)

```
Instrucciones enteras por iteración: sub, bnez = 2 (CPI=1 cada una)
Ciclos enteros por iteración: 2
Ciclos FP por iteración (cuello de botella): 8
Ciclos/iteración = max(8, 2) = 8 ciclos (FP domina)

T = n × 8 / (1×10⁹) = 8n ns = 8n × 10⁻⁹ s
```

**T(n) = 8n / 10⁹ segundos = 8n ns**

#### Velocidad en MFLOPS

Operaciones FP por iteración: **1** (`fadd.d`)

```
MFLOPS = (1 op × 10⁶) / (8 × 10⁻⁹ s × 10⁶) = 1 / 8×10⁻³ = 125 MFLOPS
```

**Velocidad sostenida = 125 MFLOPS**

---

### 4. Estaciones de reserva necesarias y MFLOPS (1.2 GHz)

**Enunciado:** 1.2 GHz, Load/Store segmentados con 3 etapas, suma segmentada con 4 etapas. Delay slot = 1. x1 = 256.

```
# x1 = 256
loop: sub   x1, x1, 4
      fld   f0, y(x1)
      fld   f2, z(x1)
      fld   f4, t(x1)
      fadd.d f6, f4, f0
      fadd.d f8, f2, f0
      fadd.d f6, f6, f8
      bnez  x1, loop
      fsd   f6, x(x1)     ← delay slot
```

#### Dependencias del bucle

- `fadd f6, f4, f0`: RAW sobre f4 (fld t) y f0 (fld y) → espera ambas cargas
- `fadd f8, f2, f0`: RAW sobre f2 (fld z) y f0 (fld y) → espera ambas cargas
- `fadd f6, f6, f8`: RAW sobre f6 (1ª fadd) y f8 (2ª fadd) → espera ambas sumas
- `fsd f6, x(x1)`: RAW sobre f6 (3ª fadd) y x1 (sub) → fuera del bucle

#### Diagrama instrucción/tiempo (1 iteración para contar ER)

Las UF segmentadas permiten emitir una nueva operación cada ciclo. Las latencias son:
- Load: 3 ciclos (segmentado)
- Store: 3 ciclos (segmentado)
- fadd: 4 ciclos (segmentado)

| Instrucción       | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  |10  |11  |
|-------------------|----|----|----|----|----|----|----|----|----|----|----|
| sub x1,x1,4       | Is | EX | W  |    |    |    |    |    |    |    |    |
| fld f0, y(x1)     | Is | E1 | E2 | E3 | W  |    |    |    |    |    |    |
| fld f2, z(x1)     | Is | E1 | E2 | E3 | W  |    |    |    |    |    |    |
| fld f4, t(x1)     | Is | E1 | E2 | E3 | W  |    |    |    |    |    |    |
| fadd f6, f4, f0   | *  | *  | *  | Is | E1 | E2 | E3 | E4 | W  |    |    |
| fadd f8, f2, f0   | *  | *  | *  | Is | E1 | E2 | E3 | E4 | W  |    |    |
| fadd f6, f6, f8   | *  | *  | *  | *  | *  | *  | *  | *  | Is | E1 |... |
| bnez x1, loop     |    |    |    |    |    |    |    |    |    | Is | W  |
| fsd f6, x(x1)     |    |    |    |    |    |    |    |    |    | Is |... |

> Los 3 `fld` se emiten en el mismo ciclo (ciclo 1), pues hay 3 buffers de load. La 3ª `fadd` espera hasta que las dos anteriores escriban (ciclo 9).

#### Estaciones de reserva necesarias

Para emitir 1 instrucción por ciclo sin stalls estructurales:

- **Buffers de Load:** Se emiten 3 `fld` a la vez → necesitamos **3 buffers de load**
- **Estaciones de reserva de suma (fadd):** En el peor caso, las 3 `fadd` están simultáneamente en las ER (la 3ª se emite cuando las otras 2 aún no han escrito). Como la suma es segmentada, en la 2ª iteración podría haber solapamiento. Necesitamos **3 estaciones de reserva** para la unidad de suma.
- **Buffers de Store:** Solo hay 1 `fsd` activa a la vez fuera del bucle → **1 buffer de store** (pero dado el delay slot, se necesita 1)

**Respuesta:** 3 buffers de load, 3 estaciones de reserva de suma, 1 buffer de store.

#### MFLOPS

Operaciones FP por iteración: **3** (`fadd`, `fadd`, `fadd`)  
Número de iteraciones: x1 = 256 → iteraciones = 256/4 = **64**

Ciclos por iteración (cuello de botella):
- La cadena crítica: 3 fld (3 ciclos) + fadd6 (4 ciclos) + fadd8 (4 ciclos, paralela) + fadd_final (4 ciclos) = 3+4+4 = 11 ciclos
- Pero la 2ª iteración puede comenzar en cuanto sub e instrucciones enteras liberen el Issue → cada iteración tarda **~11 ciclos**

```
Tiempo total = 64 × 11 / (1.2×10⁹) = 704 / 1.2×10⁹ ≈ 586.7 ns
MFLOPS = (64 × 3 operaciones FP) / (586.7×10⁻⁹) = 192 / 586.7×10⁻⁹ ≈ 327 MFLOPS
```

**MFLOPS ≈ 327 MFLOPS**

---

### 5. Diagrama instrucción-tiempo con Tomasulo (div, mul, add)

**Enunciado:** UF FP: suma (2 ciclos, segmentada, 3 ER), mul/div (10/30 ciclos, segmentada, 2 ER). No se puede escribir e iniciar en el mismo ciclo.

```
I1: fdiv.s f10, f11, f5
I2: fadd.s f6,  f10, f1
I3: fmul.s f7,  f6,  f4
I4: fadd.s f1,  f13, f14
I5: fadd.s f4,  f1,  f17
I6: fadd.s f15, f16, f17
```

#### Dependencias

- I2 RAW← I1 (f10): I2 espera hasta que I1 termine (ciclo 1+30=31, WB=31 → I2 puede Issue en ciclo 32)
- I3 RAW← I2 (f6): I3 espera WB de I2
- I5 RAW← I4 (f1): I5 espera WB de I4
- I3 RAW← I5 (f4): si I5 termina después que I2, I3 también espera f4
- I4 usa f13, f14: sin dependencias → Issue inmediato (ciclo 2)
- I6 usa f16, f17: sin dependencias → Issue en ciclo 3 (o cuando haya ER libre)

#### Diagrama instrucción/tiempo

| Instrucción           | Ciclo Issue | Ciclos EX       | Ciclo WB |
|-----------------------|-------------|-----------------|----------|
| I1: fdiv f10,f11,f5   | 1           | 2 – 31          | 32       |
| I2: fadd f6, f10,f1   | 33 (*)      | 34 – 35         | 36       |
| I3: fmul f7, f6, f4   | 37 (**)     | 38 – 47         | 48       |
| I4: fadd f1, f13,f14  | 2           | 3 – 4           | 5        |
| I5: fadd f4, f1, f17  | 6 (***)     | 7 – 8           | 9        |
| I6: fadd f15,f16,f17  | 3           | 4 – 5           | 6        |

> (*) I2 no puede emitir hasta ciclo 33 porque I1 escribe en ciclo 32 y no se puede iniciar en el mismo ciclo que WB.  
> (**) I3 depende de f6 (WB=36 → Issue≥37) **y** de f4 (WB de I5=9 → ya disponible). El cuello de I3 es f6.  
> (***) I5 depende de f1 (WB de I4=5 → Issue≥6).

#### Tabla final

| Instrucción           | Issue | EX (inicio-fin) | WB |
|-----------------------|-------|------------------|----|
| fdiv.s f10, f11, f5   | 1     | 2 – 31           | 32 |
| fadd.s f6, f10, f1    | 33    | 34 – 35          | 36 |
| fmul.s f7, f6, f4     | 37    | 38 – 47          | 48 |
| fadd.s f1, f13, f14   | 2     | 3 – 4            | 5  |
| fadd.s f4, f1, f17    | 6     | 7 – 8            | 9  |
| fadd.s f15, f16, f17  | 3     | 4 – 5            | 6  |

---

### 6. Tiempo de ejecución de T = a + Y + Z en DLX a 100 MHz

**Enunciado:** DLX a 100 MHz. UF no segmentadas: Load (4 ciclos, 2 buf), Suma (3 ciclos, 3 ER), Store (4 ciclos, 2 buf). CPI entero = 1, delay slot = 1. f0 = a (escalar).

```
# f0 contiene a
loop: fld    f4, y(x1)
      fld    f2, z(x1)
      fadd.d f4, f0, f4    ← a + y[i]
      fadd.d f4, f2, f4    ← z[i] + (a + y[i])
      fsd    f4, t(x1)
      bnez   x1, loop
      sub    x1, x1, 8     ← delay slot
```

#### Dependencias

- `fadd f4, f0, f4`: RAW ← `fld f4` (4 ciclos)
- `fadd f4, f2, f4`: RAW ← `fld f2` (4 ciclos) y RAW ← `fadd` anterior (3 ciclos)
- `fsd f4`: RAW ← 2ª `fadd`

#### Diagrama instrucción/tiempo (primera iteración, Issue desde ciclo 1)

| Instrucción           | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  |10  |11  |12  |
|-----------------------|----|----|----|----|----|----|----|----|----|----|----|-----|
| fld f4, y(x1)         | Is | E1 | E2 | E3 | E4 | W  |    |    |    |    |    |    |
| fld f2, z(x1)         | Is | E1 | E2 | E3 | E4 | W  |    |    |    |    |    |    |
| fadd f4, f0, f4       | Is | *  | *  | *  | *  | Is | E1 | E2 | E3 | W  |    |    |
| fadd f4, f2, f4       | Is | *  | *  | *  | *  | *  | *  | *  | *  | Is | E1 |... |
| fsd f4, t(x1)         | Is | *  | *  | *  | *  | *  | *  | *  | *  | *  | *  |... |
| bnez x1, loop         |    |    |    |    |    |    |    | Is | W  |    |    |    |
| sub x1, x1, 8         |    |    |    |    |    |    |    |    | Is | W  |    |    |

> Ambas `fld` se emiten en ciclo 1 y completan en ciclo 5 (WB=ciclo 5→6 con la regla no-mismos-ciclos).

**Corrección detallada** (no se puede escribir e iniciar en el mismo ciclo):

- `fld f4`: Issue=1, EX=2-5, WB=6
- `fld f2`: Issue=1, EX=2-5, WB=6
- `fadd f4, f0, f4`: Issue=7 (espera WB de fld f4 en ciclo 6 → empieza ciclo 7), EX=8-10, WB=11
- `fadd f4, f2, f4`: Issue=12 (espera WB de f2=6 ✓ pero espera f4 de fadd anterior WB=11 → Issue=12), EX=13-15, WB=16
- `fsd f4, t(x1)`: Issue=12, espera f4 WB=16 → EX=17-20, WB=21
- `bnez x1, loop`: entero, CPI=1 → Issue cuando libre
- `sub x1, x1, 8`: delay slot, Issue tras bnez

**Ciclos por iteración ≈ 21 ciclos**

Para iteraciones solapadas (2ª iteración empieza cuando hay ER y buffers libres):
- Los buffers de load se liberan en ciclo 6, ER de suma en ciclo 11, 16
- La 2ª iteración puede comenzar sus `fld` en ciclo 2 (buf disponibles)
- Cuello de botella: cadena crítica = 4+3+3 = 10 ciclos de EX + WBs

Iniciando 2ª iteración en ciclo 2:
- 2ª fld f4: WB = ciclo 7
- 2ª fadd1: Issue=8, EX=9-11, WB=12
- 2ª fadd2: Issue=13, EX=14-16, WB=17
- 2ª fsd: EX=18-21, WB=22

Diferencia entre inicio de iteración 1 y 2 = **1 ciclo** (solapamiento total en loads). El verdadero cuello de botella es la cadena de sumas: fadd1 + fadd2 = 3+3=6 ciclos, pero secuenciales = no se pueden solapar entre sí. **Ciclos/iteración sostenidos ≈ 7 ciclos** (cadena add: 6 EX + 1 WB overhead).

#### Tiempo de ejecución

```
Ciclos/iteración (estado estacionario) ≈ 7 ciclos
Iteraciones = n (un elemento por iteración, paso de 8 bytes)

T = n × 7 / (100×10⁶) segundos = 7n × 10⁻⁸ s = 70n ns
```

**T(n) = 7n / 10⁸ s = 70n ns**

---

### 7. Diagrama instrucción-tiempo con 2 FPADDD y 1 FPMUL/DIVD

**Enunciado:** 2 UF FPADDD (3 ciclos, segmentada, 1 ER c/u), 1 FPMULD/DIVD (4/10 ciclos, no segmentada, 1 ER c/u). No se puede escribir e iniciar en el mismo ciclo. 1 ER por UF.

```
I1: fadd.d f2, f2, f4
I2: fadd.d f0, f4, f0
I3: fmul.d f4, f2, f0
I4: fadd.d f6, f0, f0
I5: fdiv.d f4, f4, f6
I6: fadd.d f2, f6, f6
I7: fadd.d f0, f2, f6
I8: fmul.d f2, f2, f0
```

#### Dependencias

- I3 RAW← I1 (f2) y I2 (f0)
- I4 RAW← I2 (f0)
- I5 RAW← I3 (f4) y I4 (f6)
- I6 RAW← I4 (f6)
- I7 RAW← I6 (f2)
- I8 RAW← I6 (f2) y I7 (f0)

#### Restricciones estructurales

- Solo 1 ER por unidad → la 2ª fadd debe esperar si la 1ª ADD aún ocupa su ER
- FPADD es segmentada: puede aceptar nueva instrucción cada ciclo si hay ER libre
- FPMUL no segmentada: ocupa la unidad los 4 ciclos completos

#### Tabla de ciclos

| Instrucción         | Issue | EX inicio | EX fin | WB  | Tipo   |
|---------------------|-------|-----------|--------|-----|--------|
| I1: fadd f2,f2,f4   | 1     | 2         | 4      | 5   | ADD    |
| I2: fadd f0,f4,f0   | 2     | 3         | 5      | 6   | ADD    |
| I3: fmul f4,f2,f0   | 7 (*) | 8         | 11     | 12  | MUL    |
| I4: fadd f6,f0,f0   | 7 (**)| 8         | 10     | 11  | ADD    |
| I5: fdiv f4,f4,f6   | 13(***) | 14      | 23     | 24  | DIV    |
| I6: fadd f2,f6,f6   | 12(+) | 13        | 15     | 16  | ADD    |
| I7: fadd f0,f2,f6   | 17(++) | 18       | 20     | 21  | ADD    |
| I8: fmul f2,f2,f0   | 22(+++)| 23       | 26     | 27  | MUL    |

> (*) I3 espera I1 (WB=5) e I2 (WB=6) → no puede iniciar hasta ciclo 7 (Issue) dado que WB=6 y no mismos ciclos.  
> (**) I4 espera I2 (WB=6) → Issue=7. Conflicto con I3 en MUL y ADD: I3→MUL (libre), I4→ADD (libre). Ambas pueden emitirse en ciclo 7 pero hay restricción de 1 ER por UF. ADD tiene 2 UF → I4 puede ir a la 2ª ADD. Issue=7.  
> (***) I5 espera I3 (WB=12) e I4 (WB=11) → Issue=13.  
> (+) I6 espera I4 (WB=11) → Issue=12. ER de ADD libres (I4 terminó).  
> (++) I7 espera I6 (WB=16) → Issue=17.  
> (+++) I8 espera I6 (f2, WB=16) e I7 (f0, WB=21) → Issue=22.

#### Tipos de riesgos

- **RAW (Read After Write):** I3←I1,I2; I4←I2; I5←I3,I4; I6←I4; I7←I6; I8←I6,I7
- **WAW (Write After Write):** I1 y I3 escriben f2; I3 e I5 escriben f4 — Tomasulo resuelve con etiquetas, no hay stall real
- **WAR:** No se producen con Tomasulo (renaming implícito con ER)
- **Estructural:** Posible competencia por la unidad MUL/DIV (no segmentada, 1 ER) entre I3 y futura instrucción MUL; y por el CDB compartido

---

### 8. Diagrama instrucción-tiempo con 2 FPADD y 2 FPMUL/DIV

**Enunciado:** 2 UF FPADD/SUBD (3/3 ciclos, no segmentadas, 1 ER c/u), 2 UF FPMULD/DIVD (6/12 ciclos, no segmentadas, 1 ER c/u). No se puede escribir e iniciar en el mismo ciclo. 1 ER por UF.

```
I1: fdiv.d f2, f2, f6
I2: fadd.d f4, f6, f4
I3: fmul.d f8, f2, f4
I4: fdiv.d f0, f6, f4
I5: fadd.d f2, f4, f0
I6: fadd.d f8, f8, f10
I7: fmul.d f0, f2, f8
I8: fsub.d f12, f2, f4
```

#### Dependencias

- I3 RAW← I1 (f2) y I2 (f4)
- I4 RAW← I2 (f4)
- I5 RAW← I4 (f0)
- I6 RAW← I3 (f8)
- I7 RAW← I5 (f2) y I6 (f8)
- I8 RAW← I5 (f2)

#### Tabla de ciclos

| Instrucción           | Issue | EX inicio | EX fin | WB  | Tipo | Espera por         |
|-----------------------|-------|-----------|--------|-----|------|--------------------|
| I1: fdiv f2, f2, f6   | 1     | 2         | 13     | 14  | DIV  | —                  |
| I2: fadd f4, f6, f4   | 1     | 2         | 4      | 5   | ADD  | —                  |
| I3: fmul f8, f2, f4   | 15(*) | 16        | 21     | 22  | MUL  | I1(f2,WB=14)       |
| I4: fdiv f0, f6, f4   | 6(**) | 7         | 18     | 19  | DIV  | I2(f4,WB=5)        |
| I5: fadd f2, f4, f0   | 20(+) | 21        | 23     | 24  | ADD  | I4(f0,WB=19)       |
| I6: fadd f8, f8, f10  | 23(++)| 24        | 26     | 27  | ADD  | I3(f8,WB=22)       |
| I7: fmul f0, f2, f8   | 28(*)| 29        | 34     | 35  | MUL  | I5(WB=24),I6(WB=27)|
| I8: fsub f12, f2, f4  | 25(+++)| 26       | 28     | 29  | SUB  | I5(f2,WB=24)       |

> (*) I3 espera I1 (f2, WB=14) e I2 (f4, WB=5) → cuello I1, Issue=15.  
> (**) I4 espera I2 (f4, WB=5) → Issue=6. Usa la 2ª unidad DIV (libre).  
> (+) I5 espera I4 (f0, WB=19) → Issue=20.  
> (++) I6 espera I3 (f8, WB=22) → Issue=23. Puede usar la 2ª ADD (I5 usa la 1ª ADD en ciclo 20).  
> (+++) I8 espera I5 (f2, WB=24) → Issue=25.  
> (***) I7 espera I5 (f2, WB=24) e I6 (f8, WB=27) → Issue=28.

#### Tipos de riesgos

- **RAW:** I3←I1,I2; I4←I2; I5←I4; I6←I3; I7←I5,I6; I8←I5
- **WAW:** I1,I5 escriben f2; I3,I6 escriben f8; I4,I7 escriben f0 → resueltos por Tomasulo con renaming
- **WAR:** No aplica con Tomasulo
- **Estructural:** Competencia por las 2 unidades DIV (I1 e I4 se solapan en ejecución, pero hay 2 unidades disponibles); competencia por CDB cuando múltiples instrucciones quieren escribir simultáneamente

---

### 9. Tomasulo con y sin ROB (primera iteración)

**Enunciado:** Pipeline con Tomasulo. UF: FPADD (2 ciclos, seg.), FPDIV (6 ciclos, seg.), FPMUL (3 ciclos, seg.), INT ALU (1 ciclo, seg.), MEM (2 ciclos, seg.). Los datos escritos no se pueden usar hasta el ciclo siguiente. Un solo CDB.

```
loop: fld    f2, 0(x1)
      fmul.d f4, f2, f0
      fld    f6, 0(x2)
      fdiv.d f4, f6, f4
      fsd    f4, 0(x1)
      fmul.d f4, f6, f0
      fadd.d f4, f8, f2
      addi   x1, x1, 8
      addi   x2, x2, 8
      slti   x3, x2, done
      beqz   x3, loop
```

#### a) Sin especulación (Tomasulo estándar)

Dependencias clave:
- `fmul f4,f2,f0` RAW← `fld f2` (2 ciclos MEM)
- `fdiv f4,f6,f4` RAW← `fld f6` (2 ciclos) y `fmul` (3 ciclos)
- `fsd f4,0(x1)` RAW← `fdiv` (6 ciclos)
- `fmul f4,f6,f0` RAW← `fld f6` (2 ciclos) — independiente de fdiv
- `fadd f4,f8,f2` RAW← `fld f2` (disponible temprano)
- `beqz x3,loop` espera `slti` que espera `addi x2`

| Instrucción           | Issue | EX ini | EX fin | WB |
|-----------------------|-------|--------|--------|----|
| fld f2, 0(x1)         | 1     | 2      | 3      | 4  |
| fmul f4, f2, f0       | 2     | 5(*)   | 7      | 8  |
| fld f6, 0(x2)         | 3     | 4      | 5      | 6  |
| fdiv f4, f6, f4       | 4     | 9(**)  | 14     | 15 |
| fsd f4, 0(x1)         | 5     | 16(***)| 17     | 18 |
| fmul f4, f6, f0       | 6     | 7(+)   | 9      | 10 |
| fadd f4, f8, f2       | 7     | 5(++)  | 6      | 7  |
| addi x1, x1, 8        | 8     | 9      | 9      | 10 |
| addi x2, x2, 8        | 9     | 10     | 10     | 11 |
| slti x3, x2, done     | 10    | 12(+++)| 12     | 13 |
| beqz x3, loop         | 11    | 14     | 14     | 15 |

> (*) `fmul` espera f2 (WB=4) → EX inicia ciclo 5.  
> (**) `fdiv` espera f6 (WB=6) y f4 de `fmul` (WB=8) → EX inicia ciclo 9.  
> (***) `fsd` espera f4 de `fdiv` (WB=15) → EX ciclo 16.  
> (+) 2ª `fmul` espera f6 (WB=6) → EX ciclo 7. Conflicto CDB con 1ª `fmul` en WB=8 y 2ª en WB=10 → OK.  
> (++) `fadd f4,f8,f2` espera f2 (WB=4) → EX podría ser ciclo 5 (pero MUL ocupa la UF en ciclo 5). ADD es distinta UF → EX=5, WB=7. Pero tiene WAW sobre f4 con `fmul f4,f2,f0` → Tomasulo lo maneja con etiquetas.  
> (+++) `slti` espera `addi x2` (WB=11) → EX=12.

**Nota:** El `beqz` no puede resolverse hasta ciclo 15 (WB de `slti`=13 → `beqz` Issue=14 si hay stall, pero es entero). Con Tomasulo sin especulación, no se buscan instrucciones de la siguiente iteración hasta resolver el salto.

#### b) Con especulación (ROB)

Con ROB, las instrucciones se emiten especulativamente **antes** de resolver el salto. El ROB actúa como store buffer, por lo que los `fsd` se completan especulativamente y sólo se confirman al hacer commit.

Diferencias clave con respecto al diagrama sin especulación:
1. Tras el `beqz`, se buscan y emiten instrucciones de la siguiente iteración especulativamente en cuanto hay predicción.
2. Los resultados se escriben en el ROB en lugar de directamente en los registros; el renaming usa entradas del ROB.
3. Los commits ocurren en orden (head del ROB) aunque la ejecución sea fuera de orden.

Con predicción de salto tomado (bucle) y ROB de, por ejemplo, 8 entradas:
- Las instrucciones de la 2ª iteración se empiezan a buscar en ciclo ~12 (tras la decodificación del `beqz`)
- Si el salto se predice correctamente, no hay penalización
- Si el salto se predice erróneamente, se hace squash de todas las instrucciones especulativas en el ROB

El diagrama sería igual al anterior para las instrucciones de la 1ª iteración, con la adición de que las instrucciones de la 2ª iteración aparecerían solapadas a partir de ciclo ~12-13, permitiendo mayor ILP.

**Ventaja principal del ROB:** Permite especulación más allá del salto, reduciendo el tiempo efectivo de ejecución del bucle cuando la predicción es correcta.

---

### 10. Branch Target Buffer de 2 bits — análisis de predicción

**Enunciado:** Procesador con BTB de 2 bits (predictor de 2 bits). Estado inicial: "no salta fuerte". El salto `bnez x1, loop` se ejecuta 4 veces (bucle con x1=4 decrementing a 0):

```
addi x1, x0, 4
loop: sub  x5, x5, x2
      add  x2, x2, 1
      subi x1, x1, 1
      bnez x1, loop       ← salto condicional
      andi x6, x5, 63
      ori  x6, x6, 128
```

#### Comportamiento del salto

| Iteración | x1 antes de bnez | ¿Salta? | Comportamiento real |
|-----------|-----------------|---------|---------------------|
| 1         | 3               | Sí (T)  | Toma el salto       |
| 2         | 2               | Sí (T)  | Toma el salto       |
| 3         | 1               | Sí (T)  | Toma el salto       |
| 4         | 0               | No (NT) | No toma el salto    |

#### Evolución del predictor de 2 bits

Estados: `00`=NT fuerte, `01`=NT débil, `10`=T débil, `11`=T fuerte

Estado inicial: `00` (no salta fuerte)

| Ejecución | Estado inicial | Real  | Predicción | ¿Fallo? | Estado final |
|-----------|---------------|-------|------------|---------|--------------|
| 1         | 00 (NT)       | T     | NT         | **Sí**  | 01           |
| 2         | 01 (NT débil) | T     | NT         | **Sí**  | 10           |
| 3         | 10 (T débil)  | T     | T          | No      | 11           |
| 4         | 11 (T fuerte) | NT    | T          | **Sí**  | 10           |

**Total de fallos de predicción: 3** (iteraciones 1, 2 y 4)

#### Instrucciones ejecutadas tras el salto (pipeline 5 etapas: IF/ID/EX/ME/WB)

Dado que el BTB se accede en IF y da respuesta al final de IF, hay una penalización de 1 ciclo si la predicción falla (etapa ID ya cargada con la instrucción incorrecta).

| Iteración | Predicción | Real | Instrucción tras salto (en ID) | ¿Cancelada? |
|-----------|-----------|------|-------------------------------|-------------|
| 1         | NT        | T    | `andi x6, x5, 63`             | **Sí (X)**  |
| 2         | NT        | T    | `andi x6, x5, 63`             | **Sí (X)**  |
| 3         | T         | T    | 1ª instr. del bucle (`sub`)   | No          |
| 4         | T         | NT   | 1ª instr. del bucle (`sub`)   | **Sí (X)**  |

En los fallos, la instrucción en ID se cancela (fases marcadas con X) y se vuelve a buscar la instrucción correcta.

---

### 11. Fallos de predicción en bucles anidados con BTB de 1 y 2 bits

**Enunciado:** Bucles anidados a 3 niveles (i, j, k). Se estudian predictores de 1 bit y 2 bits, partiendo de "no saltar".

```
for i = 1..n
  for j = 1..n
    for k = 1..n
      ...
    end  ← salto k
  end    ← salto j
end      ← salto i
```

#### Análisis para el bucle más interno (salto k)

El salto k se toma n-1 veces (T) y luego 1 vez no se toma (NT), por iteración de j.

**Predictor de 1 bit:**
- Estado inicial: NT
- 1ª ejecución real=T → fallo, cambia a T
- Ejecuciones 2..n-1: predice T, real=T → sin fallos
- Ejecución n: predice T, real=NT → **fallo**, cambia a NT
- **Fallos por "vuelta" del bucle k:** 2 (al entrar y al salir)
- Total para el bucle k: 2 fallos por cada iteración de (i,j) → 2·n² fallos

**Predictor de 2 bits:**
- Estado inicial: NT fuerte (00)
- 1ª ejecución real=T → fallo → estado 01 (NT débil) → **fallo**
- 2ª ejecución real=T → fallo → estado 10 → **fallo** (si n≥2)
- 3ª..n-1 ejecuciones real=T → predice T → sin fallos
- n-ésima ejecución real=NT → predice T → **fallo**, estado → 10 (T débil)
- Para la siguiente vuelta de j: estado es T débil, 1ª real=T → sin fallo (predice T correctamente)
- Excepto la primera vuelta de cada j donde el estado inicial puede variar

**Fallos por el bucle k con 2 bits:**
- Primera vuelta de j: 2 fallos (al entrar: estado viene de NT fuerte o anterior vuelta)
- Vueltas intermedias de j: 1 fallo (solo al salir, al entrar ya predice T)
- Última vuelta de j: 1 fallo al salir + posiblemente 0 al entrar

Aproximación para n grande:
- **1 bit:** ~2·n² fallos (2 por cada uno de los n² pares (i,j))
- **2 bits:** ~n² + 2·n fallos (1 por cada vuelta de j excepto las 2 iniciales, más los fallos de inicio/fin de j e i)

Para n grande, el predictor de **2 bits** reduce significativamente los fallos respecto al de **1 bit**, siendo aproximadamente la mitad para el bucle k.

---

### 12. Tournament Predictor — análisis de BEQ y BNE

**Enunciado:** Tournament Predictor (Alpha 21264). Instrucción `beq x1, loop` con patrón T-T-NT-NT repetido (1000 ejecuciones). Instrucción `bne x2, loop` siempre tomada.

#### a) Entradas de la Tabla de Predicción Local para BEQ

El Tournament Predictor usa:
- **Predictor local:** tabla de historia local indexada por PC, con contador de saturación de 2 bits
- **Predictor global:** basado en historia global de saltos
- **Selector:** elige entre local y global basándose en cuál ha acertado más

El patrón de BEQ es: T-T-NT-NT-T-T-NT-NT-...  
La historia local de 2 bits para BEQ captura los últimos 2 resultados:

Las últimas 100 ejecuciones siguen el mismo patrón periódico (período 4). Las entradas de la tabla local accedidas son aquellas correspondientes a las historias de 2 bits:

| Historia (2 últimos) | Siguiente real | Frecuencia |
|----------------------|----------------|------------|
| TT                   | NT             | 25 veces   |
| T-NT                 | NT             | 25 veces   |
| NT-NT                | T              | 25 veces   |
| NT-T                 | T              | 25 veces   |

Al final de las 1000 ejecuciones, los contadores de 2 bits de esas 4 entradas habrán convergido a:
- Historia TT → predice NT → contador en "NT fuerte" (00)
- Historia TNT → predice NT → contador en "NT fuerte" (00)
- Historia NTNT → predice T → contador en "T fuerte" (11)
- Historia NTT → predice T → contador en "T fuerte" (11)

#### b) Fallos en las siguientes 1000 ejecuciones

Dado que el patrón es perfectamente periódico y los contadores han convergido, el predictor local habrá aprendido perfectamente el patrón T-T-NT-NT.

Con el predictor local perfectamente entrenado:
- Cada ciclo del patrón (4 ejecuciones) → **0 fallos** (predice perfectamente)
- En 1000 ejecuciones = 250 ciclos completos del patrón

**Fallos de predicción = 0** en las siguientes 1000 ejecuciones (asumiendo que el selector ha elegido el predictor local para BEQ, que es óptimo para este patrón regular).

*Nota:* Si el selector eligiera el predictor global (compartido con BNE que siempre salta), podría haber interferencias. Con el selector entrenado correctamente, elegirá el local para BEQ.

---

### 13. Tomasulo con ROB — ciclos de ejecución y valores de f12

**Enunciado:** Tomasulo sin especulación. UF: MUL/DIV segmentada (12/16 ciclos, 3 ER), ADD/SUB no segmentada (4 ciclos, 1 ER), Load segmentado (2 ciclos, buf 4), Store segmentado (2 ciclos, buf 3). Datos escritos en WB no disponibles hasta el ciclo siguiente. 1 CDB. ER se liberan al final de WB.

```
I1:  fld   f2,  0(x1)
I2:  fld   f4,  0(x2)
I3:  fmul.d f6, f2, f4
I4:  fadd.d f10, f2, f6
I5:  fld   f6,  0(x3)
I6:  fld   f8,  0(x4)
I7:  fadd.d f12, f6, f10
I8:  fmul.d f12, f8, f12
I9:  fadd.d f12, f12, f4
I10: fadd.d f10, f12, f8
I11: fsub.d f14, f2, f10
I12: fadd.d f12, f12, f14
I13: fsd   f12, 0(x1)
I14: fsd   f14, 0(x2)
```

#### Dependencias

| Instrucción | Espera a              | Registro |
|-------------|-----------------------|----------|
| I3 fmul     | I1 (f2), I2 (f4)      | f6       |
| I4 fadd     | I1 (f2), I3 (f6)      | f10      |
| I7 fadd     | I5 (f6), I4 (f10)     | f12      |
| I8 fmul     | I6 (f8), I7 (f12)     | f12      |
| I9 fadd     | I8 (f12), I2 (f4)     | f12      |
| I10 fadd    | I9 (f12), I6 (f8)     | f10      |
| I11 fsub    | I1 (f2), I10 (f10)    | f14      |
| I12 fadd    | I9 (f12), I11 (f14)   | f12      |
| I13 fsd     | I12 (f12)             | —        |
| I14 fsd     | I11 (f14)             | —        |

#### Tabla de ciclos

> Latencias: fld=2, fadd/fsub=4, fmul=12 (segmentado: puede aceptar nueva en cualquier ciclo si hay ER). ADD no segmentada: bloquea su UF 4 ciclos. CDB compartido: si dos instrucciones terminan en el mismo ciclo, una espera al siguiente.

| Instr.  | Issue | EX ini | EX fin | WB | Conflictos/Notas                              |
|---------|-------|--------|--------|----|-----------------------------------------------|
| I1 fld  | 1     | 2      | 3      | 4  | —                                             |
| I2 fld  | 2     | 3      | 4      | 5  | —                                             |
| I3 fmul | 3     | 6(*)   | 17     | 18 | Espera I1(WB=4),I2(WB=5) → EX=6              |
| I4 fadd | 4     | 6(**)  | 9      | 10 | Espera I1(WB=4),I3(WB=18)→ cuello I3; pero f2 libre en 4, f6 no hasta 18 → EX=19 |
| I5 fld  | 5     | 6      | 7      | 8  | —                                             |
| I6 fld  | 6     | 7      | 8      | 9  | —                                             |
| I7 fadd | 7     | —(+)   | —      | —  | Espera I5(f6,WB=8),I4(f10,WB=?) → bloqueada  |
| I8 fmul | 8     | —      | —      | —  | Espera I6(f8,WB=9),I7(f12,WB=?)              |
| I9 fadd | 9     | —      | —      | —  | Espera I8,I2                                  |
| I10 fadd| 10    | —      | —      | —  | Espera I9,I6                                  |
| I11 fsub| 11    | —      | —      | —  | Espera I1(f2,WB=4✓),I10                       |
| I12 fadd| 12    | —      | —      | —  | Espera I9,I11                                 |
| I13 fsd | 13    | —      | —      | —  | Espera I12                                    |
| I14 fsd | 14    | —      | —      | —  | Espera I11                                    |

**Recalculando con la cadena crítica correcta:**

I4 `fadd f10, f2, f6`: espera I1(f2, WB=4) e I3(f6, WB=18) → Issue=4, EX comienza en ciclo 19 (WB de I3=18, no mismo ciclo), EX=19-22, WB=23.

> Nota: ADD no es segmentada, por lo que si I4 empieza en ciclo 19, bloquea la unidad hasta ciclo 22.

I7 `fadd f12, f6, f10`: espera I5(f6, WB=8) e I4(f10, WB=23) → EX=24, WB=27. Pero ADD no segmentada — si I4 la ocupa hasta ciclo 22, I7 puede empezar en 24 (libre).

I8 `fmul f12, f8, f12`: espera I6(f8, WB=9) e I7(f12, WB=27) → EX=28, EX=28-39, WB=40.

I9 `fadd f12, f12, f4`: espera I8(WB=40), I2(f4,WB=5) → EX=41, WB=44.

I10 `fadd f10, f12, f8`: espera I9(WB=44), I6(f8,WB=9) → EX=45, WB=48.

I11 `fsub f14, f2, f10`: espera I1(f2,WB=4), I10(WB=48) → EX=49, WB=52.

I12 `fadd f12, f12, f14`: espera I9(f12,WB=44), I11(f14,WB=52) → EX=53, WB=56.

I13 `fsd f12, 0(x1)`: espera I12(WB=56) → EX=57-58, WB=59.
I14 `fsd f14, 0(x2)`: espera I11(WB=52) → EX=53-54, WB=55.

**Tabla final de ciclos:**

| Instrucción           | Issue | EX ini | EX fin | WB |
|-----------------------|-------|--------|--------|----|
| I1  fld  f2, 0(x1)   | 1     | 2      | 3      | 4  |
| I2  fld  f4, 0(x2)   | 2     | 3      | 4      | 5  |
| I3  fmul f6, f2, f4   | 3     | 6      | 17     | 18 |
| I4  fadd f10, f2, f6  | 4     | 19     | 22     | 23 |
| I5  fld  f6, 0(x3)   | 5     | 6      | 7      | 8  |
| I6  fld  f8, 0(x4)   | 6     | 7      | 8      | 9  |
| I7  fadd f12, f6, f10 | 7     | 24     | 27     | 28 |
| I8  fmul f12, f8, f12 | 8     | 29     | 40     | 41 |
| I9  fadd f12, f12, f4 | 9     | 42     | 45     | 46 |
| I10 fadd f10, f12, f8 | 10    | 47     | 50     | 51 |
| I11 fsub f14, f2, f10 | 11    | 52     | 55     | 56 |
| I12 fadd f12, f12, f14| 12    | 57     | 60     | 61 |
| I13 fsd  f12, 0(x1)  | 13    | 62     | 63     | 64 |
| I14 fsd  f14, 0(x2)  | 14    | 57     | 58     | 59 |

> Ajuste: I7 EX comienza en ciclo 24 (ADD libre en 23, WB I4=23 → no mismo ciclo → 24). I8 EX en 29 (WB I7=28 → 29). I9 EX en 42 (WB I8=41 → 42). I10 EX en 47. I11 EX en 52. I12 EX en 57 (WB I11=56 → 57). 

> Posible conflicto CDB en ciclo 41 (I8 WB) vs otras instrucciones: se asume que solo I8 quiere el CDB en ese ciclo (las demás están esperando).

#### b) Valores de f12 (campo Valor y Tag)

| Ciclo | Evento                              | Tag de f12 | Valor de f12 |
|-------|-------------------------------------|------------|--------------|
| 1–6   | I3 emitida (Issue=3)                | ER_MUL(I3) | —            |
| 7     | I7 emitida (Issue=7), nueva tag     | ER_ADD(I7) | —            |
| 8     | I8 emitida (Issue=8), nueva tag     | ER_MUL(I8) | —            |
| 9     | I9 emitida (Issue=9), nueva tag     | ER_ADD(I9) | —            |
| 12    | I12 emitida (Issue=12), nueva tag   | ER_ADD(I12)| —            |
| 28    | I7 WB: escribe f12                  | ER_MUL(I8) | — (I8 es la productora vigente, f12 apunta a I8) |
| 41    | I8 WB: escribe f12                  | ER_ADD(I9) | — |
| 46    | I9 WB: escribe f12                  | ER_ADD(I12)| — |
| 61    | I12 WB: escribe f12                 | —          | f12 = I12_resultado |

**Resumen de cambios en el Tag de f12:**

El Tag de f12 cambia cada vez que se emite una nueva instrucción que escribe en f12:
- Ciclo 3: Tag ← I3 (fmul f6 → en realidad escribe f6, no f12; **corrección:** I3 escribe f6, no f12)

**Corrección de instrucciones que escriben f12:**
- I7: `fadd f12` → Tag de f12 ← I7 al Issue (ciclo 7)
- I8: `fmul f12` → Tag de f12 ← I8 al Issue (ciclo 8)
- I9: `fadd f12` → Tag de f12 ← I9 al Issue (ciclo 9)
- I12: `fadd f12` → Tag de f12 ← I12 al Issue (ciclo 12)

| Ciclo | Cambio en f12                              |
|-------|--------------------------------------------|
| 7     | Tag = ER(I7); Valor = indefinido           |
| 8     | Tag = ER(I8); Valor = indefinido           |
| 9     | Tag = ER(I9); Valor = indefinido           |
| 12    | Tag = ER(I12); Valor = indefinido          |
| 28    | I7 escribe en CDB: actualiza ER que esperan I7, pero Tag de f12 sigue apuntando a I12 (la última instrucción que emitió a f12) |
| 41    | I8 WB: ídem                                |
| 46    | I9 WB: ídem                                |
| 61    | I12 WB: **Tag = vacío; Valor = resultado final de f12** |

El valor arquitectural de f12 solo se actualiza definitivamente en **ciclo 61** cuando I12 (la última instrucción que escribe f12 en orden programa) completa su WB.
