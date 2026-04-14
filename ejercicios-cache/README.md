# Ejercicios EC

## Mario González García

---

## Caché

### Problemas Básicos

---

Ejercicio 1

Enunciado: En un computador con cachés separadas de datos e instrucciones, de 32 KB cada una y bloques de 256 B, se quiere ejecutar el siguiente código C que realiza la operación matricial `C = A+B`:

```c
#define N 128
int A[N][N];
int B[N][N];
int C[N][N];

for (i=0; i < N; i++)
    for (j=0; j < N; j++)
        C[i][j] = A[i][j] + B[i][j];
```

La matriz A se ubica en `0x0C000000`, y B y C se almacenan consecutivamente a continuación de A. Las variables `i`, `j` y las direcciones base de los arrays están en registros.

a) Calcular el número de fallos de caché en lectura (load) y escritura (store) si la caché es de emplazamiento directo sin asignación en escritura.
b) Repetir el cálculo para una caché asociativa por conjuntos de 2 vías con política LRU, sin y con asignación en escritura.
c) Variante con array de structs `AB[N][N]` con A y B entrelazados. Calcular fallos y comparar.
d) ¿Puede obtenerse un resultado similar al del apartado c con el código original sin aumentar el tamaño de la caché?

Resolución:

**Datos de la caché**

La caché tiene 32 KB con bloques de 256 B, lo que da 128 bloques en total. Cada bloque almacena 64 enteros, ya que cada `int` ocupa 4 bytes. Cada fila de la matriz tiene 128 enteros y ocupa 2 bloques. Cada matriz completa ocupa 256 bloques, lo que significa que ninguna de las tres matrices cabe por sí sola en caché.

El problema fundamental es el aliasing: A ocupa los bloques 0 a 255, B ocupa los bloques 256 a 511 y C ocupa los bloques 512 a 767. Con emplazamiento directo y 128 bloques en caché, el bloque k de A, el bloque k de B y el bloque k de C mapean siempre al mismo índice (k % 128), lo que genera conflictos en cada acceso.

**Apartado a: emplazamiento directo sin asignación en escritura**

En cada iteración del bucle interno se lee A[i][j], se lee B[i][j] y se escribe C[i][j].

Para las lecturas, A y B mapean al mismo índice de caché porque sus bloques están separados exactamente 256 posiciones, que es múltiplo de 128. Al cargar un bloque de A se expulsa el bloque de B que ocupa ese índice, y viceversa. Esto produce thrashing continuo: cada acceso a A expulsa a B y cada acceso a B expulsa a A. El resultado es un fallo en cada acceso de lectura a ambas matrices, lo que suma 128 x 128 + 128 x 128 = **32 768 fallos de lectura**.

Para las escrituras, sin asignación en escritura los fallos no cargan el bloque en caché, sino que van directamente a memoria principal. C nunca entra en caché, por lo que todos los stores fallan: 128 x 128 = **16 384 fallos de escritura**.

Total apartado a: **49 152 fallos**.

**Apartado b: asociativa por conjuntos de 2 vías con LRU**

Con 2 vías y bloques de 256 B, el número de conjuntos es 32 768 / (256 x 2) = **64 conjuntos**. Sin embargo, los tres arrays siguen mapeando a los mismos conjuntos: el bloque k de A va al conjunto k % 64, igual que el bloque k de B y el bloque k de C. Como hay tres arrays compitiendo y solo dos vías por conjunto, el thrashing persiste.

Sin asignación en escritura, el comportamiento es idéntico al apartado a: **32 768 fallos de lectura y 16 384 fallos de escritura**.

Con asignación en escritura, los stores a C también traen el bloque a caché, pero ahora los tres arrays compiten por las mismas 2 vías. El patrón es: fallo en A (carga A), fallo en B (carga B, expulsa A), fallo en C (carga C, expulsa B). En el siguiente elemento del mismo bloque, A vuelve a fallar. El thrashing de tres vías hace que el resultado sea prácticamente el mismo que sin asignación. La asociatividad de 2 vías mejoraría el rendimiento si solo hubiera 2 arrays en conflicto, pero con 3 no es suficiente.

**Apartado c: struct con A y B entrelazados**

```c
struct { int A; int B; } AB[N][N];
int C[N][N];
```

Ahora `AB[i][j].A` y `AB[i][j].B` son contiguos en memoria (8 bytes por par). Un bloque de 256 B almacena 32 pares. El acceso a `.A` y `.B` del mismo elemento produce un solo fallo que trae ambos valores.

AB ocupa 512 bloques y C ocupa 256 bloques. Como solo hay dos arrays compitiendo en cada conjunto (AB y C), y con 2 vías pueden coexistir sin expulsarse mutuamente, los fallos se reducen drásticamente.

Fallos en AB: 512 (uno por cada bloque, acceso secuencial). Fallos en C: 256 (uno por bloque). Total aproximado: **768 fallos**, frente a los 49 152 del caso original. La mejora viene de que el thrashing entre A y B desaparece al unirlos en el mismo bloque de caché.

**Apartado d: alternativa sin modificar el código**

Es posible obtener un resultado similar aplicando **fusión de arrays** a nivel de layout en memoria, que es exactamente lo que hace el struct del apartado c. Sin cambiar el algoritmo, también se puede aplicar **padding** a los arrays para que sus bloques no mapeen al mismo índice de caché, rompiendo el aliasing. Esto no requiere aumentar la caché ni cambiar el código fuente, solo reorganizar cómo se declaran o alinean los datos en memoria.

---

Ejercicio 2

Enunciado: Sea un computador con memoria principal de 64 KB direccionable en bytes, con caché de 2 KB, líneas de 256 B y prebúsqueda bajo fallo (en caso de fallo se trae el bloque que lo provoca y el siguiente).

Secuencia de accesos: `0x3345`, `0x1244`, `0x3374`, `0x5BAC`, `0x132A`, `0x5C41`, `0x13AF`.

Mostrar la evolución de la caché en:
a) Organización directa.
b) Asociativa por conjuntos de 2 vías con LRU.
c) Organización directa con buffer de 512 B completamente asociativo con política FIFO.

Resolución:

**Datos de la caché**

La memoria principal tiene 64 KB, lo que implica direcciones de 16 bits. La caché tiene 2 KB con líneas de 256 B, lo que da 8 bloques en total. Para emplazamiento directo: 8 bits de offset (bits 7:0), 3 bits de índice (bits 10:8) y 5 bits de etiqueta (bits 15:11). Para 2 vías: 4 conjuntos, 2 bits de índice (bits 9:8) y 6 bits de etiqueta (bits 15:10). La prebúsqueda funciona así: en cada fallo se cargan el bloque accedido y el siguiente (número de bloque + 1).

**Descomposición de las direcciones (emplazamiento directo)**

| Dirección | Etiqueta | Índice | Bloque MP |
|-----------|----------|--------|-----------|
| 0x3345    | 00110    | 011    | 0x33      |
| 0x1244    | 00010    | 010    | 0x12      |
| 0x3374    | 00110    | 011    | 0x33      |
| 0x5BAC    | 01011    | 011    | 0x5B      |
| 0x132A    | 00010    | 011    | 0x13      |
| 0x5C41    | 01011    | 100    | 0x5C      |
| 0x13AF    | 00010    | 011    | 0x13      |

**Apartado a: organización directa**

Acceso 1 (0x3345, índice 3): el índice 3 está vacío, **fallo**. Se carga el bloque 0x33 en el índice 3 y por prebúsqueda se carga el bloque 0x34 en el índice 4.

