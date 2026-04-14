# Ejercicios EC

## Mario González García

---

## Caché

### Problemas Básicos

---

Ejercicio 1

Enunciado: En un computador con cachés separadas de datos e instrucciones, de 32KB cada una y bloques de 256B, se quiere ejecutar el siguiente código C que realiza la operación matricial `C = A+B`:

```c
#define N 128
int A[N][N];
int B[N][N];
int C[N][N];

for (i=0; i < N; i++)
    for (j=0; j < N; j++)
        C[i][j] = A[i][j] + B[i][j];
```

La matriz A se ubica en `0x0C000000`, B y C se almacenan consecutivamente a continuación de A. Las variables `i`, `j` y las direcciones base de los arrays están en registros.

- a) Calcular el número de fallos de caché en lectura (load) y escritura (store) si la caché es de emplazamiento directo sin asignación en escritura.
- b) Repetir el cálculo para una caché asociativa por conjuntos de 2 vías con política LRU, sin y con asignación en escritura.
- c) Variante con array de structs `AB[N][N]` con A y B entrelazados. Calcular fallos y comparar.
- d) ¿Puede obtenerse un resultado similar al del apartado c con el código original sin aumentar el tamaño de la caché?

Resolución:

#### Parámetros de la caché

- Tamaño: 32 KB = 32768 B
- Bloque: 256 B
- Número de bloques en caché: 32768 / 256 = **128 bloques**
- Cada `int` ocupa 4 bytes → cada bloque almacena 256/4 = **64 enteros**
- Cada fila de la matriz tiene N=128 enteros → ocupa 128×4 = 512 B = **2 bloques**
- Cada matriz completa: 128×128×4 = 65536 B = **256 bloques** (no cabe en caché)
- Las tres matrices A, B, C son contiguas. El número de bloque físico de `M[i][j]` determina el índice en caché (emplazamiento directo: índice = nº bloque % 128).

**Observación clave de aliasing:** A ocupa bloques 0–255, B los bloques 256–511, C los bloques 512–767. Con emplazamiento directo (128 bloques), el bloque k de A, el bloque k de B y el bloque k de C mapean al **mismo conjunto** (índice k % 128), causando conflictos en cada acceso.

#### a) Emplazamiento directo, sin asignación en escritura

En cada iteración del bucle interno se accede a `A[i][j]`, `B[i][j]` y se escribe `C[i][j]`.

**Lecturas (loads):**

Para cada par (i, j), se leen A[i][j] y B[i][j]. Dado que A y B mapean al mismo índice de caché (sus bloques están separados exactamente 256 bloques, múltiplo de 128), al cargar un bloque de A se expulsa el bloque de B que ocupa el mismo índice, y viceversa. Esto provoca **thrashing**: cada acceso a A expulsa a B y viceversa.

- Total de accesos de lectura: 128×128×2 = 32768 lecturas
- En cada iteración j, el acceso a A[i][j] provoca fallo y carga bloque (si no estaba); luego el acceso a B[i][j] provoca fallo (su bloque mapea al mismo índice y A lo ha ocupado). Al volver al siguiente j dentro del mismo bloque, el bloque de A sigue siendo válido solo si B no lo expulsó, pero B lo expulsa. Resultado: **fallo en cada acceso de lectura a A y a B** para todos los elementos: 128×128 + 128×128 = **32768 fallos de lectura**.

**Escrituras (stores):**

Sin asignación en escritura (no write-allocate): en un fallo de escritura se escribe directamente en memoria principal sin traer el bloque a caché. C[i][j] mapea al mismo índice que A y B, pero como no se asigna en escritura, no se carga en caché.

- Todos los stores a C fallan (el bloque no está en caché) → **16384 fallos de escritura** (128×128 stores, cada uno con fallo).

> **Resultado a): 32768 fallos de lectura + 16384 fallos de escritura = 49152 fallos totales.**

#### b) Asociativa por conjuntos de 2 vías, LRU

Con 2 vías y 128 conjuntos (32KB / 256B / 2 = 64 conjuntos, en realidad: 32768 / (256×2) = 64 conjuntos).

**Recalculamos:** con 2 vías y bloques de 256B → número de conjuntos = 32768 / (256×2) = **64 conjuntos**.

Bloques de A: 256 bloques → índice de conjunto = (nº bloque) % 64. Bloques de B también: su bloque k mapea al conjunto (256+k) % 64 = k % 64. Ídem C: (512+k) % 64 = k % 64. Los tres arrays siguen mapeando al **mismo conjunto**, pero ahora hay 2 vías → pueden coexistir 2 bloques por conjunto. Con 3 arrays compitiendo en el mismo conjunto, el thrashing persiste (3 > 2 vías).

**Sin asignación en escritura:**

El comportamiento de lectura es idéntico al apartado a: A y B compiten por las 2 vías junto con C. En cada iteración se accede a A[i][j] (fallo, carga bloque de A), luego B[i][j] (fallo, carga bloque de B, las 2 vías ocupadas por A y B), luego store a C sin asignación (no entra en caché). LRU: la vía más antigua se reemplaza, pero como son 3 arrays y 2 vías, siempre hay fallo.

- Fallos de lectura: **32768** (igual que antes)
- Fallos de escritura: **16384** (igual que antes)

> **Sin asignación en escritura: mismo resultado. El rendimiento no mejora.**

**Con asignación en escritura:**

Ahora el store a C[i][j] también trae el bloque a caché. Efecto: ahora compiten A, B y C por las 2 vías → sigue habiendo thrashing. Sin embargo, cuando se vuelve a escribir C[i][j+1] dentro del mismo bloque, ese bloque ya estaría en caché (si no fue expulsado). Con 3 arrays y 2 vías LRU:

- Patrón en j dentro de un bloque (64 elementos por bloque): acceso A → fallo, carga A; acceso B → fallo, carga B (expulsa A, LRU); store C → fallo, carga C (expulsa B, LRU). Siguiente j: acceso A → fallo (A fue expulsado), ...

El ciclo de 3 accesos con 2 vías produce fallo en todos los accesos.

- Fallos de lectura: **32768**
- Fallos de escritura: **16384** (todos los primeros accesos a cada bloque de C)

> **Con asignación en escritura: mismo resultado. El rendimiento tampoco mejora en este caso de thrashing de 3 vías.**

*Nota:* La mejora de la asociatividad 2 vías sería efectiva si solo hubiera 2 arrays compitiendo; con 3, el thrashing persiste.

#### c) Código con struct AB entrelazado, caché 2 vías con asignación en escritura

```c
struct { int A; int B; } AB[N][N];
int C[N][N];
```

Ahora `AB[i][j].A` y `AB[i][j].B` están contiguos en memoria (8 bytes juntos). Un bloque de 256B almacena 256/8 = **32 pares (A,B)**. El array C está a continuación.

**Mapeo de bloques:**
- AB ocupa 128×128×8 = 131072 B = **512 bloques** de 256B → índices 0–511, conjuntos 0–63 (módulo 64).
- C ocupa 256 bloques → índices 512–767, conjuntos 0–63 (módulo 64).

Para cada elemento `AB[i][j]`, el acceso a `.A` y `.B` está en el **mismo bloque**: un solo fallo trae ambos valores. Ya no hay thrashing entre A y B porque van juntos.

**Fallos de lectura:**
- En cada fila i, se acceden 128 pares AB[i][0..127]: ocupan 128×8/256 = **4 bloques** de AB por fila.
- Total bloques de AB: 128 filas × 4 bloques = **512 bloques**, pero la caché tiene 64 conjuntos × 2 vías = 128 entradas. Los bloques de AB repiten conjuntos, pero como el patrón es secuencial fila a fila, cada bloque se usa exactamente una vez antes de ser reemplazado.
- Fallos de lectura en AB: **512 fallos** (un fallo por cada bloque de AB, ya que cada bloque se lee por primera vez y no vuelve a usarse antes de ser expulsado).

**Fallos de escritura (con asignación):**
- C ocupa 256 bloques. En cada iteración j dentro de un bloque de C (64 elementos por bloque), el primer acceso al bloque produce un fallo y los siguientes 63 son aciertos. Pero C y AB pueden colisionar en conjuntos. Sin colisión, fallos en C = número de bloques de C = **256 fallos**.
- Con colisión: si C[i][j] mapea al mismo conjunto que AB[i][j], se produce thrashing. Pero al ir entrelazados A y B, el número de conflictos activos se reduce a solo 2 arrays (AB y C) vs. las 2 vías → **pueden coexistir** si no hay más de 2 bloques distintos en el mismo conjunto al mismo tiempo.

Con 2 vías y solo AB y C compitiendo, los bloques de AB se cargan secuencialmente (patrón lineal), y con LRU/FIFO coexisten en las 2 vías sin expulsión mutua frecuente. Cada bloque de C se carga una vez.

> **Resultado c): ≈ 512 + 256 = 768 fallos totales**, frente a los ~49152 del caso original.
>
> La mejora es drástica: el entrelazado elimina el thrashing entre A y B al unirlos en el mismo bloque.

#### d) ¿Resultado similar con el código original sin aumentar la caché?

**Sí, es posible** mediante una transformación de código llamada **fusión de arrays** (*array merging*), que es exactamente lo que hace el struct del apartado c: en lugar de tener A y B como matrices separadas, reorganizar los datos para que los elementos de A y B que se usan juntos estén en el mismo bloque de caché.

Otra alternativa sin cambiar la estructura de datos es aplicar **tiling/blocking**: dividir el problema en submatrices lo suficientemente pequeñas como para que quepan en caché sin thrashing. Sin embargo, dado que el acceso es puramente lineal (fila por fila, sin reutilización), el tiling no aportaría ventaja aquí.

La única forma efectiva con el código original es **cambiar el layout en memoria** (fusión de arrays o padding para evitar aliasing), que es equivalente a lo que hace el struct, o aumentar la asociatividad de la caché.

---

Ejercicio 2

Enunciado: Sea un computador con memoria principal de 64KB direccionable en bytes, con caché de 2KB, líneas de 256B y prebúsqueda bajo fallo (en caso de fallo se trae el bloque que lo provoca y el siguiente).

Secuencia de accesos: `0x3345`, `0x1244`, `0x3374`, `0x5BAC`, `0x132A`, `0x5C41`, `0x13AF`.

Mostrar la evolución de la caché en:
- a) Organización directa.
- b) Asociativa por conjuntos de 2 vías con LRU.
- c) Organización directa con buffer de 512B (almacena el bloque prebuscado, asociativo, política FIFO).

Resolución:

#### Parámetros de la caché

- Memoria principal: 64KB = 2^16 B → direcciones de **16 bits**
- Caché: 2KB = 2048B, líneas de 256B
- Número de bloques en caché: 2048/256 = **8 bloques**
- Offset de bloque: log2(256) = **8 bits** (bits 7:0)
- **Emplazamiento directo:** índice = 3 bits (bits 10:8), etiqueta = 5 bits (bits 15:11)
- **2 vías:** conjuntos = 4, índice = 2 bits (bits 9:8), etiqueta = 6 bits (bits 15:10)
- Prebúsqueda: en cada fallo se cargan el bloque accedido y el **bloque siguiente** (nº bloque + 1)

#### Descomposición de direcciones (emplazamiento directo)

| Dirección | Binario (16 bits) | Etiqueta (5b) | Índice (3b) | Offset (8b) |
|-----------|------------------|--------------|------------|------------|
| 0x3345    | 0011 0011 0100 0101 | 00110 | 011 | 01000101 |
| 0x1244    | 0001 0010 0100 0100 | 00010 | 010 | 01000100 |
| 0x3374    | 0011 0011 0111 0100 | 00110 | 011 | 01110100 |
| 0x5BAC    | 0101 1011 1010 1100 | 01011 | 011 | 10101100 |
| 0x132A    | 0001 0011 0010 1010 | 00010 | 011 | 00101010 |
| 0x5C41    | 0101 1100 0100 0001 | 01011 | 100 | 01000001 |
| 0x13AF    | 0001 0011 1010 1111 | 00010 | 011 | 10101111 |

#### a) Organización directa

Bloques de las direcciones y sus prebúsquedas:

- **0x3345** → bloque 0x33 (índice 3, etiqueta 0x06): **FALLO**. Carga bloque 0x33 (índice 3, etq 0x06) + prebúsqueda bloque 0x34 (índice 4, etq 0x06).
- **0x1244** → bloque 0x12 (índice 2, etiqueta 0x02): **FALLO**. Carga bloque 0x12 (índice 2, etq 0x02) + prebúsqueda bloque 0x13 (índice 3, etq 0x02). ⚠️ Expulsa bloque 0x33 del índice 3.
- **0x3374** → bloque 0x33 (índice 3, etq 0x06): **FALLO** (fue expulsado). Carga bloque 0x33 + prebúsqueda bloque 0x34 (índice 4, etq 0x06).
- **0x5BAC** → bloque 0x5B (índice 3, etq 0x0B): **FALLO**. Carga bloque 0x5B (índice 3, etq 0x0B) + prebúsqueda bloque 0x5C (índice 4, etq 0x0B). Expulsa bloque 0x33 del índice 3.
- **0x132A** → bloque 0x13 (índice 3, etq 0x02): **FALLO** (índice 3 tiene etq 0x0B). Carga bloque 0x13 + prebúsqueda bloque 0x14 (índice 4, etq 0x02). Expulsa bloque 0x5B.
- **0x5C41** → bloque 0x5C (índice 4, etq 0x0B): **FALLO** (índice 4 tiene etq 0x02 de la prebúsqueda anterior). Carga bloque 0x5C + prebúsqueda bloque 0x5D (índice 5, etq 0x0B).
- **0x13AF** → bloque 0x13 (índice 3, etq 0x02): **ACIERTO** (cargado en el paso 5).

**Resumen a):** 6 fallos, 1 acierto. Se producen 6 prebúsquedas.

Estado final de la caché (índice → etiqueta):

| Índice | Etiqueta | Bloque MP |
|--------|----------|-----------|
| 2      | 0x02     | 0x12      |
| 3      | 0x02     | 0x13      |
| 4      | 0x02     | 0x14      |
| 5      | 0x0B     | 0x5D      |

#### b) Asociativa por conjuntos de 2 vías, LRU

Con 2 vías: 4 conjuntos, índice = 2 bits (bits 9:8), etiqueta = 6 bits (bits 15:10).

Descomposición para 2 vías:

| Dirección | Etiqueta (6b) | Conjunto (2b) | Bloque |
|-----------|--------------|--------------|--------|
| 0x3345    | 001100       | 11           | 0x33   |
| 0x1244    | 000100       | 10           | 0x12   |
| 0x3374    | 001100       | 11           | 0x33   |
| 0x5BAC    | 010110       | 11           | 0x5B   |
| 0x132A    | 000100       | 11           | 0x13   |
| 0x5C41    | 010111       | 00           | 0x5C   |
| 0x13AF    | 000100       | 11           | 0x13   |

- **0x3345** (conj 3, etq 0x0C): **FALLO**. Carga bloque 0x33 en vía 0 + prebúsqueda 0x34 en vía 1. Conj 3: [0x33(LRU), 0x34].
- **0x1244** (conj 2, etq 0x04): **FALLO**. Carga 0x12 en vía 0 + prebúsqueda 0x13 en vía 1. Conj 2: [0x12, 0x13].
- **0x3374** (conj 3, etq 0x0C, bloque 0x33): **ACIERTO** (bloque 0x33 en vía 0). Actualiza LRU: 0x33 pasa a ser MRU.
- **0x5BAC** (conj 3, etq 0x16, bloque 0x5B): **FALLO**. Las 2 vías están ocupadas (0x33 MRU, 0x34 LRU). LRU expulsa 0x34. Carga 0x5B + prebúsqueda 0x5C. Conj 3: [0x33, 0x5B(MRU), 0x5C en vía libre... en realidad con 2 vías: expulsa 0x34, carga 0x5B; prebúsqueda 0x5C expulsa 0x33 (ahora LRU)]. Con 2 vías: conj 3 → [0x5B, 0x5C].
- **0x132A** (conj 3, etq 0x04, bloque 0x13): **FALLO**. Expulsa LRU (0x5B). Carga 0x13 + prebúsqueda 0x14 expulsa 0x5C. Conj 3: [0x13, 0x14].
- **0x5C41** (conj 0, etq 0x17, bloque 0x5C): **FALLO**. Conj 0 vacío. Carga 0x5C + prebúsqueda 0x5D.
- **0x13AF** (conj 3, etq 0x04, bloque 0x13): **ACIERTO** (bloque 0x13 en conj 3).

**Resumen b):** 5 fallos, 2 aciertos. La asociatividad 2 vías evita el conflicto en 0x3374 respecto a a).

#### c) Organización directa con buffer de prebúsqueda de 512B (FIFO)

El buffer almacena los bloques prebuscados antes de que sean referenciados. Si el bloque solicitado está en el buffer, se pasa a caché sin contar como fallo en MP (aunque sí hay movimiento buffer→caché). El buffer tiene 512/256 = **2 entradas**, política FIFO.

- **0x3345** (bloque 0x33, índice 3): No está en caché ni buffer → **FALLO**. Carga 0x33 en caché[3]. Prebúsqueda: 0x34 va al **buffer** (FIFO, posición 0).
- **0x1244** (bloque 0x12, índice 2): No está en caché ni buffer → **FALLO**. Carga 0x12 en caché[2]. Prebúsqueda: 0x13 va al **buffer** (posición 1). Buffer: [0x34, 0x13].
- **0x3374** (bloque 0x33, índice 3): **ACIERTO** en caché (0x33 sigue en índice 3).
- **0x5BAC** (bloque 0x5B, índice 3): Caché[3] tiene 0x33 (etq distinta) → no en caché. ¿Está en buffer? No. → **FALLO**. Expulsa 0x33 de caché[3], carga 0x5B. Prebúsqueda: 0x5C → buffer (FIFO expulsa 0x34, entra 0x5C). Buffer: [0x13, 0x5C].
- **0x132A** (bloque 0x13, índice 3): Caché[3] tiene 0x5B → no en caché. ¿Está en buffer? **Sí, 0x13 está en buffer**. → Se mueve 0x13 de buffer a caché[3] (expulsa 0x5B). No cuenta como fallo de MP. Buffer: [0x5C]. (Prebúsqueda del bloque siguiente al solicitado: 0x14 entra al buffer). Buffer: [0x5C, 0x14].
- **0x5C41** (bloque 0x5C, índice 4): Caché[4] vacía → no en caché. ¿Está en buffer? **Sí, 0x5C está en buffer**. → Se mueve 0x5C de buffer a caché[4]. No fallo. Prebúsqueda: 0x5D entra al buffer (expulsa 0x14). Buffer: [0x5D].
- **0x13AF** (bloque 0x13, índice 3): Caché[3] tiene 0x13 → **ACIERTO**.

**Resumen c):** 4 fallos en MP, 2 aciertos de buffer (0x132A y 0x5C41), 2 aciertos directos en caché (0x3374 y 0x13AF).

> El buffer de prebúsqueda reduce los fallos efectivos de 6 a 4, aprovechando la localidad espacial.

---

Ejercicio 3

Enunciado: Computador con memoria principal de 4MB, caché de 4KB con líneas de 512B y 2 vías (FIFO), y caché víctima de 1024B completamente asociativa (FIFO). Tablas de estado iniciales dadas. Secuencia: `0x334500`, `0x14BF00`, `0x150084`, `0x004021`, `0x0540AB`, `0x0041F1`.

Resolución:

#### Parámetros

- MP: 4MB = 2^22 B → direcciones de **22 bits**
- Caché L1: 4KB, líneas 512B, 2 vías → conjuntos = 4KB / (512B × 2) = **4 conjuntos**
  - Offset: log2(512) = 9 bits (bits 8:0)
  - Índice: log2(4) = 2 bits (bits 10:9)
  - Etiqueta: 22 - 9 - 2 = **11 bits** (bits 21:11)
- Caché víctima: 1024B / 512B = **2 entradas** completamente asociativa, FIFO

**Estado inicial caché L1 (conjuntos 0–3, 2 vías, FIFO):**

| Conj | Vía 0 (FIFO=0) | Vía 1 (FIFO=1) |
|------|----------------|----------------|
| 0    | etq 0x668      | etq 0x008      |
| 1    | etq 0x297      | etq 0x368      |
| 2    | etq 0x5FF      | etq 0x668      |
| 3    | etq 0x297      | etq 0x200      |

*Nota: FIFO=0 es el más antiguo (próximo a ser reemplazado), FIFO=1 es el más reciente.*

**Estado inicial caché víctima:**

| Entrada | Etiqueta | FIFO |
|---------|----------|------|
| 0       | 0x0A80   | 0    |
| 1       | 0x3FFF   | 1    |

#### Descomposición de direcciones

Formato: [etiqueta 11b | índice 2b | offset 9b]

| Dirección  | Etiqueta (11b) | Índice (2b) | Offset (9b) |
|-----------|---------------|------------|------------|
| 0x334500  | 0x19A (bits 21:11 de 0x334500 = 0011 0011 0100 >> 11 → 0x19A) | ... | ... |

Calculemos más cuidadosamente. 0x334500 = 0b 0011 0011 0100 0101 0000 0000 (22 bits).

- Bits 8:0 (offset) = 0x000 = 0
- Bits 10:9 (índice) = 0b10 = 2
- Bits 21:11 (etiqueta) = 0b001 1001 1010 = 0x19A → **etq=0x19A, conj=2**

| Dirección  | Hex 22b       | Etiqueta (b21:b11) | Índice (b10:b9) |
|-----------|---------------|--------------------|----------------|
| 0x334500  | 00 1100 1101 0001 0100 0000 0000... vamos a usar división directa | | |

Usamos: etiqueta = dirección >> 11, índice = (dirección >> 9) & 0x3, offset = dirección & 0x1FF.

| Dirección  | >> 9 (bloque) | Bloque % 4 (índice) | Etiqueta (>> 11) |
|-----------|--------------|---------------------|-----------------|
| 0x334500  | 0x19A2       | 0x19A2 % 4 = 2      | 0x19A2 >> 2 = 0x668 |
| 0x14BF00  | 0xA5F        | 0xA5F % 4 = 3       | 0xA5F >> 2 = 0x297  |
| 0x150084  | 0xA80        | 0xA80 % 4 = 0       | 0xA80 >> 2 = 0x2A0  |
| 0x004021  | 0x020        | 0x020 % 4 = 0       | 0x020 >> 2 = 0x008  |
| 0x0540AB  | 0x2A0        | 0x2A0 % 4 = 0       | 0x2A0 >> 2 = 0x0A8... |
| 0x0041F1  | 0x020        | 0x020 % 4 = 0       | 0x020 >> 2 = 0x008  |

*Nota: el índice y etiqueta se obtienen dividiendo entre el tamaño de bloque (512B):*  
- número de bloque = dirección ÷ 512 = dirección >> 9
- índice = número de bloque mod 4
- etiqueta = número de bloque div 4

| Dirección  | Nº bloque | Índice | Etiqueta |
|-----------|-----------|--------|----------|
| 0x334500  | 0x19A2    | 2      | 0x668    |
| 0x14BF00  | 0x0A5F    | 3      | 0x297    |
| 0x150084  | 0x0A80    | 0      | 0x2A0    |
| 0x004021  | 0x0020    | 0      | 0x008    |
| 0x0540AB  | 0x02A0    | 0      | 0x0A8    |
| 0x0041F1  | 0x0020    | 0      | 0x008    |

#### Traza de accesos

**Acceso 1 — 0x334500 (conj=2, etq=0x668):**
Conj 2 tiene: vía 0 → 0x5FF (FIFO=0), vía 1 → 0x668 (FIFO=1).
**ACIERTO** en vía 1. No hay cambios.

**Acceso 2 — 0x14BF00 (conj=3, etq=0x297):**
Conj 3 tiene: vía 0 → 0x297 (FIFO=0), vía 1 → 0x200 (FIFO=1).
**ACIERTO** en vía 0. No hay cambios.