Acceso 2 (0x1244, índice 2): el índice 2 está vacío, **fallo**. Se carga el bloque 0x12 en el índice 2 y por prebúsqueda el bloque 0x13 en el índice 3, que expulsa el bloque 0x33.

Acceso 3 (0x3374, índice 3): el índice 3 ahora tiene el bloque 0x13 (etiqueta 0x02), que no coincide con la etiqueta buscada (0x06), **fallo**. Se carga el bloque 0x33 y por prebúsqueda el bloque 0x34 vuelve al índice 4.

Acceso 4 (0x5BAC, índice 3): el índice 3 tiene el bloque 0x33 (etiqueta 0x06), que no coincide con la buscada (0x0B), **fallo**. Se carga el bloque 0x5B y por prebúsqueda el bloque 0x5C en el índice 4.

Acceso 5 (0x132A, índice 3): el índice 3 tiene el bloque 0x5B (etiqueta 0x0B), que no coincide, **fallo**. Se carga el bloque 0x13 y por prebúsqueda el bloque 0x14 en el índice 4.

Acceso 6 (0x5C41, índice 4): el índice 4 tiene el bloque 0x14 (etiqueta 0x02), que no coincide, **fallo**. Se carga el bloque 0x5C y por prebúsqueda el bloque 0x5D en el índice 5.

Acceso 7 (0x13AF, índice 3): el índice 3 tiene el bloque 0x13 cargado en el acceso 5, y la etiqueta coincide, **acierto**.

Resultado del apartado a: 6 fallos y 1 acierto.

**Apartado b: asociativa por conjuntos de 2 vías con LRU**

Acceso 1 (0x3345, conjunto 3): vacío, **fallo**. Se carga el bloque 0x33 en la vía 0. Por prebúsqueda, el bloque 0x34 va al conjunto 0 en la vía 0.

Acceso 2 (0x1244, conjunto 2): vacío, **fallo**. Se carga el bloque 0x12 en la vía 0. Por prebúsqueda, el bloque 0x13 va al conjunto 3 en la vía 1. El conjunto 3 ahora tiene los bloques 0x33 y 0x13.

Acceso 3 (0x3374, conjunto 3): el bloque 0x33 está en la vía 0, **acierto**. Se actualiza LRU: 0x33 pasa a ser el más reciente.

Acceso 4 (0x5BAC, conjunto 3): ninguna de las dos vías tiene el bloque 0x5B, **fallo**. LRU expulsa el bloque 0x13 (el menos reciente). Se carga 0x5B. Por prebúsqueda, el bloque 0x5C va al conjunto 0 y expulsa 0x34 por LRU. El conjunto 3 queda con 0x33 y 0x5B.

Acceso 5 (0x132A, conjunto 3): el bloque 0x13 no está, **fallo**. LRU expulsa el bloque 0x5B. Se carga 0x13. Por prebúsqueda, el bloque 0x14 va al conjunto 3 y expulsa 0x33 (ahora el LRU). El conjunto 3 queda con 0x13 y 0x14.

Acceso 6 (0x5C41, conjunto 0): el conjunto 0 está vacío tras las expulsiones anteriores, **fallo**. Se cargan 0x5C y su prebúsqueda 0x5D.

Acceso 7 (0x13AF, conjunto 3): el bloque 0x13 sigue en el conjunto 3, **acierto**.

Resultado del apartado b: 5 fallos y 2 aciertos. La asociatividad de 2 vías permite que el acceso 3 sea un acierto, lo que no ocurría en el apartado a.

**Apartado c: organización directa con buffer de prebúsqueda**

El buffer tiene 512 / 256 = 2 entradas y funciona con política FIFO. Cuando se produce un fallo, se comprueba primero si el bloque está en el buffer. Si es así, se mueve del buffer a la caché sin contar como fallo de memoria principal.

Acceso 1 (0x3345): **fallo** en caché y buffer. Se carga el bloque 0x33 en caché (índice 3) y el bloque 0x34 va al buffer. Buffer: [0x34].

Acceso 2 (0x1244): **fallo** en caché y buffer. Se carga el bloque 0x12 en caché (índice 2) y el bloque 0x13 va al buffer. Buffer: [0x34, 0x13].

Acceso 3 (0x3374): el bloque 0x33 sigue en el índice 3, **acierto** directo en caché.

Acceso 4 (0x5BAC): el bloque 0x5B no está en caché ni en el buffer, **fallo**. El bloque 0x33 se expulsa del índice 3 y entra 0x5B. El bloque 0x5C se añade al buffer, que por FIFO expulsa el 0x34. Buffer: [0x13, 0x5C].

Acceso 5 (0x132A): el bloque 0x13 no está en la caché (índice 3 tiene 0x5B), pero sí está en el buffer. Se mueve de buffer a caché y el bloque 0x5B pasa al buffer. A su vez entra la prebúsqueda del bloque 0x14. Buffer: [0x5C, 0x14]. Este acceso se resuelve en el buffer sin ir a MP.

Acceso 6 (0x5C41): el bloque 0x5C no está en caché (índice 4), pero sí en el buffer. Se mueve de buffer a caché. El bloque que ocupaba el índice 4 se expulsa y entra al buffer, mientras que el bloque 0x5D se añade. Este acceso también se resuelve en el buffer.

Acceso 7 (0x13AF): el bloque 0x13 está en caché (índice 3, cargado en el acceso 5), **acierto** directo.

Resultado del apartado c: 4 fallos en memoria principal, 2 aciertos de buffer y 2 aciertos directos en caché. El buffer reduce los fallos efectivos de 6 a 4 aprovechando la localidad espacial.

---

Ejercicio 3

Enunciado: Computador con MP de 4 MB, caché de 4 KB con líneas de 512 B y 2 vías (FIFO), y caché víctima de 1024 B completamente asociativa (FIFO). Las tablas de estado inicial están dadas. Secuencia de accesos: `0x334500`, `0x14BF00`, `0x150084`, `0x004021`, `0x0540AB`, `0x0041F1`.

Resolución:

**Datos de la caché**

La memoria principal tiene 4 MB, con direcciones de 22 bits. La caché L1 tiene 4 KB con líneas de 512 B y 2 vías, lo que da 4 conjuntos. El offset ocupa 9 bits (bits 8:0), el índice 2 bits (bits 10:9) y la etiqueta los 11 bits restantes (bits 21:11). La caché víctima tiene 1024 / 512 = 2 entradas.

**Estado inicial de la caché L1**

| Conjunto | Via 0 (mas antigua) | Via 1 (mas reciente) |
|----------|---------------------|----------------------|
| 0        | etiqueta 0x668      | etiqueta 0x008       |
| 1        | etiqueta 0x297      | etiqueta 0x368       |
| 2        | etiqueta 0x5FF      | etiqueta 0x668       |
| 3        | etiqueta 0x297      | etiqueta 0x200       |

Estado inicial de la caché víctima: etiquetas 0x0A80 (posición 0, FIFO) y 0x3FFF (posición 1).

**Descomposición de las direcciones**

El número de bloque se obtiene dividiendo la dirección entre 512. El índice de conjunto es el número de bloque módulo 4 y la etiqueta es el número de bloque dividido entre 4.

| Dirección  | Numero de bloque | Indice | Etiqueta |
|-----------|-----------------|--------|----------|
| 0x334500  | 0x19A2          | 2      | 0x668    |
| 0x14BF00  | 0x0A5F          | 3      | 0x297    |
| 0x150084  | 0x0A80          | 0      | 0x2A0    |
| 0x004021  | 0x0020          | 0      | 0x008    |
| 0x0540AB  | 0x02A0          | 0      | 0x0A8    |
| 0x0041F1  | 0x0020          | 0      | 0x008    |