**Acceso 3 — 0x150084 (conj=0, etq=0x2A0):**
Conj 0 tiene: vía 0 → 0x668 (FIFO=0), vía 1 → 0x008 (FIFO=1). Etq 0x2A0 no está.
¿Está en caché víctima? Víctima: {0x0A80, 0x3FFF}. No. → **FALLO**.
FIFO expulsa vía 0 (0x668) del conj 0 → va a caché víctima (expulsa 0x0A80, FIFO=0). Carga 0x2A0 en vía 0.

Estado conj 0: vía 0 → 0x2A0 (FIFO=0 nuevo), vía 1 → 0x008 (FIFO=1). *(El FIFO se reinicia: el recién llegado es FIFO=1 o el más antiguo es 0x008; con política FIFO el siguiente a expulsar es 0x008).*
Caché víctima: {0x668 (FIFO=0), 0x3FFF (FIFO=1)}.

**Acceso 4 — 0x004021 (conj=0, etq=0x008):**
Conj 0: vía 0 → 0x2A0, vía 1 → 0x008. **ACIERTO** en vía 1. No hay cambios.

**Acceso 5 — 0x0540AB (conj=0, etq=0x0A8):**

Nota: número de bloque de 0x0540AB = 0x0540AB >> 9. 0x0540AB = 344235 decimal. 344235 / 512 = 672.3 → bloque 672 = 0x2A0. Índice = 0x2A0 % 4 = 0. Etiqueta = 0x2A0 / 4 = 0xA8.

Conj 0: vía 0 → 0x2A0, vía 1 → 0x008. Etq 0xA8 no está.
¿Está en caché víctima? {0x668, 0x3FFF}. No. → **FALLO**.
FIFO expulsa el más antiguo del conj 0: depende del orden de llegada. Tras el acceso 3, llegó 0x2A0 y quedó 0x008. El más antiguo es 0x008 → se expulsa a caché víctima (expulsa 0x668, FIFO=0 de víctima). Carga 0xA8 en conj 0.

Estado conj 0: {0x2A0, 0xA8}.
Caché víctima: {0x3FFF (FIFO=0), 0x008 (FIFO=1)}.

**Acceso 6 — 0x0041F1 (conj=0, etq=0x008):**
Conj 0: {0x2A0, 0xA8}. Etq 0x008 no está.
¿Está en caché víctima? {0x3FFF, 0x008}. **SÍ**, 0x008 está en víctima.
→ Intercambio: 0x008 pasa de víctima a caché (conj 0), y el bloque expulsado de caché va a víctima. FIFO expulsa 0x2A0 de conj 0. Caché víctima: expulsa 0x3FFF (FIFO=0), entra 0x2A0.

Estado conj 0: {0xA8, 0x008}.
Caché víctima: {0x008... } ← se reformula: {0x2A0 (FIFO=0), ...}. Solo una entrada libre.

**Resumen:**

| Acceso     | Conj | Etiqueta | Resultado      | Transferencia             |
|-----------|------|----------|---------------|--------------------------|
| 0x334500  | 2    | 0x668    | **Acierto**   | —                        |
| 0x14BF00  | 3    | 0x297    | **Acierto**   | —                        |
| 0x150084  | 0    | 0x2A0    | **Fallo**     | MP→Caché; 0x668 Caché→Víctima |
| 0x004021  | 0    | 0x008    | **Acierto**   | —                        |
| 0x0540AB  | 0    | 0x0A8    | **Fallo**     | MP→Caché; 0x008 Caché→Víctima |
| 0x0041F1  | 0    | 0x008    | **Fallo→Víctima** | Víctima→Caché; 0x2A0 Caché→Víctima |

---

Ejercicio 4

Enunciado: Computador con direcciones de 32 bits, caché de 128B con bloques de 16B, emplazamiento directo, sin asignación en escritura. Variable `a` en registro. Código:

```c
int nota[128];   // nota[0] en 0x00000000
int media[128];  // media[0] a continuación de nota[127]

for (i=0; i<128; i++) {
    if (i>7 && i<64) {
        nota[i] = media[i]/2;
    } else {
        a = nota[i] * media[i];
    }
}
```

- a) ¿Cuántos fallos de caché se producen?
- b) Dos optimizaciones independientes que mejoren el resultado.
- c) Tiempo de acceso a todas las referencias con tiempo de caché = 1ns, palabra en MP = 50ns, bloque MP→caché = 200ns.
- d) Con caché L2 de 1MB, asociatividad 4, bloques de 16B, write-allocate, write-back y tiempo de transferencia bloque L2→L1 = 15ns.

Resolución:

#### Parámetros de la caché

- Tamaño: 128 B, bloques de 16 B → **8 bloques** en caché
- Offset: log2(16) = 4 bits; Índice: log2(8) = 3 bits; Etiqueta: 32-4-3 = 25 bits
- Cada `int` = 4 bytes → 16B/4 = **4 enteros por bloque**
- `nota[128]`: 128×4 = 512 B = 32 bloques (índices 0x0000... a 0x01F...)
- `media[128]`: empieza en 0x00000200, 512 B = 32 bloques

**Mapeo de bloques:**
- nota[i] está en bloque (i/4), índice en caché = (i/4) % 8
- media[i] está en bloque (i/4 + 32) → índice en caché = (i/4 + 32) % 8 = (i/4) % 8

⚠️ `nota[i]` y `media[i]` **mapean al mismo índice de caché** → conflicto directo.

#### a) Conteo de fallos

El bucle recorre i de 0 a 127. Se distinguen dos ramas:

**Rama `else` (i∈[0,7] ∪ [64,127]):** se leen `nota[i]` y `media[i]`.

Por cada grupo de 4 elementos (mismo bloque):
- Primer elemento del bloque: fallo en `nota[i]` → carga bloque de nota. Luego acceso a `media[i]` → fallo (mismo índice, etiqueta distinta) → carga bloque de media, expulsa nota. Los 3 elementos siguientes del bloque: `nota[i+1]` → fallo nuevamente (nota fue expulsada), etc.
- → **Thrashing**: 2 fallos por cada elemento (nota + media) = 8 fallos por cada 4 elementos.

Elementos en rama else: i∈[0..7] = 8 elementos, i∈[64..127] = 64 elementos → total **72 elementos**.

Fallos rama else: 72 × 2 = **144 fallos** (todos en lectura).

**Rama `if` (i∈[8,63]):** se lee `media[i]` y se escribe `nota[i]` (sin asignación).

- Lectura de `media[i]`: mismo análisis de conflicto. Los bloques de media mapean al mismo índice que nota. Pero nota no se lee aquí, solo se escribe sin asignación. Por tanto, en la rama `if` solo se accede a caché para leer media.
- Fallo en `media[i]`: primer acceso a cada bloque de media → fallo. Bloques de media accedidos: i∈[8..63] = 56 elementos = 14 bloques → **14 fallos de lectura**.
- Escritura de `nota[i]`: sin asignación en escritura → **no entra en caché** → 0 fallos de escritura (se escribe directo a MP).

Fallos rama if: **14 fallos de lectura + 0 fallos de escritura**.

**Total fallos:**
- Lectura: 144 + 14 = **158 fallos**
- Escritura: 0 fallos (sin asignación)

> **Total: 158 fallos de caché.**

#### b) Dos optimizaciones

**Optimización 1 — Fusión de arrays (array merging):**

Intercalar nota y media en una misma estructura:
```c
struct { int nota; int media; } datos[128];
```
Ahora `datos[i].nota` y `datos[i].media` están en el mismo bloque (8 bytes por par, 2 pares por bloque). Cada bloque contiene 2 pares. El índice de caché es el mismo para el par, pero al acceder a ambos campos del mismo struct se produce un solo fallo por cada par de elementos.

- Fallos totales ≈ (128 pares) / (2 pares/bloque) = **64 fallos** (aproximadamente, dependiendo de la distribución). Reducción drástica del thrashing.

**Optimización 2 — Padding para evitar aliasing:**

Sin modificar el algoritmo, añadir padding a `nota` para que `media` no mapee al mismo índice:

```c
int nota[128 + 32]; // 32 enteros extra = 128B de padding
int media[128];
```

Ahora media empieza en 0x00000280 (offset = 512 + 128 = 640 bytes), lo que desplaza su mapeo en caché y evita el conflicto con nota.

Con padding correcto, `nota[i]` y `media[i]` mapean a **índices distintos** → no hay thrashing. Cada bloque se carga una sola vez:
- nota: 32 bloques de 4 elementos → 32 fallos de lectura (rama else) + lecturas de media en rama else: 18 bloques → 18 fallos.
- media en rama if: 14 bloques → 14 fallos.
- Escrituras en nota (rama if): sin asignación → 0.

Fallos con padding ≈ **32 + 18 + 14 = 64 fallos** (una mejora de ~2.5×).

#### c) Tiempo total de acceso

**Referencias al programa:**

- Total de accesos (lecturas + escrituras): 128 iteraciones.
  - Rama else (72 iter.): 2 loads + 0 stores = 144 accesos de lectura.
  - Rama if (56 iter.): 1 load + 1 store = 56 + 56 = 112 accesos.
  - Total: **144 + 56 + 56 = 256 accesos** (pero hay que ver cuántos son aciertos y cuántos fallos).

Total accesos: 128×(nota + media) aproximado:
- Lecturas: 72×2 + 56×1 = 200 lecturas
- Escrituras: 56 escrituras (sin asignación, van directo a MP)

Aciertos en lectura: 200 - 158 = 42 aciertos.
Fallos en lectura: 158 fallos (cada uno trae un bloque → 200ns de penalización).
Escrituras: 56 (todas a MP directamente → 50ns cada una, sin pasar por caché).

**Tiempo total:**
- Aciertos en lectura: 42 × 1 ns = 42 ns
- Fallos en lectura: 158 × (1 + 200) ns = 158 × 201 = 31758 ns
- Escrituras (a MP): 56 × 50 ns = 2800 ns

> **Tiempo total ≈ 42 + 31758 + 2800 = 34600 ns ≈ 34.6 µs**

#### d) Con caché L2 (1MB, 4 vías, bloques 16B, write-allocate, write-back, L2→L1 = 15ns)

La caché L2 es lo suficientemente grande (1MB) para contener todos los datos del programa (nota + media = 1KB). Con L2, los fallos de L1 que eran penalizados con 200ns (acceso a MP) ahora se atienden en L2 con solo **15ns**.

- Los 158 fallos de L1 se sirven desde L2 (suponiendo que L2 siempre acierta, dado su gran tamaño respecto a los datos).
- Escrituras: con write-allocate y write-back en L2, las escrituras de la rama if se resuelven en L1/L2 sin ir a MP.

**Tiempo total con L2:**
- Aciertos en L1 (lectura): 42 × 1 ns = 42 ns
- Fallos en L1 (servidos por L2): 158 × (1 + 15) ns = 158 × 16 = 2528 ns
- Escrituras con write-allocate: si hay fallo en L1, se trae bloque a L1 (15ns) y luego se escribe (write-back, diferido). Escrituras en rama if = 56. Con caché L2 grande, los bloques de nota están en L1 tras el primer acceso → la mayoría de escrituras son aciertos en L1 (0 penalización extra).

> **Tiempo total con L2 ≈ 42 + 2528 + escrituras en L1 (acierto) = ~2600 ns ≈ 2.6 µs**
>
> La mejora respecto al caso sin L2 es de aproximadamente **13×** (de 34.6µs a 2.6µs).

---

Ejercicio 5

Enunciado: Computador con MP de 256KB, caché de 128B con bloques de 16B. Código:

```c
int A[16][16];
int B[32];
int C[16][16];

for (i=0; i < 16; i++)
    C[0][i] = A[0][i] + B[4];
```

A se almacena en `0x10000`, B y C a continuación. Variables `i`, `j` y punteros en registros.

- a) Fallos con emplazamiento directo y asignación en escritura.
- b) Fallos con asociativa por conjuntos de 2 vías, FIFO.
- c) ¿Se puede reducir el número de errores sin modificar el programa ni el tamaño de la caché?
- d) Tiempo de acceso con t_cache = 2ns y penalización por fallo = 150ns.

Resolución:

#### Parámetros de la caché

- MP: 256KB = 2^18 B → direcciones de 18 bits
- Caché: 128B, bloques de 16B → **8 bloques**
- Offset: 4 bits; Índice: 3 bits; Etiqueta: 11 bits
- Cada `int` = 4B → 4 enteros por bloque de 16B

**Tamaños y ubicaciones:**
- A[16][16]: 16×16×4 = 1024B = **64 bloques** → en 0x10000 a 0x103FF
- B[32]: 32×4 = 128B = **8 bloques** → en 0x10400 a 0x1047F
- C[16][16]: 1024B = **64 bloques** → en 0x10480 a 0x1087F

**Solo se accede a:**
- A[0][0..15]: bloque base de A es 0x10000/16 = 0x1000. A[0][i] ∈ bloques 0x1000 y 0x1001 (4 primeros en bloque 0x1000, siguientes 4 en 0x1001, etc. → 4 bloques para 16 elementos).
- B[4]: B empieza en 0x10400. B[4] está en 0x10410. Bloque: 0x10410/16 = 0x1041. Solo 1 bloque de B se usa.
- C[0][0..15]: C empieza en 0x10480. C[0][i] ∈ 4 bloques.

**Mapeo de índices en caché (emplazamiento directo, índice = nº bloque % 8):**

| Array | Bloque MP | Índice caché |
|-------|-----------|-------------|
| A[0][0..3]  | 0x1000 | 0x1000 % 8 = **0** |
| A[0][4..7]  | 0x1001 | **1** |
| A[0][8..11] | 0x1002 | **2** |
| A[0][12..15]| 0x1003 | **3** |
| B[4] (B[4..7]) | 0x1041 | 0x1041 % 8 = **1** |
| C[0][0..3]  | 0x1048 | 0x1048 % 8 = **0** |
| C[0][4..7]  | 0x1049 | **1** |
| C[0][8..11] | 0x104A | **2** |
| C[0][12..15]| 0x104B | **3** |

**Conflictos detectados:**
- Índice 0: A[0][0..3] y C[0][0..3] → conflicto.
- Índice 1: A[0][4..7], B[4..7] y C[0][4..7] → triple conflicto.
- Índice 2: A[0][8..11] y C[0][8..11] → conflicto.
- Índice 3: A[0][12..15] y C[0][12..15] → conflicto.

#### a) Emplazamiento directo con asignación en escritura

El bucle para i=0..15:
- Lee B[4] (misma posición en todas las iteraciones).
- Lee A[0][i].
- Escribe C[0][i] (con asignación → trae bloque si no está).

**i=0 (A[0][0], B[4], C[0][0]):**
- Load A[0][0]: bloque 0x1000 → índice 0. **Fallo** (caché vacía). Carga bloque A.
- Load B[4]: bloque 0x1041 → índice 1. **Fallo**. Carga bloque B.
- Store C[0][0]: bloque 0x1048 → índice 0. Con asignación: **Fallo** (índice 0 tenía A, etiqueta distinta). Carga bloque C, expulsa A.

**i=1 (A[0][1], B[4], C[0][1]):** todos en mismos bloques que i=0.
- Load A[0][1]: índice 0, ahora hay C → **Fallo** (etiqueta A ≠ etiqueta C). Carga A, expulsa C.
- Load B[4]: índice 1. B[4] todavía está (no fue expulsado) → **Acierto**.
- Store C[0][1]: índice 0, hay A → **Fallo** con asignación. Carga C, expulsa A.

**i=2, i=3:** mismo patrón que i=1. 2 fallos por iteración (A y C en conflicto en índice 0, B acierto).

**i=4 (A[0][4], B[4], C[0][4]):** cambia bloque (índice 1 para A y C).
- Load A[0][4]: bloque 0x1001 → índice 1. Índice 1 tiene B[4] → **Fallo** (etiqueta distinta). Carga A, expulsa B.
- Load B[4]: bloque 0x1041 → índice 1. Ahora hay A → **Fallo**. Carga B, expulsa A.
- Store C[0][4]: bloque 0x1049 → índice 1. Hay B → **Fallo**. Carga C, expulsa B.

**i=5, 6, 7:** A (índice 1) → Fallo (C está), luego B → Fallo, luego C → Fallo. 3 fallos por iteración para i∈[5,7]... pero espera: en i=5 tras haber cargado C[0][4] en índice 1:
- Load A[0][5]: índice 1, tiene C → Fallo. Carga A.
- Load B[4]: índice 1, tiene A → Fallo. Carga B.
- Store C[0][5]: índice 1, tiene B → Fallo. Carga C.

Efectivamente 3 fallos/iter para i∈[4..7].

**i=8..11:** índice 2. A y C en conflicto en índice 2, B no interviene.
- Load A[0][8]: **Fallo**. Carga A.
- Load B[4]: índice 1 (B vuelve a estar en índice 1 si no fue expulsado). Al final de i=7, índice 1 tiene C[0][4..7]. → **Fallo** en B para i=8. Carga B, expulsa C de índice 1.
- Store C[0][8]: índice 2, hay A → Fallo. Carga C, expulsa A.

i=9,10,11: A → Fallo (C en índ 2), B → Acierto (B no se mueve), C → Fallo. 2 fallos/iter.

**i=12..15:** índice 3. A y C en conflicto. B → verificar.
- i=12: Load A → Fallo. Load B → Acierto (B sigue en índice 1 tras i=8). Store C → Fallo. 2 fallos.
- i=13,14,15: A→Fallo, B→Acierto, C→Fallo. 2 fallos/iter.

**Recuento total de fallos (emplazamiento directo, write-allocate):**

| Rango i | Fallos/iter | Iters | Total fallos |
|---------|------------|-------|-------------|
| 0       | 3 (A,B,C)  | 1     | 3            |
| 1–3     | 2 (A,C; B acierto) | 3 | 6          |
| 4       | 3 (A,B,C)  | 1     | 3            |
| 5–7     | 3 (A,B,C)  | 3     | 9            |
| 8       | 3 (A,B,C)  | 1     | 3            |
| 9–11    | 2 (A,C; B acierto) | 3 | 6          |
| 12      | 2 (A,C; B acierto) | 1 | 2           |
| 13–15   | 2 (A,C; B acierto) | 3 | 6           |

> **Total fallos ≈ 3+6+3+9+3+6+2+6 = 38 fallos** (lectura + escritura combinados con asignación).

#### b) Asociativa por conjuntos de 2 vías, FIFO

Con 2 vías: 4 conjuntos, índice = 2 bits (bits 5:4 del nº bloque), etiqueta = resto.

- Cada conjunto tiene 2 vías → pueden coexistir 2 bloques con el mismo índice.
- Índice 1 (anterior): ahora pueden convivir A y B sin expulsarse, pero cuando llega C → 3 arrays en 2 vías, sigue el thrashing parcial.

Para los índices donde solo compiten 2 arrays (índices 0, 2, 3: solo A y C):
- A y C caben en las 2 vías → **no hay thrashing**. Un solo fallo al cargar cada bloque.

Para índice 1 (A, B, C compiten): B[4] es la misma posición siempre y se reutiliza en 4 iteraciones consecutivas. Con FIFO y 2 vías:
- Primer acceso al conjunto: carga A, luego B (llena las 2 vías). Al acceder a C (3er array), FIFO expulsa A. Siguiente lectura de A → fallo.

**Fallos con 2 vías (análisis simplificado):**

- Bloques de A sin conflicto (índices 0, 2, 3): 3 bloques, 1 fallo cada uno → 3 fallos de lectura (acierto en elementos 2,3,4 de cada bloque).
- Bloque de A en índice 1 (A[0][4..7]): thrashing A/B/C → ~4 fallos.
- B[4]: 1 fallo inicial, luego acierto en las iteraciones de su bloque (salvo expulsiones).
- Bloques de C (write-allocate): si A y C no colisionan (índices 0,2,3 con 2 vías), 1 fallo por bloque → 3 fallos.

> **Total aproximado con 2 vías ≈ 15–20 fallos**, notablemente mejor que los 38 del emplazamiento directo para los índices sin triple conflicto.

#### c) ¿Se puede reducir errores sin modificar programa ni tamaño de caché?

**Sí**, mediante **cambio del tamaño de bloque** o **aumento de la asociatividad** (ambas son parámetros de la caché, no del programa ni del tamaño). Concretamente:

- **Aumentar la asociatividad** a 2 vías (sin cambiar tamaño total) permite que A y C coexistan en los conjuntos que solo tienen 2 arrays en conflicto, eliminando el thrashing en esos conjuntos.
- Alternativamente, si se puede cambiar el **mapeo de direcciones** mediante padding del SO o del enlazador, sin tocar el programa fuente, los arrays se ubicarían en zonas que no colisionen.

Sin ninguna modificación hardware ni de parámetros de caché, con emplazamiento directo y el mismo programa, **no es posible reducir los fallos por conflicto** causados por el aliasing entre A, B y C.

#### d) Tiempo de acceso (configuración apartado a)

- t_caché = 2 ns, penalización por fallo = 150 ns
- Total accesos del bucle: 16 iteraciones × 3 accesos = 48 accesos
- Fallos: 38; Aciertos: 48 - 38 = 10

**Tiempo total:**
- Aciertos: 10 × 2 ns = 20 ns
- Fallos: 38 × (2 + 150) ns = 38 × 152 = 5776 ns

> **Tiempo total = 20 + 5776 = 5796 ns ≈ 5.8 µs**

---

### Problemas Adicionales

---

Ejercicio A1

Enunciado: Computador con memoria física de 32KB y caché de 512B con bloques de 128B, emplazamiento directo.

- a) Formato de dirección para MP.
- b) Rango de direcciones físicas de cada bloque de caché según la tabla dada.
- c) Número de aciertos para las cadenas de referencias: 0x2080–0x209F, 0x2880–0x289F, 0x03F0–0x0410.

| Etiqueta | Bloque |
|----------|--------|
| 0x35     | 0      |
| 0x10     | 1      |
| 0x10     | 2      |
| 0x08     | 3      |

Resolución:

#### a) Formato de dirección

- MP: 32KB = 2^15 B → **15 bits** de dirección
- Caché: 512B, bloques de 128B → 512/128 = **4 bloques**
- Offset de bloque: log2(128) = **7 bits** (bits 6:0)
- Índice de bloque en caché: log2(4) = **2 bits** (bits 8:7)
- Etiqueta: 15 - 7 - 2 = **6 bits** (bits 14:9)

**Formato:** `[etiqueta: 6 bits | índice: 2 bits | offset: 7 bits]`

#### b) Rango de direcciones de cada bloque de caché

Un bloque de caché con índice `i` y etiqueta `t` corresponde al bloque de MP número `t×4 + i`. La dirección base es `(t×4 + i) × 128`.

| Bloque caché | Etiqueta | Nº bloque MP | Dir. base | Rango de direcciones |
|-------------|----------|-------------|-----------|----------------------|
| 0           | 0x35=53  | 53×4+0=212  | 212×128=27136=0x6A00 | 0x6A00–0x6A7F |
| 1           | 0x10=16  | 16×4+1=65   | 65×128=8320=0x2080   | 0x2080–0x20FF |
| 2           | 0x10=16  | 16×4+2=66   | 66×128=8448=0x2100   | 0x2100–0x217F |
| 3           | 0x08=8   | 8×4+3=35    | 35×128=4480=0x1180   | 0x1180–0x11FF |

#### c) Número de aciertos

**Serie 1: 0x2080 a 0x209F** (32 bytes = 8 palabras de 4B en el bloque 0x2080–0x20FF)

El bloque 0x2080–0x20FF es el bloque MP número 65 → índice caché 1, etiqueta 0x10. Según la tabla, **bloque caché 1 tiene etiqueta 0x10** → coincide.