**Traza de accesos**

Acceso 1 (0x334500, conjunto 2, etiqueta 0x668): el conjunto 2 tiene en la vía 1 la etiqueta 0x668, que coincide. **Acierto** directo en caché.

Acceso 2 (0x14BF00, conjunto 3, etiqueta 0x297): el conjunto 3 tiene en la vía 0 la etiqueta 0x297, que coincide. **Acierto** directo en caché.

Acceso 3 (0x150084, conjunto 0, etiqueta 0x2A0): el conjunto 0 tiene las etiquetas 0x668 y 0x008, ninguna coincide. Tampoco está en la caché víctima (que tiene 0x0A80 y 0x3FFF). **Fallo**. Por política FIFO se expulsa la vía 0 (etiqueta 0x668) del conjunto 0, que pasa a la caché víctima (donde se expulsa 0x0A80, la más antigua). Entra el nuevo bloque con etiqueta 0x2A0. El conjunto 0 queda con 0x2A0 (vía 0) y 0x008 (vía 1). Víctima: [0x668, 0x3FFF].

Acceso 4 (0x004021, conjunto 0, etiqueta 0x008): el conjunto 0 tiene las etiquetas 0x2A0 y 0x008. La etiqueta 0x008 coincide con la vía 1. **Acierto**.

Acceso 5 (0x0540AB, conjunto 0, etiqueta 0x0A8): el conjunto 0 tiene 0x2A0 y 0x008, ninguna coincide. La caché víctima tiene 0x668 y 0x3FFF, tampoco. **Fallo**. FIFO expulsa la vía más antigua del conjunto 0 (etiqueta 0x2A0, que llegó en el acceso 3) y la manda a la caché víctima (que expulsa 0x668, la más antigua). Entra 0x0A8. Víctima: [0x3FFF, 0x2A0].

Acceso 6 (0x0041F1, conjunto 0, etiqueta 0x008): el conjunto 0 tiene 0x008 (vía 1) y 0x0A8 (vía 0). La etiqueta 0x008 coincide. **Acierto** directo en caché.

**Resumen**

| Acceso     | Resultado         |
|-----------|-------------------|
| 0x334500  | Acierto en L1     |
| 0x14BF00  | Acierto en L1     |
| 0x150084  | Fallo             |
| 0x004021  | Acierto en L1     |
| 0x0540AB  | Fallo             |
| 0x0041F1  | Acierto en L1     |

2 fallos en MP, 4 aciertos directos en caché y ningún acceso a la caché víctima en este caso.

---

Ejercicio 4

Enunciado: Computador con direcciones de 32 bits, caché de datos de emplazamiento directo con 8 bloques de 16 B, write-back con asignación en escritura. La variable `a` está en registro. Código:

```c
int nota[128];   // nota[0] en 0x00000000
int media[128];  // media[0] a continuacion de nota[127]

for (i=0; i<128; i++) {
    if (i>7 && i<64) {
        nota[i] = media[i]/2;
    } else {
        a = nota[i] * media[i];
    }
}
```

a) ¿Cuántos fallos de caché se producen?
b) Dos optimizaciones independientes que mejoren el resultado.
c) Tiempo de acceso total con tiempo de caché = 1 ns, palabra en MP = 50 ns y bloque MP a caché = 200 ns.
d) Con caché L2 de 1 MB, asociatividad 4, bloques de 16 B, write-allocate, write-back y tiempo de transferencia de bloque L2 a L1 = 15 ns.

Resolución:

**Datos de la caché**

La caché tiene 128 B con bloques de 16 B, lo que da 8 bloques en total. El offset ocupa 4 bits, el índice 3 bits y la etiqueta los 25 bits restantes. Cada bloque almacena 4 enteros. El array `nota` ocupa 512 B (32 bloques de MP) y `media` también, comenzando justo después.

El problema de aliasing es inmediato: `nota[i]` está en el bloque número i/4, cuyo índice en caché es (i/4) % 8. `media[i]` está en el bloque número i/4 + 32, cuyo índice en caché también es (i/4 + 32) % 8 = (i/4) % 8. Ambos arrays mapean siempre al mismo índice de caché.

**Apartado a: conteo de fallos**

La rama `else` cubre i en [0, 7] y en [64, 127], es decir, 72 elementos. En cada acceso se lee `nota[i]` y `media[i]`, que compiten por el mismo índice. El resultado es thrashing constante: cada acceso a uno expulsa al otro. Esto produce 2 fallos por elemento (uno para nota, uno para media), con lo que la rama else genera 72 x 2 = **144 fallos de lectura**.

La rama `if` cubre i en [8, 63], es decir, 56 elementos. Aquí se lee `media[i]` y se escribe `nota[i]`. Los 56 elementos de media abarcan 14 bloques, con un fallo por bloque, lo que suma **14 fallos de lectura**. Las escrituras a `nota` con asignación en escritura traen el bloque a caché, pero como `media` ya ocupa ese índice, siempre hay fallo también en las escrituras: **14 fallos adicionales**.

Total: **158 fallos de caché** (172 si se cuenta también el thrashing exacto en la rama if).

**Apartado b: optimizaciones**

La primera optimización es la **fusión de arrays**: intercalar `nota` y `media` en una misma estructura.

```c
struct { int nota; int media; } datos[128];
```

Ahora `datos[i].nota` y `datos[i].media` son contiguos en memoria. Un solo fallo trae ambos valores, eliminando el thrashing. Los fallos se reducen aproximadamente a la mitad de los bloques accedidos.

La segunda optimización es el **padding**: añadir elementos de relleno al final de `nota` para desplazar `media` a un índice de caché diferente.

```c
int nota[128 + 32]; // 32 enteros extra desplazan media un indice
int media[128];
```

Con el desplazamiento adecuado, `nota[i]` y `media[i]` ya no compiten por el mismo índice y el thrashing desaparece. Los fallos bajan a aproximadamente 64.

**Apartado c: tiempo total de acceso**

Con 158 fallos y 200 ns de penalización por fallo, y aproximadamente 42 aciertos de lectura más 56 escrituras directas a MP a 50 ns cada una:

- Aciertos en lectura: 42 x 1 ns = 42 ns
- Fallos en lectura: 158 x (1 + 200) ns = 31 758 ns
- Escrituras directas a MP: 56 x 50 ns = 2 800 ns

**Tiempo total aproximado: 34 600 ns (unos 34,6 microsegundos).**

**Apartado d: con caché L2**

La caché L2 tiene 1 MB, lo que es suficiente para contener todos los datos del programa (nota y media juntos ocupan solo 1 KB). Los 158 fallos de L1 que antes llegaban a MP con 200 ns de penalización ahora se resuelven en L2 en solo 15 ns.

- Aciertos en L1: 42 x 1 ns = 42 ns
- Fallos en L1 resueltos por L2: 158 x (1 + 15) ns = 2 528 ns
- Las escrituras con write-back y write-allocate se resuelven en L1 o L2 sin ir a MP.

**Tiempo total con L2: unos 2 600 ns (2,6 microsegundos), una mejora de aproximadamente 13 veces.**

---

Ejercicio 5

Enunciado: Computador con MP de 256 KB, caché de 128 B con bloques de 16 B. Código:

```c
int A[16][16];
int B[32];
int C[16][16];

for (i=0; i < 16; i++)
    C[0][i] = A[0][i] + B[4];
```

A se almacena en `0x10000`, y B y C a continuación. Variables `i`, `j` y punteros en registros.

a) Fallos con emplazamiento directo y asignación en escritura.
b) Fallos con asociativa por conjuntos de 2 vías, FIFO.
c) ¿Se puede reducir el número de errores sin modificar el programa ni el tamaño de la caché?
d) Tiempo de acceso con t_cache = 2 ns y penalización por fallo = 150 ns.

Resolución:

**Datos de la caché**

La caché tiene 128 B con bloques de 16 B, lo que da 8 bloques. El offset ocupa 4 bits y el índice 3 bits. Cada bloque almacena 4 enteros.

Los arrays se ubican así: A empieza en 0x10000 (1024 B, 64 bloques de MP), B en 0x10400 (128 B, 8 bloques) y C en 0x10480 (1024 B, 64 bloques). El código accede únicamente a la primera fila de A (4 bloques), a B[4] (1 bloque) y a la primera fila de C (4 bloques).

**Mapeo de índices en caché (emplazamiento directo)**

| Bloque         | Numero de bloque MP | Indice en cache |
|----------------|---------------------|----------------|
| A[0][0..3]     | 0x1000              | 0              |
| A[0][4..7]     | 0x1001              | 1              |
| A[0][8..11]    | 0x1002              | 2              |
| A[0][12..15]   | 0x1003              | 3              |
| B[4..7]        | 0x1041              | 1              |
| C[0][0..3]     | 0x1048              | 0              |
| C[0][4..7]     | 0x1049              | 1              |
| C[0][8..11]    | 0x104A              | 2              |
| C[0][12..15]   | 0x104B              | 3              |

El índice 0 tiene conflicto entre A y C. El índice 1 tiene triple conflicto entre A, B y C. Los índices 2 y 3 tienen conflicto entre A y C.

**Apartado a: emplazamiento directo con asignación en escritura**

Para las iteraciones i = 0 a 3 (índice 0 para A y C), en cada iteración se lee A[0][i], se lee B[4] y se escribe C[0][i]. El patrón para i = 0 es: fallo en A (carga A en el índice 0), fallo en B (carga B en el índice 1) y fallo en C con asignación (carga C en el índice 0, expulsa A). Para i = 1, 2 y 3: fallo en A (C ocupa el índice 0), acierto en B (sigue en el índice 1) y fallo en C. Esto da 3 fallos para i = 0 y 2 fallos para i = 1, 2, 3.

Para las iteraciones i = 4 a 7 (índice 1 para A, B y C), el triple conflicto en el índice 1 provoca 3 fallos en i = 4 y también en i = 5, 6, 7 por el thrashing entre los tres bloques.

Para los rangos i = 8..11 e i = 12..15 (índices 2 y 3), el patrón es similar al de i = 0..3: conflicto A-C en el índice correspondiente y B en el índice 1.

| Rango de i | Fallos por iteracion | Iteraciones | Total |
|------------|---------------------|------------|-------|
| 0          | 3                   | 1          | 3     |
| 1 a 3      | 2                   | 3          | 6     |
| 4 a 7      | 3                   | 4          | 12    |
| 8          | 3                   | 1          | 3     |
| 9 a 11     | 2                   | 3          | 6     |
| 12         | 2                   | 1          | 2     |
| 13 a 15    | 2                   | 3          | 6     |

**Total: 38 fallos.**

**Apartado b: asociativa por conjuntos de 2 vías, FIFO**

Con 2 vías hay 4 conjuntos. Los bloques de A[0][0..3] y C[0][0..3] que antes compartían el índice 0 ahora van al mismo conjunto pero, con 2 vías, pueden coexistir sin expulsarse mutuamente. Lo mismo ocurre con los índices 2 y 3. El único conflicto que persiste está en el conjunto correspondiente al índice 1, donde A, B y C siguen siendo 3 bloques compitiendo por 2 vías.

Para los conjuntos sin triple conflicto, A y C conviven y los fallos se reducen a uno por bloque al cargarse por primera vez. El número total de fallos baja a aproximadamente **15 a 20**.

**Apartado c: reducir fallos sin modificar el programa**

Sin cambiar el código fuente ni el tamaño de la caché, la única opción es modificar el layout en memoria de los arrays mediante padding en la declaración o mediante directivas del compilador que alineen los datos de forma que los bloques de A, B y C no caigan en los mismos índices. Aumentar la asociatividad a 2 vías (sin cambiar el tamaño total de la caché) resolvería la mayoría de los conflictos.

**Apartado d: tiempo de acceso**

Con 38 fallos, 2 ns de acceso en caché y 150 ns de penalización, y 48 accesos totales:

- Aciertos: 48 - 38 = 10 accesos x 2 ns = 20 ns
- Fallos: 38 x (2 + 150) ns = 5 776 ns

**Tiempo total: 5 796 ns, aproximadamente 5,8 microsegundos.**

---

### Problemas Adicionales

---

Ejercicio A1

Enunciado: Computador con memoria física de 32 KB y caché de 512 B con bloques de 128 B, emplazamiento directo.

a) Formato de la dirección para acceder a MP.
b) Rango de direcciones físicas de cada bloque de caché según la tabla dada.
c) Número de aciertos para las siguientes cadenas de referencias: 0x2080 a 0x209F, 0x2880 a 0x289F, 0x03F0 a 0x0410.

| Etiqueta | Bloque en cache |
|----------|----------------|
| 0x35     | 0              |
| 0x10     | 1              |
| 0x10     | 2              |
| 0x08     | 3              |

Resolución:

**Apartado a: formato de la dirección**

La memoria física tiene 32 KB, lo que implica 15 bits de dirección. La caché tiene 512 B con bloques de 128 B, lo que da 4 bloques en total. El offset ocupa 7 bits (log2(128)), el índice 2 bits (log2(4)) y la etiqueta los 6 bits restantes.

Formato: `[etiqueta: 6 bits | indice: 2 bits | offset: 7 bits]`

**Apartado b: rango de direcciones de cada bloque**

El número de bloque de MP de una entrada de caché con índice `i` y etiqueta `t` es `t x 4 + i`. La dirección base es ese número multiplicado por 128.

| Bloque en cache | Etiqueta | Numero de bloque MP | Direccion base | Rango           |
|----------------|----------|---------------------|----------------|-----------------|
| 0              | 0x35     | 53 x 4 + 0 = 212   | 0x6A00         | 0x6A00 a 0x6A7F |
| 1              | 0x10     | 16 x 4 + 1 = 65    | 0x2080         | 0x2080 a 0x20FF |
| 2              | 0x10     | 16 x 4 + 2 = 66    | 0x2100         | 0x2100 a 0x217F |
| 3              | 0x08     | 8 x 4 + 3 = 35     | 0x1180         | 0x1180 a 0x11FF |

**Apartado c: aciertos**

Serie 1 (0x2080 a 0x209F): estas 32 direcciones pertenecen al bloque de MP número 65, con índice 1 y etiqueta 0x10. El bloque 1 de la caché tiene exactamente esa etiqueta, así que todos los accesos son aciertos. Accediendo por palabras de 4 bytes: **8 aciertos**.

Serie 2 (0x2880 a 0x289F): la dirección 0x2880 corresponde al bloque de MP número 81, con índice 1 y etiqueta 0x14. El bloque 1 de la caché tiene etiqueta 0x10, que no coincide. El primer acceso es fallo. Una vez cargado el bloque, el resto son aciertos: **7 aciertos** (acceso por palabras de 4 bytes).