- Todos los accesos de 0x2080 a 0x209F están dentro del bloque caché 1 → **acierto en todos**. El primer acceso puede ser fallo (bloque ya estaba cargado), pero el enunciado dice que la tabla *ya contiene* esos valores → asumimos que el bloque está en caché.

Número de bytes: 0x209F - 0x2080 + 1 = 32B. Si se accede byte a byte: **32 aciertos**. Si se accede por palabras de 4B: **8 aciertos**.

**Serie 2: 0x2880 a 0x289F** (32 bytes)

0x2880 = 10368 decimal. Bloque MP = 10368/128 = 81. Índice = 81%4 = 1. Etiqueta = 81/4 = 20 = 0x14. Bloque caché 1 tiene etiqueta **0x10 ≠ 0x14** → **fallo en el primer acceso**. El resto (una vez cargado el bloque): **aciertos** dentro del mismo bloque.

Si se cuenta el primer acceso como fallo: 1 fallo + 31 aciertos (byte) o 1 fallo + 7 aciertos (palabras 4B).

**Serie 3: 0x03F0 a 0x0410** (33 bytes, cruza un límite de bloque)

- 0x03F0–0x03FF: pertenece al bloque MP 0x03F0/128 = 7. Índice=7%4=3. Etiqueta=7/4=1=0x01. Caché bloque 3 tiene etiqueta **0x08 ≠ 0x01** → **Fallo**.
- 0x0400–0x0410: pertenece al bloque MP 0x0400/128 = 8. Índice=8%4=0. Etiqueta=8/4=2=0x02. Caché bloque 0 tiene etiqueta **0x35 ≠ 0x02** → **Fallo**.

→ **0 aciertos** en la serie 3 (2 fallos).

> **Resumen de aciertos:** Serie 1 = 8 (todas palabras), Serie 2 = 7 (tras el primer fallo), Serie 3 = 0. **Total aciertos ≈ 15** (dependiendo de la granularidad de acceso).

---

Ejercicio A2

Enunciado: MP de 32MB, caché de 2KB con líneas de 256B, prebúsqueda bajo fallo, reemplazo FIFO. Secuencia: `0x00023FA`, `0x00014A2`, `0x0003F02`, `0x00040B1`, `0x0005572`, `0x00023AA`.

- a) Emplazamiento directo.
- b) Asociativo por conjuntos de 2 vías.

Resolución:

#### Parámetros

- MP: 32MB = 2^25 B → 25 bits de dirección
- Caché: 2KB, líneas 256B → 8 bloques
- Offset: 8 bits; **Directo:** índice 3 bits, etiqueta 14 bits
- **2 vías:** 4 conjuntos, índice 2 bits, etiqueta 15 bits
- Prebúsqueda: en fallo, se cargan bloque actual + bloque siguiente

Nº de bloque = dirección >> 8:

| Dirección   | Nº bloque | Índice (directo) | Etiqueta (directo) | Índice (2v) | Etiqueta (2v) |
|------------|-----------|------------------|--------------------|------------|--------------|
| 0x00023FA  | 0x0023    | 3                | 0x0004             | 3          | 0x0008       |
| 0x00014A2  | 0x0014    | 4                | 0x0002             | 0          | 0x0005       |
| 0x0003F02  | 0x003F    | 7                | 0x0007             | 3          | 0x000F       |
| 0x00040B1  | 0x0040    | 0                | 0x0008             | 0          | 0x0010       |
| 0x0005572  | 0x0055    | 5                | 0x000A             | 1          | 0x0015       |
| 0x00023AA  | 0x0023    | 3                | 0x0004             | 3          | 0x0008       |

#### a) Emplazamiento directo

- **0x00023FA** (bloque 0x23, índice 3, etq 0x04): **F**. Carga 0x23 en índice 3 + 0x24 en índice 4 (prebúsqueda).
- **0x00014A2** (bloque 0x14, índice 4, etq 0x02): Índice 4 tiene bloque 0x24 (etq 0x04) ≠ 0x02 → **F**. Carga 0x14 en índice 4 (expulsa 0x24) + 0x15 en índice 5.
- **0x0003F02** (bloque 0x3F, índice 7, etq 0x07): **F**. Carga 0x3F en índice 7 + 0x40 en índice 0.
- **0x00040B1** (bloque 0x40, índice 0, etq 0x08): Índice 0 tiene 0x40 (prebuscado) → **A**.
- **0x0005572** (bloque 0x55, índice 5, etq 0x0A): Índice 5 tiene 0x15 (etq 0x02) ≠ 0x0A → **F**. Carga 0x55 en índice 5 + 0x56 en índice 6.
- **0x00023AA** (bloque 0x23, índice 3, etq 0x04): Índice 3 tiene 0x23 (etq 0x04) → **A**.

**Resumen a):** F, F, F, A, F, A → 4 fallos, 2 aciertos, 4 prebúsquedas.

#### b) Asociativo por conjuntos de 2 vías, FIFO

- **0x00023FA** (conj 3, etq 0x08): **F**. Carga 0x23 en conj 3, vía 0. Prebúsqueda 0x24 en conj 0 (0x24%4=0), vía 0.
- **0x00014A2** (conj 0, etq 0x05): Conj 0 tiene 0x24 (etq 0x09). Etq 0x05 ≠ 0x09 → **F**. Carga 0x14 en conj 0, vía 1 (2 vías). Prebúsqueda 0x15 en conj 3, vía 1.
- **0x0003F02** (conj 3, etq 0x0F): Conj 3 tiene 0x23 (etq 0x08) y 0x15 (etq 0x05). Etq 0x0F no está → **F**. FIFO expulsa 0x23 (el más antiguo). Carga 0x3F en conj 3. Prebúsqueda 0x40 (conj 0, etq 0x10): conj 0 tiene 0x24 y 0x14, FIFO expulsa 0x24.
- **0x00040B1** (conj 0, etq 0x10): Conj 0 tiene 0x14 (etq 0x05) y 0x40 (etq 0x10, prebuscado). **A**.
- **0x0005572** (conj 1, etq 0x15): Conj 1 vacío → **F**. Carga 0x55 en conj 1, vía 0. Prebúsqueda 0x56 en conj 1, vía 1.
- **0x00023AA** (conj 3, etq 0x08): Conj 3 tiene 0x15 (etq 0x05) y 0x3F (etq 0x0F). Etq 0x08 no está → **F**.

**Resumen b):** F, F, F, A, F, F → 5 fallos, 1 acierto.

> Curiosamente, la organización directa resulta mejor aquí (4 fallos vs 5) debido a que la prebúsqueda en directo acertó en el acceso 4, mientras que con 2 vías el bloque 0x23 fue expulsado antes de ser referenciado de nuevo.

---

Ejercicio A3

Enunciado: Caché de 256B con bloques de 16B.
- a) Organización directa: analizar secuencia 0xA01, 0xB1F, 0x70A, 0x60F, 0xA70, 0xB11, 0xA7A, 0xA0B, 0x67A, 0xA7F, 0x071, 0x67F. Indicar fallos y reemplazamientos.
- b) Asociativa 2 vías, LRU: misma secuencia. Indicar reemplazamientos.

Resolución:

#### Parámetros

- Caché: 256B, bloques 16B → 16 bloques
- Offset: 4 bits; Índice: 4 bits; Etiqueta: bits superiores
- Dirección: formato [etiqueta | índice (4b) | offset (4b)]

Índice = bits [7:4] de la dirección, Etiqueta = bits [11:8] y superiores.

Nº bloque = dirección >> 4, índice = nº bloque % 16:

| Dirección | Nº bloque | Índice | Etiqueta |
|-----------|-----------|--------|----------|
| 0xA01     | 0xA0      | 0      | 0xA      |
| 0xB1F     | 0xB1      | 1      | 0xB      |
| 0x70A     | 0x70      | 0      | 0x7      |
| 0x60F     | 0x60      | 0      | 0x6      |
| 0xA70     | 0xA7      | 7      | 0xA      |
| 0xB11     | 0xB1      | 1      | 0xB      |
| 0xA7A     | 0xA7      | 7      | 0xA      |
| 0xA0B     | 0xA0      | 0      | 0xA      |
| 0x67A     | 0x67      | 7      | 0x6      |
| 0xA7F     | 0xA7      | 7      | 0xA      |
| 0x071     | 0x07      | 7      | 0x0      |
| 0x67F     | 0x67      | 7      | 0x6      |

#### a) Organización directa

| # | Dir.  | Índice | Etiqueta | Caché[índice] | Resultado        |
|---|-------|--------|----------|---------------|-----------------|
| 1 | 0xA01 | 0      | 0xA      | vacío         | **Fallo**        |
| 2 | 0xB1F | 1      | 0xB      | vacío         | **Fallo**        |
| 3 | 0x70A | 0      | 0x7      | 0xA           | **Fallo** (reemplazo) |
| 4 | 0x60F | 0      | 0x6      | 0x7           | **Fallo** (reemplazo) |
| 5 | 0xA70 | 7      | 0xA      | vacío         | **Fallo**        |
| 6 | 0xB11 | 1      | 0xB      | 0xB           | **Acierto**      |
| 7 | 0xA7A | 7      | 0xA      | 0xA           | **Acierto**      |
| 8 | 0xA0B | 0      | 0xA      | 0x6           | **Fallo** (reemplazo) |
| 9 | 0x67A | 7      | 0x6      | 0xA           | **Fallo** (reemplazo) |
|10 | 0xA7F | 7      | 0xA      | 0x6           | **Fallo** (reemplazo) |
|11 | 0x071 | 7      | 0x0      | 0xA           | **Fallo** (reemplazo) |
|12 | 0x67F | 7      | 0x6      | 0x0           | **Fallo** (reemplazo) |

**Resumen a):** 10 fallos (8 con reemplazamiento), 2 aciertos.

Fallos con reemplazamiento: accesos 3, 4, 8, 9, 10, 11, 12 (7 reemplazamientos en total, pues el acceso 5 no reemplaza ya que el índice 7 estaba vacío).

#### b) Asociativa 2 vías, LRU

Con 2 vías: 8 conjuntos, índice = 3 bits (bits 6:4), etiqueta = bits [11:7].

Recalculamos índice = nº_bloque % 8:

| # | Dir.  | Bloque | Índice(mod 8) | Etiqueta |
|---|-------|--------|--------------|----------|
| 1 | 0xA01 | 0xA0   | 0            | 0x14     |
| 2 | 0xB1F | 0xB1   | 1            | 0x16     |
| 3 | 0x70A | 0x70   | 0            | 0x0E     |
| 4 | 0x60F | 0x60   | 0            | 0x0C     |
| 5 | 0xA70 | 0xA7   | 7            | 0x14     |
| 6 | 0xB11 | 0xB1   | 1            | 0x16     |
| 7 | 0xA7A | 0xA7   | 7            | 0x14     |
| 8 | 0xA0B | 0xA0   | 0            | 0x14     |
| 9 | 0x67A | 0x67   | 7            | 0x0C     |
|10 | 0xA7F | 0xA7   | 7            | 0x14     |
|11 | 0x071 | 0x07   | 7            | 0x00     |
|12 | 0x67F | 0x67   | 7            | 0x0C     |

| # | Conj | Etiqueta | Conj estado (vía0, vía1) LRU→MRU | Resultado |
|---|------|----------|----------------------------------|-----------|
| 1 | 0    | 0x14     | vacío → (0x14)                   | **Fallo** |
| 2 | 1    | 0x16     | vacío → (0x16)                   | **Fallo** |
| 3 | 0    | 0x0E     | (0x14) → (0x14, 0x0E)           | **Fallo** |
| 4 | 0    | 0x0C     | (0x14, 0x0E) → LRU=0x14 reemplaza → (0x0E, 0x0C) | **Fallo** (reemplazo) |
| 5 | 7    | 0x14     | vacío → (0x14)                   | **Fallo** |
| 6 | 1    | 0x16     | (0x16) → (0x16) MRU             | **Acierto** |
| 7 | 7    | 0x14     | (0x14) → MRU                    | **Acierto** |
| 8 | 0    | 0x14     | (0x0E, 0x0C) → LRU=0x0E reemplaza → (0x0C, 0x14) | **Fallo** (reemplazo) |
| 9 | 7    | 0x0C     | (0x14) → (0x14, 0x0C)           | **Fallo** |
|10 | 7    | 0x14     | (0x14, 0x0C) → MRU=0x14        | **Acierto** |
|11 | 7    | 0x00     | (0x0C, 0x14) → LRU=0x0C reemplaza → (0x14, 0x00) | **Fallo** (reemplazo) |
|12 | 7    | 0x0C     | (0x14, 0x00) → LRU=0x14 reemplaza → (0x00, 0x0C) | **Fallo** (reemplazo) |