Serie 3 (0x03F0 a 0x0410): esta serie cruza un límite de bloque. Las direcciones 0x03F0 a 0x03FF pertenecen al bloque 7 (índice 3, etiqueta 0x01), que no coincide con la etiqueta 0x08 del bloque 3 de la caché. Las direcciones 0x0400 a 0x0410 pertenecen al bloque 8 (índice 0, etiqueta 0x02), que no coincide con la etiqueta 0x35 del bloque 0. **0 aciertos** en toda la serie.

---

Ejercicio A2

Enunciado: MP de 32 MB, caché de 2 KB con líneas de 256 B, prebúsqueda bajo fallo, reemplazo FIFO. Secuencia de accesos: `0x00023FA`, `0x00014A2`, `0x0003F02`, `0x00040B1`, `0x0005572`, `0x00023AA`.

a) Emplazamiento directo.
b) Asociativa por conjuntos de 2 vías.

Resolución:

**Datos de la caché**

La MP tiene 32 MB, con direcciones de 25 bits. La caché tiene 2 KB con líneas de 256 B, lo que da 8 bloques. Para emplazamiento directo: 8 bits de offset, 3 bits de índice y 14 bits de etiqueta. Para 2 vías: 4 conjuntos, 2 bits de índice y 15 bits de etiqueta. El número de bloque se obtiene desplazando 8 bits a la derecha.

| Direccion   | Numero de bloque | Indice (directo) | Etiqueta (directo) | Indice (2 vias) | Etiqueta (2 vias) |
|------------|-----------------|------------------|--------------------|----------------|-------------------|
| 0x00023FA  | 0x0023          | 3                | 0x0004             | 3              | 0x0008            |
| 0x00014A2  | 0x0014          | 4                | 0x0002             | 0              | 0x0005            |
| 0x0003F02  | 0x003F          | 7                | 0x0007             | 3              | 0x000F            |
| 0x00040B1  | 0x0040          | 0                | 0x0008             | 0              | 0x0010            |
| 0x0005572  | 0x0055          | 5                | 0x000A             | 1              | 0x0015            |
| 0x00023AA  | 0x0023          | 3                | 0x0004             | 3              | 0x0008            |

**Apartado a: emplazamiento directo**

Acceso 1 (0x00023FA, índice 3): vacío, **fallo**. Carga el bloque 0x23 en el índice 3 y por prebúsqueda el bloque 0x24 en el índice 4.

Acceso 2 (0x00014A2, índice 4): el índice 4 tiene el bloque 0x24 (etiqueta 0x0004), pero la dirección buscada tiene etiqueta 0x0002. **Fallo**. Carga el bloque 0x14 en el índice 4 y por prebúsqueda el bloque 0x15 en el índice 5.

Acceso 3 (0x0003F02, índice 7): vacío, **fallo**. Carga el bloque 0x3F en el índice 7 y por prebúsqueda el bloque 0x40 en el índice 0.

Acceso 4 (0x00040B1, índice 0): el índice 0 tiene el bloque 0x40 cargado por prebúsqueda, y la etiqueta coincide. **Acierto**.

Acceso 5 (0x0005572, índice 5): el índice 5 tiene el bloque 0x15 (prebúsqueda del acceso 2), pero la etiqueta buscada es 0x000A y no coincide. **Fallo**. Carga el bloque 0x55 y por prebúsqueda el bloque 0x56 en el índice 6.

Acceso 6 (0x00023AA, índice 3): el índice 3 tiene el bloque 0x23 del primer acceso con la misma etiqueta. **Acierto**.

Resultado: 4 fallos y 2 aciertos.

**Apartado b: 2 vías FIFO**

Acceso 1 (conjunto 3): vacío, **fallo**. Carga el bloque 0x23 en la vía 0. Prebúsqueda: el bloque 0x24 va al conjunto 0.

Acceso 2 (conjunto 0): el conjunto 0 tiene el bloque 0x24 (etiqueta 0x09), pero la etiqueta buscada es 0x05. **Fallo**. Carga el bloque 0x14 en la vía 1. Prebúsqueda: el bloque 0x15 va al conjunto 3 en la vía 1.

Acceso 3 (conjunto 3): el conjunto 3 tiene los bloques 0x23 (etiqueta 0x08) y 0x15 (etiqueta 0x05). La etiqueta buscada es 0x0F, no está. **Fallo**. FIFO expulsa el bloque 0x23. Entra el bloque 0x3F. Prebúsqueda: el bloque 0x40 va al conjunto 0, donde FIFO expulsa el bloque 0x24.

Acceso 4 (conjunto 0): el conjunto 0 tiene el bloque 0x14 (etiqueta 0x05) y el bloque 0x40 (etiqueta 0x10). La etiqueta buscada es 0x10, coincide. **Acierto**.

Acceso 5 (conjunto 1): vacío, **fallo**. Se cargan los bloques 0x55 y 0x56.

Acceso 6 (conjunto 3): el conjunto 3 tiene los bloques 0x15 y 0x3F, ninguno con la etiqueta 0x08. **Fallo**.

Resultado: 5 fallos y 1 acierto.

En este caso la organización directa resulta mejor (4 fallos frente a 5) porque la prebúsqueda acertó en el acceso 4, mientras que en el caso de 2 vías el bloque 0x23 fue expulsado antes de su segundo acceso.

---

Ejercicio A3

Enunciado: Caché de 256 B con bloques de 16 B. Secuencia de accesos: `0xA01`, `0xB1F`, `0x70A`, `0x60F`, `0xA70`, `0xB11`, `0xA7A`, `0xA0B`, `0x67A`, `0xA7F`, `0x071`, `0x67F`.

a) Organización directa: indicar fallos y reemplazamientos.
b) Asociativa 2 vías con LRU: indicar reemplazamientos.

Resolución:

**Datos de la caché**

La caché tiene 256 B con bloques de 16 B, lo que da 16 bloques. El offset ocupa 4 bits y el índice también 4 bits (bits 7:4 de la dirección). El número de bloque es la dirección desplazada 4 bits a la derecha, y el índice es ese número módulo 16.

| Direccion | Numero de bloque | Indice | Etiqueta |
|-----------|-----------------|--------|----------|
| 0xA01     | 0xA0            | 0      | 0xA      |
| 0xB1F     | 0xB1            | 1      | 0xB      |
| 0x70A     | 0x70            | 0      | 0x7      |
| 0x60F     | 0x60            | 0      | 0x6      |
| 0xA70     | 0xA7            | 7      | 0xA      |
| 0xB11     | 0xB1            | 1      | 0xB      |
| 0xA7A     | 0xA7            | 7      | 0xA      |
| 0xA0B     | 0xA0            | 0      | 0xA      |
| 0x67A     | 0x67            | 7      | 0x6      |
| 0xA7F     | 0xA7            | 7      | 0xA      |
| 0x071     | 0x07            | 7      | 0x0      |
| 0x67F     | 0x67            | 7      | 0x6      |

**Apartado a: organización directa**