**Resumen b):** 9 fallos (5 con reemplazamiento), 3 aciertos.

> La asociatividad de 2 vías reduce los reemplazamientos de 7 a 5, mejorando la tasa de aciertos de 2/12 a 3/12.

---

Ejercicio A4

Enunciado: MP de 128MB, caché de 1MB con líneas de 256B (emplazamiento directo) y caché víctima de 1KB completamente asociativa, FIFO. Caché inicialmente vacía. Secuencia: `0x0334500`, `0x014BF00`, `0x4134501`, `0x124BF00`, `0x03345FE`, `0x34345AC`, `0x214BFAA`, `0x014BF54`.

Resolución:

#### Parámetros

- MP: 128MB = 2^27 B → 27 bits de dirección
- Caché L1: 1MB, líneas 256B → 4096 bloques, emplazamiento directo
  - Offset: 8 bits; Índice: 12 bits; Etiqueta: 7 bits
- Caché víctima: 1KB / 256B = **4 entradas**, completamente asociativa, FIFO

Nº bloque = dirección >> 8; índice = nº bloque % 4096; etiqueta = nº bloque >> 12

| Dirección   | Nº bloque (>>8) | Índice (%4096) | Etiqueta (>>12) |
|------------|----------------|----------------|----------------|
| 0x0334500  | 0x03345         | 0x0345         | 0x3            |
| 0x014BF00  | 0x014BF         | 0x0BBF         | 0x1            |
| 0x4134501  | 0x41345         | 0x0345         | 0x41           |
| 0x124BF00  | 0x124BF         | 0x0BBF         | 0x12           |
| 0x03345FE  | 0x03345         | 0x0345         | 0x3            |
| 0x34345AC  | 0x34345         | 0x0345         | 0x34           |
| 0x214BFAA  | 0x214BF         | 0x0BBF         | 0x21           |
| 0x014BF54  | 0x014BF         | 0x0BBF         | 0x1            |

Caché víctima (estado inicial): vacía.

#### Traza de accesos

**1 — 0x0334500 (índice=0x0345, etq=0x3):** Caché vacía → **Fallo**. Carga bloque en índice 0x0345. Víctima: vacía.

**2 — 0x014BF00 (índice=0x0BBF, etq=0x1):** Caché[0x0BBF] vacía → **Fallo**. Carga. Víctima: sin cambios.

**3 — 0x4134501 (índice=0x0345, etq=0x41):** Caché[0x0345] tiene etq=0x3 ≠ 0x41. ¿Está en víctima? No → **Fallo**. Expulsa etq=0x3 a víctima (FIFO pos.0). Carga etq=0x41. Víctima: {etq=0x3, pos=0}.

**4 — 0x124BF00 (índice=0x0BBF, etq=0x12):** Caché[0x0BBF] tiene etq=0x1 ≠ 0x12. ¿En víctima? {etq=0x3} No → **Fallo**. Expulsa etq=0x1 a víctima (pos=1). Carga etq=0x12. Víctima: {0x3(pos0), 0x1(pos1)}.

**5 — 0x03345FE (índice=0x0345, etq=0x3):** Caché[0x0345] tiene etq=0x41 ≠ 0x3. ¿En víctima? **Sí, etq=0x3 en pos=0**. → Intercambio: etq=0x3 pasa a caché, etq=0x41 va a víctima. Víctima expulsa pos=0 (FIFO, el más antiguo es 0x3... pero 0x3 es el que sale). Víctima tras intercambio: {0x1(pos0→pos1 pasa a ser el más antiguo ahora), 0x41 entra}. Víctima: {0x1(pos0), 0x41(pos1)}.

**6 — 0x34345AC (índice=0x0345, etq=0x34):** Caché[0x0345] tiene etq=0x3. ¿En víctima? {0x1, 0x41} No etq=0x34 → **Fallo**. Expulsa etq=0x3 a víctima (FIFO expulsa el más antiguo: 0x1). Víctima: {0x41(pos0), 0x3(pos1)}. Caché[0x0345] ← etq=0x34.

**7 — 0x214BFAA (índice=0x0BBF, etq=0x21):** Caché[0x0BBF] tiene etq=0x12 ≠ 0x21. ¿En víctima? {0x41, 0x3} No → **Fallo**. Expulsa 0x12 a víctima (FIFO expulsa 0x41). Víctima: {0x3(pos0), 0x12(pos1)}.

**8 — 0x014BF54 (índice=0x0BBF, etq=0x1):** Caché[0x0BBF] tiene etq=0x21 ≠ 0x1. ¿En víctima? {0x3, 0x12} No etq=0x1 → **Fallo**. Expulsa 0x21 a víctima (FIFO expulsa 0x3). Víctima: {0x12(pos0), 0x21(pos1)}.

**Resumen:**

| Acceso       | Resultado           | Acción                                      |
|-------------|---------------------|---------------------------------------------|
| 0x0334500   | **Fallo**           | MP → Caché[0x0345]                          |
| 0x014BF00   | **Fallo**           | MP → Caché[0x0BBF]                          |
| 0x4134501   | **Fallo**           | Caché[0x0345](etq3) → Víctima; MP → Caché  |
| 0x124BF00   | **Fallo**           | Caché[0x0BBF](etq1) → Víctima; MP → Caché  |
| 0x03345FE   | **Fallo→Víctima** ✓ | Víctima(etq3) → Caché; Caché(etq41) → Víctima |
| 0x34345AC   | **Fallo**           | Caché[0x0345](etq3) → Víctima; MP → Caché  |
| 0x214BFAA   | **Fallo**           | Caché[0x0BBF](etq12) → Víctima; MP → Caché |
| 0x014BF54   | **Fallo**           | Caché[0x0BBF](etq21) → Víctima; MP → Caché |

7 fallos a MP, 1 acierto en caché víctima (acceso 5).

---

Ejercicio A5

Enunciado: MP de 4MB, caché de 2KB con líneas de 512B, emplazamiento directo, asignación en escritura. Código:

```c
for (i=0; i<1024; i++) {
    C[i] = A[i] - B[1023-i];
}
```

Arrays de enteros almacenados desde `0x200000`.

- a) ¿Cuántos fallos de caché se producen?
- b) ¿Cuántos fallos con fusión de arrays? ¿Importa el orden?
- c) ¿Cuántos fallos con alargamiento de arrays? ¿Importa la cantidad?

Resolución:

#### Parámetros

- MP: 4MB = 2^22 B; Caché: 2KB, líneas 512B → **4 bloques** en caché
- Offset: 9 bits; Índice: 2 bits; Etiqueta: 11 bits
- Cada `int` = 4B → 512/4 = **128 enteros por bloque**
- A[1024]: 1024×4 = 4096B = 8 bloques (índices: 0x200000>>9 = 0x1000; mod 4: 0,1,2,3,0,1,2,3)
- B[1024]: empieza en 0x201000, 8 bloques
- C[1024]: empieza en 0x202000, 8 bloques

**Mapeo de bloques** (base/512 % 4):
- A: bloque k de A → índice = (0x200000/512 + k) % 4 = (0x1000 + k) % 4 = k % 4
- B: (0x201000/512 + k) % 4 = (0x1008 + k) % 4 = k % 4
- C: (0x202000/512 + k) % 4 = k % 4

⚠️ Los tres arrays mapean a los **mismos 4 índices**, con el mismo patrón de bloque k → índice k%4. Por tanto los bloques A[k], B[k] y C[k] van todos al mismo conjunto → **thrashing entre los 3 arrays**.

#### a) Fallos sin optimización

En cada iteración i:
- Se lee A[i] (bloque i/128, índice (i/128)%4).
- Se lee B[1023-i] (bloque (1023-i)/128, índice ((1023-i)/128)%4).
- Se escribe C[i] con asignación (bloque i/128, índice (i/128)%4, mismo que A[i]).

Con 3 arrays en 4 índices y el patrón de acceso simultáneo a A[i] y B[1023-i], los bloques de A y B activos en cada iteración mapean a índices distintos (A avanza de 0 a 7 linealmente, B retrocede de 7 a 0). En las primeras 128 iteraciones: A usa bloque 0 (índice 0) y B usa bloque 7 (índice 3) → índices distintos → no hay conflicto A-B. Pero C usa el mismo índice que A → conflicto A-C con write-allocate.

Análisis detallado por grupos de 128 iteraciones (un bloque completo):

En cada grupo de 128 iteraciones (mismo bloque de A y C activo):
- Primer acceso al bloque de A: **fallo**, carga bloque A.
- Primer acceso al bloque de B correspondiente: puede haber conflicto según el índice.
- Primer store a C con write-allocate: **fallo** si A está en el mismo índice (lo expulsa).

Con 3 arrays y 4 bloques en caché, los bloques activos simultáneamente son:
- 1 bloque de A (índice i/128 % 4)
- 1 bloque de B ((1023-i)/128 % 4)
- 1 bloque de C (= mismo índice que A)

Si el bloque de B mapea a un índice distinto del bloque de A/C → conviven sin conflicto entre A y B. Pero A y C siempre están en el mismo índice → thrashing A-C en cada iteración que cambia entre A y C dentro del mismo bloque.

En cada iteración dentro de un bloque: Load A → si C ocupa el índice, fallo en A; Store C (write-allocate) → fallo en C si A ocupa el índice. Resultado: **2 fallos por iteración** para las iteraciones en que A y C colisionan.

Para las 1024 iteraciones con 8 bloques de cada array:

- Fallos en A: al menos 1 por bloque (carga inicial) + conflictos con C → con thrashing A-C: 1024 fallos en A.
- Fallos en B: 1 por bloque × 8 bloques = 8 fallos (acceso secuencial inverso, sin conflicto con A en muchos casos).
- Fallos en C (write-allocate): 1024 fallos (cada write-allocate falla si A ocupó el índice).

> **Total aproximado ≈ 1024 + 8 + 1024 = ~2056 fallos.**

En la práctica con thrashing completo A-C: cada uno de los 1024 accesos a A y a C produce fallo → **2048 fallos** de A y C, más **8 fallos** de B (uno por bloque, si B no colisiona con A ni con C durante ese bloque).

> **Total ≈ 2056 fallos.**

#### b) Fusión de arrays (array merging)

Crear una estructura `{int A, B, C}` entrelazada:

```c
struct { int a; int b; int c; } data[1024];
// data[i].a = A[i], data[i].b = B[1023-i] no es trivial de entrelazar
```

Para este código, la fusión natural sería `{A[i], C[i]}` (que acceden al mismo índice i), dejando B separada:

```c
struct { int a; int c; } AC[1024];
int B[1024];
```

Ahora `AC[i].a` y `AC[i].c` están contiguos (8B por par). Un bloque de 512B almacena 64 pares. El acceso a `AC[i].a` y luego escribir `AC[i].c` están en el **mismo bloque** → 1 solo fallo para ambos.

- Fallos en AC: 1024/64 bloques × 1 fallo = **16 fallos** (carga de bloque una sola vez por par).
- Fallos en B: 8 bloques × 1 fallo = **8 fallos**.

> **Total con fusión ≈ 24 fallos.** Mejora drástica respecto a ~2056.

**¿Importa el orden de fusión?** Sí. Fusionar A con C (los que compiten por el mismo índice) elimina el thrashing. Fusionar A con B no ayudaría si C sigue siendo un array separado que colisiona con AB.

#### c) Alargamiento de arrays (array padding)