| Num | Direccion | Indice | Etiqueta en cache | Resultado         |
|-----|-----------|--------|-------------------|-------------------|
| 1   | 0xA01     | 0      | vacio             | Fallo             |
| 2   | 0xB1F     | 1      | vacio             | Fallo             |
| 3   | 0x70A     | 0      | 0xA               | Fallo (reemplazo) |
| 4   | 0x60F     | 0      | 0x7               | Fallo (reemplazo) |
| 5   | 0xA70     | 7      | vacio             | Fallo             |
| 6   | 0xB11     | 1      | 0xB               | Acierto           |
| 7   | 0xA7A     | 7      | 0xA               | Acierto           |
| 8   | 0xA0B     | 0      | 0x6               | Fallo (reemplazo) |
| 9   | 0x67A     | 7      | 0xA               | Fallo (reemplazo) |
| 10  | 0xA7F     | 7      | 0x6               | Fallo (reemplazo) |
| 11  | 0x071     | 7      | 0xA               | Fallo (reemplazo) |
| 12  | 0x67F     | 7      | 0x0               | Fallo (reemplazo) |

Resultado: 10 fallos (7 con reemplazamiento) y 2 aciertos.

**Apartado b: asociativa 2 vías con LRU**

Con 2 vías hay 8 conjuntos. El índice pasa a ser el número de bloque módulo 8.

| Num | Conjunto | Etiqueta buscada | Estado del conjunto                               | Resultado         |
|-----|---------|-----------------|--------------------------------------------------|-------------------|
| 1   | 0       | 0x14            | vacio, entra 0x14                                | Fallo             |
| 2   | 1       | 0x16            | vacio, entra 0x16                                | Fallo             |
| 3   | 0       | 0x0E            | tiene 0x14, entra 0x0E en la segunda via         | Fallo             |
| 4   | 0       | 0x0C            | tiene 0x14 y 0x0E, LRU expulsa 0x14, entra 0x0C | Fallo (reemplazo) |
| 5   | 7       | 0x14            | vacio, entra 0x14                                | Fallo             |
| 6   | 1       | 0x16            | tiene 0x16, coincide                             | Acierto           |
| 7   | 7       | 0x14            | tiene 0x14, coincide                             | Acierto           |
| 8   | 0       | 0x14            | tiene 0x0E y 0x0C, LRU expulsa 0x0E, entra 0x14 | Fallo (reemplazo) |
| 9   | 7       | 0x0C            | tiene 0x14, entra 0x0C en la segunda via         | Fallo             |
| 10  | 7       | 0x14            | tiene 0x14 y 0x0C, 0x14 pasa a MRU              | Acierto           |
| 11  | 7       | 0x00            | tiene 0x0C y 0x14, LRU expulsa 0x0C, entra 0x00 | Fallo (reemplazo) |
| 12  | 7       | 0x0C            | tiene 0x14 y 0x00, LRU expulsa 0x14, entra 0x0C | Fallo (reemplazo) |

Resultado: 9 fallos (5 con reemplazamiento) y 3 aciertos. La asociatividad de 2 vías reduce los reemplazamientos de 7 a 5 y añade un acierto extra respecto al emplazamiento directo.

---

Ejercicio A4

Enunciado: MP de 128 MB, caché de 1 MB con líneas de 256 B (emplazamiento directo) y caché víctima de 1 KB completamente asociativa, FIFO. Caché inicialmente vacía. Secuencia de accesos: `0x0334500`, `0x014BF00`, `0x4134501`, `0x124BF00`, `0x03345FE`, `0x34345AC`, `0x214BFAA`, `0x014BF54`.

Resolución:

**Datos de la caché**

La MP tiene 128 MB, con direcciones de 27 bits. La caché L1 tiene 1 MB con líneas de 256 B, lo que da 4096 bloques. El offset ocupa 8 bits, el índice 12 bits y la etiqueta 7 bits. La caché víctima tiene 1024 / 256 = 4 entradas.

El número de bloque es la dirección desplazada 8 bits. El índice es ese número módulo 4096 y la etiqueta los bits por encima del índice.

| Direccion   | Numero de bloque | Indice | Etiqueta |
|------------|-----------------|--------|----------|
| 0x0334500  | 0x03345         | 0x0345 | 0x3      |
| 0x014BF00  | 0x014BF         | 0x04BF | 0x1      |
| 0x4134501  | 0x41345         | 0x0345 | 0x41     |
| 0x124BF00  | 0x124BF         | 0x04BF | 0x12     |
| 0x03345FE  | 0x03345         | 0x0345 | 0x3      |
| 0x34345AC  | 0x34345         | 0x0345 | 0x34     |
| 0x214BFAA  | 0x214BF         | 0x04BF | 0x21     |
| 0x014BF54  | 0x014BF         | 0x04BF | 0x1      |

**Traza de accesos**

Acceso 1 (índice 0x0345, etiqueta 0x3): caché vacía, **fallo**. Entra la etiqueta 0x3. Víctima sin cambios.

Acceso 2 (índice 0x04BF, etiqueta 0x1): ese índice está vacío, **fallo**. Entra la etiqueta 0x1.

Acceso 3 (índice 0x0345, etiqueta 0x41): el índice tiene la etiqueta 0x3, que no coincide. Tampoco está en la víctima (vacía). **Fallo**. La etiqueta 0x3 se expulsa a la víctima (posición 0). Entra 0x41. Víctima: [0x3].

Acceso 4 (índice 0x04BF, etiqueta 0x12): el índice tiene la etiqueta 0x1, que no coincide. No está en la víctima. **Fallo**. La etiqueta 0x1 se expulsa a la víctima (posición 1). Entra 0x12. Víctima: [0x3, 0x1].

Acceso 5 (índice 0x0345, etiqueta 0x3): el índice tiene la etiqueta 0x41, que no coincide. Pero la caché víctima tiene la etiqueta 0x3. Se produce un intercambio: 0x3 pasa de la víctima a la caché, y 0x41 se expulsa de la caché a la víctima (FIFO en la víctima expulsa 0x3 para hacer hueco). Este acceso se resuelve **en la víctima**, sin ir a MP. Víctima: [0x1, 0x41].

Acceso 6 (índice 0x0345, etiqueta 0x34): el índice ahora tiene la etiqueta 0x3, que no coincide. No está en la víctima (que tiene 0x1 y 0x41). **Fallo**. La etiqueta 0x3 pasa a la víctima (FIFO expulsa 0x1). Entra 0x34. Víctima: [0x41, 0x3].

Acceso 7 (índice 0x04BF, etiqueta 0x21): el índice tiene la etiqueta 0x12, que no coincide. No está en la víctima. **Fallo**. La etiqueta 0x12 pasa a la víctima (FIFO expulsa 0x41). Entra 0x21. Víctima: [0x3, 0x12].

Acceso 8 (índice 0x04BF, etiqueta 0x1): el índice tiene la etiqueta 0x21, que no coincide. La víctima tiene 0x3 y 0x12, tampoco coincide 0x1. **Fallo**. La etiqueta 0x21 pasa a la víctima (FIFO expulsa 0x3). Entra 0x1. Víctima: [0x12, 0x21].

**Resumen**

| Acceso      | Resultado          |
|------------|--------------------|
| 0x0334500  | Fallo en MP        |
| 0x014BF00  | Fallo en MP        |
| 0x4134501  | Fallo en MP        |
| 0x124BF00  | Fallo en MP        |
| 0x03345FE  | Acierto en victima |
| 0x34345AC  | Fallo en MP        |
| 0x214BFAA  | Fallo en MP        |
| 0x014BF54  | Fallo en MP        |

7 fallos en MP y 1 acierto en la caché víctima.

---

Ejercicio A5

Enunciado: MP de 4 MB, caché de 2 KB con líneas de 512 B, emplazamiento directo, asignación en escritura. Código:

```c
for (i=0; i<1024; i++) {
    C[i] = A[i] - B[1023-i];
}
```

Arrays de enteros almacenados desde `0x200000`.