Añadir padding a A y/o B para que C no mapee al mismo índice:

```c
int A[1024 + P]; // P enteros de padding
int B[1024];
int C[1024];
```

Si P = 128 (un bloque de padding = 512B), C se desplaza en memoria lo suficiente para que sus bloques mapeen a índices distintos de los de A en la mayoría de iteraciones.

Con P = 128: A ocupa 1024+128 = 1152×4 = 4608B. C empieza 4608B después de A. El desplazamiento de C respecto a A es 4608B + 4096B(B) = 8704B = 17 bloques de 512B. 17 % 4 = **1** → C se desplaza 1 índice respecto a A.

Con este desplazamiento, A[i] y C[i] ya no mapean al mismo índice → desaparece el thrashing A-C.

- Fallos en A: 8 bloques × 1 fallo = **8 fallos**.
- Fallos en B: 8 bloques × 1 fallo = **8 fallos**.
- Fallos en C: 8 bloques × 1 fallo = **8 fallos** (write-allocate, 1 fallo por bloque nuevo).

> **Total con alargamiento ≈ 24 fallos.**

**¿Importa la cantidad alargada?** Sí. El padding debe ser tal que el desplazamiento resultante en índices de caché no introduzca nuevos conflictos. Un padding de múltiplo del tamaño de caché (2KB = 512 bloques de 4B = 128 enteros) no ayudaría (desplazamiento en índices = 0). Se necesita un padding que desplace al menos 1 posición de índice, por ejemplo **P = 128 enteros (1 bloque)**, que cambia el índice en 1.

---

Ejercicio A6

Enunciado: Caché de 4KB, bloques de 16B, emplazamiento directo, asignación en escritura. Código:

```c
int A[1024]; // A[0] en 0x0C000000
int B[1024];
int C[1024];

for (i=0; i<10; i++) {
    for (j=0; j<1024; j++) {
        C[i] += A[j] * B[j];
    }
}
```

- a) ¿Cuántos fallos?
- b) ¿Qué asociatividad mínima para evitar todos los fallos por conflicto?
- c) ¿Qué modificaciones de código para evitar todos los fallos por conflicto con emplazamiento directo?

Resolución:

#### Parámetros

- Caché: 4KB = 4096B, bloques 16B → **256 bloques**
- Offset: 4 bits; Índice: 8 bits; Etiqueta: 20 bits
- `int` = 4B → 4 enteros por bloque
- A[1024]: 1024×4 = 4096B = 256 bloques → bloques 0x0C000000/16 = 0xC00000 a 0xC000FF. Índices: 0x00 a 0xFF (todos los 256 índices).
- B[1024]: empieza en 0x0C001000 (4096B después). Bloques 0xC00100 a 0xC001FF. Índices: 0x00 a 0xFF (¡mismos que A!).
- C[1024]: empieza en 0x0C002000. Bloques 0xC00200 a 0xC002FF. Índices: 0x00 a 0xFF (¡mismos que A y B!).

**Conflicto total:** A, B y C mapean exactamente a los mismos 256 índices. Cualquier acceso simultáneo a A[j] y B[j] provoca conflicto (mismo índice j/4).

#### a) Fallos

**Bucle exterior i=0..9, bucle interior j=0..1023:**

En cada iteración j:
- Load A[j]: índice (j/4) % 256 = j/4 (para j<1024, todos distintos)
- Load B[j]: mismo índice que A[j] → **conflicto A-B en el mismo índice**
- Load C[i]: C[i] está en el bloque i/4, índice i/4 → solo 1 posición fija durante el bucle interior

Para la iteración i=0, C[0] está en índice 0. En el bucle j:
- j=0: Load A[0] (índice 0, fallo, carga bloque). Load B[0] (índice 0, fallo, expulsa A). Load C[0] (índice 0, fallo, expulsa B, carga C). Store C[0] (write-allocate, C está en índice 0 → acierto en escritura tras la carga).
- j=1: Load A[1] (índice 0, fallo, C ocupa índice 0). Load B[1] (índice 0, fallo). Store C[0] (índice 0, fallo, load C de nuevo). → 3 fallos/iter para j∈[0,3] (mismo bloque de A y B).
- j=4: nuevo bloque de A y B (índice 1). C[0] sigue en índice 0. No hay conflicto A-B-C para j>3 a menos que C[i] comparta índice con A[j] o B[j].

En general, C[i] ocupa el índice i/4 durante todo el bucle interior. Solo cuando j/4 = i/4 (es decir, j ∈ [4i, 4i+3]) hay triple conflicto A-B-C. Para los demás j, solo hay conflicto A-B.

**Fallos por vuelta del bucle interior (i fijo):**
- j∈[0..1023] con conflicto A-B: 2 fallos por acceso (A y B se expulsan mutuamente) → 1024×2 = 2048 fallos en A+B.
- C[i]: se carga una vez por vuelta del bucle interior pero se expulsa cuando j/4 = i/4. Esa expulsión ocurre 4 veces (j=4i..4i+3). En cada uno de esos j, C[i] debe recargarse → 4 fallos adicionales en C.
- Sin contar los j donde A/B/C coinciden más cuidadosamente: **≈ 2048 + 4 = 2052 fallos por i**.

**Total para i=0..9:** 10 × 2052 ≈ **20520 fallos**.

*(Nota: el primer fallo de cada bloque de A y B sería 256+256=512 fallos si no hubiera conflicto; el thrashing A-B los convierte en ~2048.)*

#### b) Asociatividad mínima para evitar fallos por conflicto

Para que A[j] y B[j] convivan sin expulsarse: necesitan estar en el mismo conjunto con al menos **2 vías**.

Para que A[j], B[j] y C[i] convivan también: necesitan **3 vías**.

> **Asociatividad mínima = 3 vías** (para evitar todos los conflictos de los 3 arrays).

#### c) Modificaciones de código con emplazamiento directo

**Opción 1 — Fusión de arrays A y B:**

```c
struct { int a; int b; } AB[1024];
int C[1024];

for (i=0; i<10; i++) {
    for (j=0; j<1024; j++) {
        C[i] += AB[j].a * AB[j].b;
    }
}
```

AB[j].a y AB[j].b están en el mismo bloque (8B por par, 2 pares por bloque de 16B). Un solo fallo trae ambos. C ocupa otros índices, sin conflicto con AB (siempre que no se superpongan). Fallos reducidos drásticamente.

**Opción 2 — Padding para desplazar B (o C) en memoria:**

```c
int A[1024];
int _pad[256]; // 256 enteros = 1KB de padding
int B[1024];
int C[1024];
```

B se desplaza 1KB = 64 bloques → B[j] mapea al índice (j/4 + 64) % 256. Para j<(256-64)×4 = 768: A[j] y B[j] mapean a índices distintos, sin conflicto. Para j≥768, vuelven a coincidir, pero se reducen los conflictos.

Con padding exacto de 256 bloques (1024 enteros): B se desplaza exactamente 256 bloques (tamaño de la caché), lo que **no ayuda** (mismo índice). El padding debe ser un número no múltiplo de 256 bloques.

> La fusión de A y B es la solución más efectiva con emplazamiento directo.

---

Ejercicio A7

Enunciado: Multiplicación de matrices 128×128:

```c
for (i=0; i<128; i++) {
    for (j=0; j<128; j++) {
        C[i][j] = 0;
        for (k=0; k<128; k++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
```

- a) ¿Qué matriz presenta peor localidad? ¿Se puede mejorar con intercambio de bucles? ¿Empeora la de alguna otra?
- b) Aplicar *loop tiling* y discutir si reduce fallos.

Resolución:

#### a) Localidad de las matrices

**Análisis de patrones de acceso** (C almacenado en row-major order):

- **A[i][k]:** i fijo en el bucle j; k varía en el bucle k. Acceso **secuencial por filas** → buena localidad espacial. A cada fila se accede 128 veces (una por cada j), pero dentro del bucle k es secuencial.
- **B[k][j]:** j fijo en el bucle j; k varía en el bucle k. Acceso **por columnas** (k aumenta, j fijo) → **mala localidad espacial** (saltos de 128 enteros entre elementos consecutivos de la columna j).
- **C[i][j]:** i y j fijos durante el bucle k → **acceso repetido al mismo elemento** (excelente localidad temporal, posiblemente mantenido en registro).

> **B es la matriz con peor localidad** espacial: se accede por columnas, lo que implica un fallo de caché por cada elemento (no hay reutilización de bloques entre accesos consecutivos a B[k][j] y B[k+1][j]).

**¿Se puede mejorar con intercambio de bucles?**

Intercambiando los bucles j y k:

```c
for (i=0; i<128; i++) {
    for (k=0; k<128; k++) {
        for (j=0; j<128; j++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
```

Ahora B[k][j] se accede con k fijo y j variando → **acceso secuencial por filas de B** → excelente localidad espacial en B. A[i][k] con k fijo y j variando → A[i][k] es un **escalar constante** en el bucle j, posible mantenerlo en registro.

> **Sí mejora la localidad de B** drásticamente. La localidad de A no empeora (con k variable en el nuevo bucle medio, A[i][k] sigue siendo acceso por filas). C[i][j] ahora varía con j → acceso secuencial por filas de C, que también es bueno.

**¿Empeora la localidad de alguna otra?** No significativamente. El intercambio j↔k mejora B sin penalizar A ni C.

#### b) Loop tiling (blocking)

El *loop tiling* divide los bucles en bloques de tamaño T para que los datos activos en cada sub-bloque quepan en caché:

```c
int T = ...; // tamaño de tile

for (i=0; i<128; i+=T) {
    for (j=0; j<128; j+=T) {
        for (k=0; k<128; k+=T) {
            // Sub-bloque de tamaño T×T
            for (ii=i; ii<min(i+T,128); ii++) {
                for (jj=j; jj<min(j+T,128); jj++) {
                    for (kk=k; kk<min(k+T,128); kk++) {
                        C[ii][jj] += A[ii][kk] * B[kk][jj];
                    }
                }
            }
        }
    }
}
```

**¿Disminuye el número de fallos de caché?**

Sí, el *loop tiling* es conveniente para este problema por las siguientes razones:

1. **Reutilización de bloques de A:** con el orden original, una fila de A[i] se carga completa para calcular C[i][0], luego para C[i][1], etc. Si la fila entera cabe en caché, hay reutilización. Con tiling, la submatriz A[i..i+T][k..k+T] se reutiliza para todos los j en el tile.

2. **Reutilización de bloques de B:** B[k..k+T][j..j+T] se carga una vez para todos los i en el tile, en lugar de recargarse 128 veces (una por cada i).

3. **El tile se ajusta a la caché:** si T² × 3 enteros caben en caché (3 submatrices de A, B, C de T×T), todos los accesos al tile son aciertos tras la carga inicial. Para una caché de C bytes y tiles de T×T enteros: T ≤ √(C / (3×4)).

> **El *loop tiling* reduce significativamente los fallos de caché** en la multiplicación de matrices, especialmente los fallos en B (que tenía peor localidad), al garantizar que los datos activos en cada fase quepan en caché. Es una transformación muy conveniente para este código, y se combina bien con el intercambio de bucles del apartado anterior.

---

Ejercicio A8

Enunciado: Máquina con direcciones de 16 bits, caché de datos de acceso directo, 4 bloques de 16B, write-back con asignación en escritura. Código (ignorando variable `tmp` en registro):

```c
int v[16];    // dirección 0x0000
int mask[4];  // dirección 0x0040

for (i=0; i<12; i++) {
    tmp = 0;
    for (j=0; j<4; j++) {
        tmp = tmp + mask[j] * v[i];
    }
    v[i] = tmp;
}
```

- a) ¿Cuántos fallos se producen? Reemplazamientos e iteraciones.
- b) Recalcular con caché asociativa 2 vías, LRU.
- c) ¿Qué etiqueta tendrá el bloque 2 de la caché tras la ejecución?

Resolución:

#### Parámetros