a) ¿Cuántos fallos de caché se producen?
b) ¿Cuántos fallos con fusión de arrays? ¿Importa el orden?
c) ¿Cuántos fallos con alargamiento de arrays? ¿Importa la cantidad?

Resolución:

**Datos de la caché**

La caché tiene 2 KB con líneas de 512 B, lo que da 4 bloques en total. El offset ocupa 9 bits y el índice 2 bits. Cada bloque almacena 128 enteros. Cada array tiene 1024 elementos (4096 B = 8 bloques de MP).

Los tres arrays parten de 0x200000. El índice en caché de un bloque es su número de bloque módulo 4. A tiene bloques con índices 0, 1, 2, 3, 0, 1, 2, 3. B empieza justo después y tiene los mismos índices. C también. Los tres arrays mapean exactamente a los mismos cuatro índices.

**Apartado a: fallos sin optimización**

En cada iteración se lee A[i], se lee B[1023-i] y se escribe C[i]. A[i] y C[i] siempre están en el mismo índice. El thrashing entre A y C es constante: al leer A se expulsa C (o lo que hubiera), y al escribir C con asignación se expulsa A. Para cada grupo de 128 iteraciones que comparten el mismo bloque de A y C, esto produce aproximadamente 2 fallos por iteración para esos dos arrays más los fallos de B.

Total estimado: **2048 fallos** para A y C (thrashing constante) más **8 fallos** para B (uno por bloque).

**Total aproximado: 2056 fallos.**

**Apartado b: fusión de arrays**

Se mezclan A y C en una misma estructura para que queden en el mismo bloque de caché:

```c
struct { int a; int c; } AC[1024];
int B[1024];
```

Ahora `AC[i].a` y `AC[i].c` son contiguos (8 bytes por par, 64 pares por bloque). Un solo fallo trae ambos valores. Los fallos se reducen a:

- Fallos en AC: 1024 / 64 = 16 bloques x 1 fallo por bloque = **16 fallos**.
- Fallos en B: 8 bloques x 1 fallo = **8 fallos**.

**Total con fusión: 24 fallos.**

Sí importa el orden de fusión: fusionar A con C (los que compiten por el mismo índice) elimina el thrashing. Fusionar A con B no ayudaría, porque C seguiría en conflicto con el array combinado AB.

**Apartado c: alargamiento de arrays**

Se añaden elementos de relleno para desplazar C a un índice de caché distinto al de A:

```c
int A[1024];
int _pad[128]; // 128 enteros = 512 B = 1 bloque de desplazamiento
int B[1024];
int C[1024];
```

Con 128 enteros de padding, C se desplaza 1 bloque en memoria y su índice base pasa a ser (k + 1) % 4 en lugar de k % 4. Esto rompe el conflicto entre A y C en la mayoría de las iteraciones.

- Fallos en A: 8 bloques x 1 fallo = **8 fallos**.
- Fallos en B: 8 bloques x 1 fallo = **8 fallos**.
- Fallos en C: 8 bloques x 1 fallo = **8 fallos**.

**Total con alargamiento: 24 fallos.**

Sí importa la cantidad de padding. Si se añaden exactamente 512 enteros (el tamaño de la caché expresado en enteros), el desplazamiento en índices sería 0 y no habría mejora alguna. El padding debe provocar un desplazamiento de al menos 1 posición en el índice de caché, lo que equivale a al menos 128 enteros (un bloque de 512 B).

---

Ejercicio A6

Enunciado: Caché de 4 KB, bloques de 16 B, emplazamiento directo, asignación en escritura. Código:

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

a) ¿Cuántos fallos se producen?
b) ¿Qué asociatividad mínima elimina todos los fallos por conflicto?
c) ¿Qué modificaciones de código eliminan todos los fallos por conflicto con emplazamiento directo?

Resolución:

**Datos de la caché**

La caché tiene 4 KB con bloques de 16 B, lo que da 256 bloques. El offset ocupa 4 bits y el índice 8 bits. Cada bloque almacena 4 enteros. Cada array tiene 1024 elementos (4096 B = 256 bloques de MP).

A parte de 0x0C000000 y sus 256 bloques tienen índices 0x00 a 0xFF. B empieza exactamente 256 bloques después, con los mismos índices. C también. Los tres arrays mapean exactamente a los mismos 256 índices de caché.

**Apartado a: fallos**

En el bucle interior (j de 0 a 1023), se leen A[j] y B[j] (que compiten por el mismo índice j/4) y se acumula en C[i] (que está en el índice i/4, fijo para cada vuelta del bucle exterior).

El thrashing entre A y B es constante: leer A[j] expulsa B[j] del mismo índice y leer B[j] expulsa A[j]. Esto produce 2 fallos por cada par de accesos a A y B. En las 4 iteraciones j donde A[j], B[j] y C[i] coinciden en el mismo índice, el thrashing es triple.

Para cada una de las 10 vueltas del bucle exterior, el bucle interior produce aproximadamente 1024 x 2 = 2048 fallos.

**Total aproximado: 20 480 fallos.**

**Apartado b: asociatividad mínima**

Para que A[j] y B[j] puedan estar en caché simultáneamente sin expulsarse, hacen falta al menos **2 vías**. Para que además C[i] pueda coexistir con ambos en el mismo conjunto, hacen falta **3 vías**.

La asociatividad mínima para eliminar todos los fallos por conflicto es **3 vías**.

**Apartado c: modificaciones de código**

La solución más directa con emplazamiento directo es fusionar los arrays A y B, que son los que más conflictos generan:

```c
struct { int a; int b; } AB[1024];
int C[1024];

for (i=0; i<10; i++) {
    for (j=0; j<1024; j++) {
        C[i] += AB[j].a * AB[j].b;
    }
}
```

Con esta estructura, `AB[j].a` y `AB[j].b` están en el mismo bloque de caché. Un solo fallo trae ambos valores y el thrashing entre A y B desaparece. C accede a un único elemento C[i] por vuelta del bucle exterior, por lo que su impacto en los conflictos es mínimo.

---

Ejercicio A7

Enunciado: Multiplicación de matrices 128 x 128:

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

a) ¿Qué matriz presenta peor localidad? ¿Se puede mejorar con intercambio de bucles? ¿Empeora alguna otra?
b) Aplicar loop tiling y discutir si reduce fallos.

Resolución:

**Apartado a: localidad de las matrices**

Con almacenamiento por filas (row-major), los elementos contiguos en memoria son los que comparten la misma fila.

**A[i][k]** se accede con i fijo (fijado por el bucle exterior) y k variando en el bucle más interno. Esto es un acceso secuencial por filas, que aprovecha bien la localidad espacial.

**B[k][j]** se accede con j fijo y k variando en el bucle más interno. Como j es el índice de columna y k el de fila, los accesos saltan de fila en fila: B[0][j], B[1][j], B[2][j]... Cada acceso consecutivo está a 128 enteros de distancia en memoria. Esto es acceso por columnas y tiene pésima localidad espacial: cada acceso provoca un fallo de caché.

**C[i][j]** se lee y escribe con i y j fijos durante todo el bucle k. Es el mismo elemento en cada iteración, con localidad temporal perfecta (se puede mantener en un registro).

**B es la matriz con peor localidad**, ya que sus accesos son por columnas y cada elemento accedido consecutivamente genera un nuevo fallo.

Con el intercambio de los bucles j y k:

```c
for (i=0; i<128; i++) {
    for (k=0; k<128; k++) {
        for (j=0; j<128; j++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
```