- Caché: 4 bloques × 16B = 64B total; dirección 16 bits
- Offset: 4 bits; Índice: 2 bits (log2(4)); Etiqueta: 10 bits
- `int` = 4B → 4 enteros por bloque de 16B
- v[16]: 16×4 = 64B = **4 bloques** → en 0x0000 a 0x003F. Bloques: 0x0000(índ 0), 0x0010(índ 1), 0x0020(índ 2), 0x0030(índ 3).
- mask[4]: 4×4 = 16B = **1 bloque** → en 0x0040 a 0x004F. Bloque: 0x0040 → índice 0x0040/16 % 4 = 4 % 4 = **0**.

**Conflicto:** mask y v[0..3] mapean al índice 0 (mask en índice 0, v[0..3] también en índice 0).

- v[0..3]: bloque 0x0000, índice 0, etiqueta 0x000
- v[4..7]: bloque 0x0010, índice 1, etiqueta 0x001
- v[8..11]: bloque 0x0020, índice 2, etiqueta 0x002
- v[12..15]: bloque 0x0030, índice 3, etiqueta 0x003
- mask[0..3]: bloque 0x0040, índice 0, etiqueta 0x004

#### a) Emplazamiento directo, write-back, write-allocate

El bucle externo i accede a `v[i]` (read y write) y el bucle interno j accede a `mask[j]`.

**i=0 (v[0], índice 0):**
- j=0: Load mask[0] (índice 0, vacío) → **Fallo**. Carga bloque mask.
- j=0: Load v[0] (índice 0, mask ocupa índice 0) → **Fallo** (reemplazo de mask). Carga v[0..3]. (mask se escribe de vuelta a MP si estaba dirty, pero mask no fue modificada → no dirty, descarte limpio).
- j=1: Load mask[1] (índice 0, v ocupa) → **Fallo** (reemplazo). Carga mask.
- j=1: Load v[0] (índice 0, mask ocupa) → **Fallo** (reemplazo). Carga v[0..3].
- j=2: Load mask[2] → **Fallo**. j=2 Load v[0] → **Fallo**.
- j=3: Load mask[3] → **Fallo**. j=3 Load v[0] → **Fallo**.
- Store v[0] (write-back, write-allocate): v[0..3] ya está en caché (del último Load v[0]) → **Acierto** en escritura (marca bloque dirty).

Total i=0: **8 fallos** (4 pares mask/v), 1 acierto (store).

**i=1 (v[1], índice 0):** mismo patrón. v[1] está en el mismo bloque que v[0], pero el bloque puede estar en caché si no fue expulsado. Al final de i=0, v[0..3] está en índice 0 (dirty). Bucle j para i=1:
- j=0: Load mask[0] (índice 0, v está dirty) → **Fallo** (reemplazo, write-back: escribe bloque v a MP). Carga mask.
- j=0: Load v[1] (índice 0, mask ocupa) → **Fallo** (reemplazo). Carga v[0..3] desde MP (fue escrito de vuelta).
- ... mismo patrón: **8 fallos** por i, para i=0..3 (todos usan índice 0 para v).

**i=2, 3:** mismo análisis → **8 fallos cada uno**.

**i=4 (v[4], índice 1):** v[4..7] en índice 1. mask en índice 0. Ya no hay conflicto v-mask en el mismo índice.
- j=0: Load mask[0] (índice 0): ¿está en caché? Al final de i=3, índice 0 tiene v[0..3] (o mask según el último acceso). Último acceso en i=3 fue Load v[3] → índice 0 tiene v[0..3]. → **Fallo** para mask (índice 0 tiene v, reemplazo dirty). Load mask → Fallo.
- j=0: Load v[4] (índice 1, vacío) → **Fallo**. Carga v[4..7].
- j=1: Load mask[1] (índice 0, mask ya cargado del j=0) → **Acierto**.
- j=1: Load v[4] (índice 1, v[4..7] en caché) → **Acierto**.
- j=2, 3: Load mask → **Acierto**; Load v[4] → **Acierto**.
- Store v[4]: índice 1, v en caché → **Acierto** (write-back, marca dirty).

Total i=4: **2 fallos** (mask inicial + v[4] inicial).

**i=5, 6, 7:** v[5..7] en el mismo bloque que v[4] (índice 1). mask ya estará en índice 0. Al pasar de i=4 a i=5:
- j=0: Load mask → ¿está en índice 0? Sí (de i=4). → **Acierto**.
- j=0: Load v[5] (índice 1, v[4..7] en caché, marcado dirty de la store de v[4]) → **Acierto**.
- j=1,2,3: mask → Acierto; v[5] → Acierto.
- Store v[5]: **Acierto**.

Total i=5,6,7: **0 fallos** cada uno.

**i=8..11 (v[8..11], índice 2):** mismo análisis que i=4..7.
- i=8: **2 fallos** (mask y v[8..11] primera vez).
- i=9,10,11: **0 fallos** cada uno.

**i=12..15 (v[12..15], índice 3):**
- i=12: **2 fallos**.
- i=13,14,15: **0 fallos**.

**Resumen total de fallos:**

| Rango i | Fallos/iter | Iters | Total |
|---------|------------|-------|-------|
| 0–3     | 8          | 4     | 32    |
| 4       | 2          | 1     | 2     |
| 5–7     | 0          | 3     | 0     |
| 8       | 2          | 1     | 2     |
| 9–11    | 0          | 3     | 0     |
| 12      | 2          | 1     | 2     |
| 13–15   | 0          | 3     | 0     |

> **Total fallos = 32 + 2 + 2 + 2 = 38 fallos.**

**Reemplazamientos:** ocurren en i=0..3 (8 reemplazamientos por i, entre mask y v en índice 0) y al cambiar de bloque de v (i=4, 8, 12, reemplazamiento de v anterior por mask y luego de mask por nuevo bloque v). **Total reemplazamientos ≈ 32 + 3 = 35**.

#### b) Caché asociativa 2 vías, LRU

Con 2 vías: 2 conjuntos, índice = 1 bit (bit 4), etiqueta = 11 bits.

- v[0..3]: bloque 0x0000, índice 0 (bit 4 = 0)
- v[4..7]: bloque 0x0010, índice 1
- v[8..11]: bloque 0x0020, índice 0
- v[12..15]: bloque 0x0030, índice 1
- mask: bloque 0x0040, índice 0

Conjunto 0: mask (etq 0x004) y v[0..3] (etq 0x000) y v[8..11] (etq 0x002) → con 2 vías, pueden coexistir **2 de los 3**.
Conjunto 1: v[4..7] (etq 0x001) y v[12..15] (etq 0x003) → solo 2 arrays, caben en las 2 vías sin conflicto.

**i=0 (conj 0: mask vs v[0..3]):**
- j=0: Load mask → Fallo (conj 0 vacío). Carga mask en vía 0.
- j=0: Load v[0] → Fallo (conj 0 tiene solo mask, vía 1 libre). Carga v[0..3] en vía 1. **Ambos coexisten**.
- j=1: Load mask[1] → **Acierto** (mask en vía 0). Load v[0] → **Acierto** (v en vía 1).
- j=2,3: mask → Acierto; v[0] → Acierto.
- Store v[0]: **Acierto** (write-back, dirty).

Total i=0: **2 fallos** (mask y v[0..3] al inicio).

**i=1,2,3:** mask y v[0..3] siguen en las 2 vías (LRU los mantiene). **0 fallos** por iteración.

**i=4 (conj 1: v[4..7]):**
- j=0: Load mask → **Acierto** (mask en conj 0, vía 0). Load v[4] → Fallo (conj 1, vía 0 vacía). Carga v[4..7].
- j=1,2,3: mask → Acierto; v[4] → Acierto.
- Store v[4]: Acierto.

Total i=4: **1 fallo**.

**i=5,6,7:** 0 fallos (v[4..7] en conj 1, mask en conj 0).

**i=8 (conj 0: v[8..11], etq 0x002):**
- j=0: Load mask → Acierto (conj 0 tiene mask y v[0..3]). Load v[8] → Fallo (etq 0x002 no está en conj 0; LRU expulsa el menos usado entre mask y v[0..3]). Dado que en i=4..7 solo se accedió a mask en conj 0, v[0..3] es el LRU → se expulsa (write-back: v[0..3] dirty de i=3). Carga v[8..11].
- Conj 0 ahora: mask (vía 0, MRU) y v[8..11] (vía 1, reciente).
- j=1,2,3: mask → Acierto; v[8] → Acierto.

Total i=8: **1 fallo**.

**i=9,10,11:** 0 fallos.

**i=12 (conj 1: v[12..15], etq 0x003):**
- j=0: Load mask → Acierto. Load v[12] → Fallo (conj 1 tiene v[4..7], LRU tras i=4..7; en i=8..11 no se accedió al conj 1). LRU expulsa v[4..7] (dirty). Carga v[12..15].
- j=1,2,3: mask → Acierto; v[12] → Acierto.

Total i=12: **1 fallo**.

**i=13,14,15 (v[12..15] sigue en caché):** 0 fallos.

**Resumen 2 vías:**

| Rango i | Fallos |
|---------|--------|
| 0       | 2      |
| 1–3     | 0      |
| 4       | 1      |
| 5–7     | 0      |
| 8       | 1      |
| 9–11    | 0      |
| 12      | 1      |
| 13–15   | 0      |

> **Total fallos con 2 vías = 5 fallos.** Mejora radical respecto a los 38 de emplazamiento directo.

#### c) Etiqueta del bloque 2 de la caché tras la ejecución

Con emplazamiento directo, el **bloque físico 2** (índice 2) de la caché almacena al final de la ejecución el último dato que fue cargado en ese índice.

El índice 2 corresponde a las direcciones cuyo bits [5:4] = 10, es decir, 0x0020..0x002F → bloque v[8..11].

El último acceso al índice 2 fue para v[8..11] en i=8..11. Tras la ejecución del bucle, el bloque de caché 2 contiene v[8..11], con etiqueta = 0x0020 >> (4+2) = dirección base del bloque / 64.

Etiqueta = dirección_bloque >> (bits_offset + bits_índice) = 0x0020 >> 4 >> 2 = 0x0002 >> 2 = 0.

Más precisamente: etiqueta = nº de bloque MP / nº de bloques en caché = (0x0020/16) / 4 = 2/4 = 0. Como nº bloque 2 → etiqueta = 2 >> 2 = 0.

Con 16-bit addresses y 4 bits offset, 2 bits índice → etiqueta = 10 bits = bits [15:6] de la dirección.

Para 0x0020: bits [15:6] = 0x0020 >> 6 = 0x0000 → pero redondeando: 0x0020 = 0b 0000 0000 0010 0000. Bits 15:6 = 0b 00 0000 0000 = **0x000**.

Hmm, revisemos: 0x0020 en binario (16 bits) = 0000 0000 0010 0000. Bits [15:6] = 00 0000 0000 = 0. Pero la etiqueta diferencia bloques → etiqueta = dirección / 64 = 0x0020/64 = 0x0020/0x40 = 0x00. No diferencia bien. Lo correcto es: etiqueta = nº_bloque_MP / num_bloques_caché = (0x0020/16) / 4 = 2/4... en realidad la etiqueta es el cociente entero:

nº bloque de MP = 0x0020 / 16 = 2. Índice en caché = 2 % 4 = 2. Etiqueta = 2 / 4 = 0 (parte entera).

Esto significa que v[0..3] (bloque 0), v[4..7] (bloque 1), v[8..11] (bloque 2) y v[12..15] (bloque 3) tienen **etiquetas 0, 0, 0, 0** respectivamente con índices 0, 1, 2, 3. La etiqueta que los distingue es la que queda en los bits superiores, que en este caso para todos los bloques de v es 0x000.

Para mask: nº bloque = 0x0040/16 = 4. Índice = 4%4 = 0. Etiqueta = 4/4 = **0x001** (en hexadecimal) → etiqueta = bits [15:6] de 0x0040 = 0x0040>>6 = 1 = **0x001**.

Etiqueta de v[8..11]: dirección 0x0020 >> 6 = 0 = **0x000**.

> **La etiqueta del bloque 2 de la caché al final de la ejecución es `0x000`** (correspondiente al bloque de memoria de v[8..11]).