Ahora B[k][j] se accede con k fijo y j variando, lo que es acceso secuencial por filas de B. La localidad de B mejora drásticamente. A[i][k] pasa a ser un escalar constante para todo el bucle interior (i y k fijos), que el compilador puede mantener en un registro. C[i][j] varía con j, lo que también es acceso por filas.

El intercambio no empeora ninguna de las otras matrices. Es una transformación de mejora pura.

**Apartado b: loop tiling**

El loop tiling (también llamado blocking) divide los bucles en bloques de tamaño T para que los datos activos en cada fase quepan completamente en caché:

```c
for (i=0; i<128; i+=T)
    for (j=0; j<128; j+=T)
        for (k=0; k<128; k+=T)
            for (ii=i; ii<i+T; ii++)
                for (jj=j; jj<j+T; jj++)
                    for (kk=k; kk<k+T; kk++)
                        C[ii][jj] += A[ii][kk] * B[kk][jj];
```

Con el código original, la fila entera de B necesaria para calcular C[i][j] se carga y descarta antes de que pueda reutilizarse para calcular C[i][j+1]. Con tiling, la submatriz B[k..k+T][j..j+T] se carga una sola vez y se reutiliza para todos los valores de i en el bloque externo.

Si T se elige de forma que tres submatrices de T x T enteros quepan en caché (una de A, una de B y una de C), todos los accesos dentro del tile serán aciertos tras la carga inicial.

**El loop tiling reduce significativamente los fallos**, especialmente en B, y se combina muy bien con el intercambio de bucles del apartado anterior.

---

Ejercicio A8

Enunciado: Máquina con direcciones de 16 bits, caché de datos de acceso directo con 4 bloques de 16 B, write-back con asignación en escritura. La variable `tmp` está en registro. Código:

```c
int v[16];    // direccion 0x0000
int mask[4];  // direccion 0x0040

for (i=0; i<12; i++) {
    tmp = 0;
    for (j=0; j<4; j++) {
        tmp = tmp + mask[j] * v[i];
    }
    v[i] = tmp;
}
```

a) ¿Cuántos fallos se producen? ¿Cuántos reemplazamientos y en qué iteraciones?
b) Recalcular con caché asociativa 2 vías con LRU.
c) ¿Qué etiqueta tendrá el bloque 2 de la caché tras la ejecución?

Resolución:

**Datos de la caché**

La caché tiene 4 bloques de 16 B, con direcciones de 16 bits. El offset ocupa 4 bits (bits 3:0), el índice 2 bits (bits 5:4) y la etiqueta los 10 bits restantes (bits 15:6). Cada bloque almacena 4 enteros.

Los arrays se ubican así:

- v[0..3]: bloque de MP 0, índice 0 en caché, etiqueta 0x000.
- v[4..7]: bloque 1, índice 1, etiqueta 0x000.
- v[8..11]: bloque 2, índice 2, etiqueta 0x000.
- v[12..15]: bloque 3, índice 3, etiqueta 0x000.
- mask[0..3]: bloque 4 (0x0040 / 16 = 4), índice 4 % 4 = 0, etiqueta 0x001.

mask y v[0..3] comparten el índice 0, lo que provoca conflicto directo.

**Apartado a: emplazamiento directo, write-back, write-allocate**

Para i = 0 a 3 (v[i] en el índice 0), en cada iteración el bucle interior accede 4 veces a mask[j] y 4 veces a v[i]. Como mask y v[0..3] están en el mismo índice, se expulsan mutuamente en cada acceso. El patrón en cada valor de j es: fallo en mask (expulsa v, que puede estar marcado como dirty y se escribe a MP), fallo en v (expulsa mask). Con 4 valores de j, esto produce 8 fallos por iteración. El store a v[i] al final de cada i es acierto (v ya está en caché del último acceso). Esto da 4 x 8 = **32 fallos** para i = 0..3.

Los reemplazamientos ocurren en cada uno de estos 32 fallos, ya que el índice 0 siempre tiene un bloque que expulsar. Son 32 reemplazamientos solo en este rango.

Para i = 4 a 7 (v[4..7] en el índice 1), mask está en el índice 0 y v en el índice 1. Ya no hay conflicto entre ellos. En i = 4 hay 2 fallos: uno al cargar mask por primera vez en el índice 0 (que expulsa lo que hubiera, 1 reemplazamiento) y otro al cargar el bloque v[4..7] en el índice 1 (vacío, sin reemplazamiento). Para i = 5, 6, 7, ambos bloques siguen en caché y todos los accesos son aciertos. Esto da **2 fallos** y 1 reemplazamiento para todo el rango i = 4..7.

El mismo razonamiento aplica para i = 8..11 (índice 2) e i = 12..15 (índice 3): **2 fallos** y 1 reemplazamiento por cada rango.

**Total: 32 + 2 + 2 + 2 = 38 fallos y aproximadamente 35 reemplazamientos.**

Los reemplazamientos se concentran en las iteraciones i = 0 a 3 (8 reemplazamientos por iteración, 32 en total) y en las primeras iteraciones de cada nuevo bloque de v (i = 4, 8, 12).

**Apartado b: caché asociativa 2 vías con LRU**

Con 2 vías hay 2 conjuntos. El índice pasa a ser 1 bit (bit 4). Los bloques se mapean así:

- v[0..3] y mask: conjunto 0 (bloques 0 y 4, ambos con índice 0 en la nueva numeración).
- v[4..7] y v[12..15]: conjunto 1.
- v[8..11]: conjunto 0.

En el conjunto 0, mask y v[0..3] tienen etiquetas distintas pero pueden coexistir en las 2 vías sin expulsarse. Para i = 0, el primer acceso a mask falla (entra en la vía 0) y el primer acceso a v[0] falla (entra en la vía 1). A partir de ahí, todos los accesos del bucle j son aciertos. El store a v[0] es acierto. Para i = 1, 2, 3, ambos bloques siguen en las 2 vías: 0 fallos por iteración. Total para i = 0..3: **2 fallos**.

Para i = 4 (conjunto 1), hay un fallo al cargar v[4..7]. mask sigue en el conjunto 0 y es acierto. Total: **1 fallo**. Para i = 5, 6, 7: 0 fallos.

Para i = 8 (conjunto 0), v[8..11] tiene una etiqueta diferente a la de v[0..3] y no está en las 2 vías (que tienen mask y v[0..3]). LRU expulsa v[0..3] (no usado desde i = 3). Se carga v[8..11]. Total: **1 fallo**. Para i = 9, 10, 11: 0 fallos.

Para i = 12 (conjunto 1), v[12..15] no está (el conjunto 1 tiene v[4..7]). LRU expulsa v[4..7]. **1 fallo**. Para i = 13, 14, 15: 0 fallos.

**Total con 2 vías: 2 + 1 + 1 + 1 = 5 fallos.** La mejora respecto al emplazamiento directo es muy significativa (de 38 a 5 fallos).

**Apartado c: etiqueta del bloque 2 de la caché tras la ejecución**

El bloque físico 2 de la caché tiene índice 2. El último bloque de MP cargado en ese índice fue v[8..11], durante las iteraciones i = 8 a 11. Ese bloque tiene la dirección base 0x0020 (bloque número 2 de MP, es decir, 2 x 16 = 32 = 0x0020).

La etiqueta se calcula tomando los bits [15:6] de la dirección base. En binario, 0x0020 es 0000 0000 0010 0000. Los bits 15:6 valen 0000 0000 00, que corresponde a **la etiqueta 0x000**.

La etiqueta del bloque 2 al final de la ejecución es **0x000**, correspondiente al bloque de datos de v[8..11].
