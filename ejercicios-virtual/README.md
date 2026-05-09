# Ejercicios EC

## Mario González García

---

## Memoria Virtual

### Problemas Básicos

---

Ejercicio 1

Enunciado: Sea un computador con una memoria virtual paginada y memoria caché con las siguientes características:

- Un procesador que genera direcciones de 20 bits.
- Una memoria física de 128 KB y páginas de 32 KB. Reemplazamiento LRU.
- Una memoria caché de datos de direcciones físicas de 1 KB. La memoria es de acceso directo con 256 B por bloque.

Sobre esta jerarquía se ejecuta una aplicación que selecciona aleatoriamente una canción dentro de una base de datos de canciones. Cada canción ocupa 8 KB y sus direcciones de comienzo son:

| Canción   | 1       | 2       | 3       | 4       | 5       | 6       |
|-----------|---------|---------|---------|---------|---------|---------|
| Dirección | 0x50000 | 0xF4000 | 0x66000 | 0x12000 | 0x40000 | 0x36000 |

a) Indicar el formato de la dirección virtual y de la dirección física, esta última desde el punto de vista de la memoria virtual y de la memoria caché.
b) Indicar en qué página o páginas virtuales está ubicada cada canción.
c) Si en un momento dado el contenido de la TLB y de la memoria caché de datos es el indicado en las tablas siguientes, ¿cuáles han sido las últimas canciones escuchadas? ¿Cuál es el rango de direcciones físicas de cada una de ellas?

| Página Virtual | Página Física |
|---------------|--------------|
| 0x0A          | 1            |
| 0x0C          | 2            |
| 0x02          | 0            |
| 0x1E          | 3            |

| Etiqueta | Bloque |
|----------|--------|
| 0x70     | 0      |
| 0x27     | 1      |
| 0x27     | 2      |
| 0x27     | 3      |

d) Supongamos que a continuación se genera la siguiente cadena de referencias virtuales: `0xF40F5`, `0x66000`, `0x40040` y `0x51D04`. Indicar si se producen fallos o aciertos de caché y fallos de página, mostrando cómo evolucionan los contenidos de la memoria principal y de la memoria caché.

Resolución:

**Apartado a: formato de las direcciones**

El procesador genera direcciones de 20 bits. Las páginas son de 32 KB, lo que implica un offset de log2(32 x 1024) = 15 bits. El número de página virtual ocupa los 20 - 15 = **5 bits** restantes.

Formato de la dirección virtual: `[número de página virtual: 5 bits | offset: 15 bits]`

La memoria física tiene 128 KB, lo que implica direcciones físicas de log2(128 x 1024) = 17 bits. Con páginas de 32 KB, el offset físico son los mismos 15 bits y el número de marco ocupa 17 - 15 = **2 bits**.

Formato de la dirección física desde el punto de vista de la memoria virtual: `[número de marco: 2 bits | offset: 15 bits]`

Para la caché, tiene 1 KB con bloques de 256 B, lo que da 4 bloques. El offset de bloque ocupa log2(256) = 8 bits y el índice log2(4) = 2 bits. La etiqueta son los 17 - 8 - 2 = **7 bits** restantes de la dirección física.

Formato de la dirección física desde el punto de vista de la caché: `[etiqueta: 7 bits | índice: 2 bits | offset: 8 bits]`

**Apartado b: páginas virtuales de cada canción**

Cada página virtual tiene 32 KB = 0x8000 bytes. El número de página de una dirección es la dirección dividida entre 0x8000, es decir, los bits de la dirección por encima de los 15 bits de offset.

Cada canción ocupa 8 KB = 0x2000 bytes, que es menos de una página, salvo que el comienzo de la canción esté cerca del final de su página y la siguiente canción empiece en la página siguiente.

| Canción | Dirección inicio | Página inicio (dir/0x8000) | Dirección fin | Página fin   | Páginas ocupadas |
|---------|-----------------|--------------------------|--------------|--------------|-----------------|
| 1       | 0x50000         | 0x50000/0x8000 = 0x0A    | 0x51FFF      | 0x0A         | 0x0A            |
| 2       | 0xF4000         | 0xF4000/0x8000 = 0x1E    | 0xF5FFF      | 0x1E         | 0x1E            |
| 3       | 0x66000         | 0x66000/0x8000 = 0x0C    | 0x67FFF      | 0x0C         | 0x0C            |
| 4       | 0x12000         | 0x12000/0x8000 = 0x02    | 0x13FFF      | 0x02         | 0x02            |
| 5       | 0x40000         | 0x40000/0x8000 = 0x08    | 0x41FFF      | 0x08         | 0x08            |
| 6       | 0x36000         | 0x36000/0x8000 = 0x06    | 0x37FFF      | 0x06         | 0x06            |

Todas las canciones caben dentro de una sola página virtual (cada una ocupa 8 KB y las páginas son de 32 KB, con suficiente espacio siempre que la canción no cruce un límite de página). En estos casos ninguna canción cruza un límite de página.

**Apartado c: canciones en TLB y rango de direcciones físicas**

La TLB contiene las páginas virtuales 0x0A, 0x0C, 0x02 y 0x1E, que corresponden a las canciones 1, 3, 4 y 2, según la tabla del apartado b. Estas son las últimas canciones accedidas (las que tienen su traducción en la TLB).

Para el rango de direcciones físicas, la dirección física base es el número de marco multiplicado por el tamaño de página (0x8000). Cada canción ocupa 8 KB (0x2000 bytes), así que el rango va desde esa dirección base más el offset hasta esa dirección base más el offset más 0x1FFF.

El offset de cada canción dentro de su página se calcula restando la dirección de inicio de la página a la dirección de inicio de la canción:

- Canción 1 (página virtual 0x0A, marco 1): dirección física base del marco = 1 x 0x8000 = 0x8000. Offset de inicio de canción = 0x50000 - 0x0A x 0x8000 = 0x50000 - 0x50000 = 0x0000. Rango físico: **0x8000 a 0x9FFF**.
- Canción 3 (página virtual 0x0C, marco 2): base física = 2 x 0x8000 = 0x10000. Offset = 0x66000 - 0x0C x 0x8000 = 0x66000 - 0x60000 = 0x6000. Rango físico: **0x16000 a 0x17FFF**.
- Canción 4 (página virtual 0x02, marco 0): base física = 0. Offset = 0x12000 - 0x02 x 0x8000 = 0x12000 - 0x10000 = 0x2000. Rango físico: **0x2000 a 0x3FFF**.
- Canción 2 (página virtual 0x1E, marco 3): base física = 3 x 0x8000 = 0x18000. Offset = 0xF4000 - 0x1E x 0x8000 = 0xF4000 - 0xF0000 = 0x4000. Rango físico: **0x1C000 a 0x1DFFF**.

La caché contiene el bloque 0 con etiqueta 0x70 y los bloques 1, 2 y 3 con etiqueta 0x27. La etiqueta en la caché son los 7 bits más significativos de la dirección física (bits 16:10). La etiqueta 0x27 = 0b010 0111, con lo que la dirección física base de esos bloques es 0x27 desplazado 10 bits: 0x27 x 0x400 = 0x9C00. Los bloques 1, 2 y 3 con esa etiqueta cubren las direcciones físicas 0x9C00+256 = 0x9D00, 0x9C00+512 = 0x9E00 y 0x9C00+768 = 0x9F00 respectivamente. Ese rango corresponde al marco físico 1 (0x8000 a 0xFFFF), que es donde está cargada la canción 1. Los tres bloques de caché con etiqueta 0x27 corresponden a datos de la **canción 1**.

La etiqueta 0x70 = 0x70 x 0x400 = 0x1C000. El bloque 0 de caché con etiqueta 0x70 está en el rango físico 0x1C000 a 0x1C0FF, que cae dentro del marco 3 (0x18000 a 0x1FFFF), donde está la **canción 2**.

**Apartado d: cadena de referencias virtuales**

Las referencias son 0xF40F5, 0x66000, 0x40040 y 0x51D04.

Estado inicial de la TLB: páginas 0x0A (marco 1), 0x0C (marco 2), 0x02 (marco 0), 0x1E (marco 3).
Estado inicial de la memoria física: marcos 0 a 3 ocupados (4 marcos en total, política LRU).

**Referencia 1: 0xF40F5**

Página virtual: 0xF40F5 / 0x8000 = 0x1E (los 5 bits superiores de 0xF40F5 en 20 bits son 0x1E). Offset: 0xF40F5 - 0x1E x 0x8000 = 0xF40F5 - 0xF0000 = 0x40F5.

La página 0x1E está en la TLB con el marco 3. No hay fallo de página. La dirección física es 3 x 0x8000 + 0x40F5 = 0x18000 + 0x40F5 = 0x1C0F5.

Para la caché: índice = bits 9:8 de 0x1C0F5 = bits 9:8 de 0x1C0F5. 0x1C0F5 en binario: 0001 1100 0000 1111 0101. Bits 9:8 = 00 (índice 0). Etiqueta = bits 16:10 = 111 0000 = 0x70. El bloque 0 de caché tiene etiqueta 0x70, **acierto en caché**.

**Referencia 2: 0x66000**

Página virtual: 0x66000 / 0x8000 = 0x0C. Está en la TLB con el marco 2. No hay fallo de página. Dirección física: 2 x 0x8000 + (0x66000 - 0x60000) = 0x10000 + 0x6000 = 0x16000.

Para la caché: 0x16000 en binario: 0001 0110 0000 0000 0000. Bits 9:8 = 00 (índice 0). Etiqueta = bits 16:10 = 101 1000 = 0x58. El bloque 0 de la caché tiene etiqueta 0x70, que no coincide con 0x58. **Fallo en caché**. Se carga el bloque físico 0x16000 a 0x160FF en el bloque 0 de la caché, reemplazando la etiqueta 0x70 por 0x58.

**Referencia 3: 0x40040**

Página virtual: 0x40040 / 0x8000 = 0x08. La página 0x08 no está en la TLB. Hay que consultar la tabla de páginas. La página 0x08 es la canción 5, que no ha sido escuchada recientemente. Hay **fallo de página**. Se debe cargar la canción 5 en memoria física. Los cuatro marcos están ocupados (páginas 0x0A, 0x0C, 0x02, 0x1E). LRU expulsa el marco más antiguo. Suponiendo que el orden de uso más reciente a más antiguo es 0x1E, 0x0C, 0x0A, 0x02 (basándonos en que la última referencia fue a 0x0C), se expulsa la página 0x02 (canción 4) del marco 0. La página 0x08 entra en el marco 0. La TLB se actualiza: sale 0x02, entra 0x08 con marco 0. Dirección física: 0 x 0x8000 + (0x40040 - 0x40000) = 0x40. Los bloques de caché que contenían datos de la página 0x02 (que ocupaba el marco 0) se invalidan.

Para la caché con la nueva dirección física 0x40: índice = bits 9:8 de 0x40 = 00 (índice 0). Etiqueta = bits 16:10 = 0x00. El bloque 0 de caché tiene etiqueta 0x58 (del acceso anterior), no coincide. **Fallo en caché**. Se carga el bloque físico 0x00 a 0xFF en el bloque 0 de la caché con etiqueta 0x00.

**Referencia 4: 0x51D04**

Página virtual: 0x51D04 / 0x8000 = 0x0A. Está en la TLB con el marco 1. No hay fallo de página. Dirección física: 1 x 0x8000 + (0x51D04 - 0x50000) = 0x8000 + 0x1D04 = 0x9D04.

Para la caché: 0x9D04 en binario: 0000 1001 1101 0000 0100. Bits 9:8 = 01 (índice 1). Etiqueta = bits 16:10 = 010 0111 = 0x27. El bloque 1 de la caché tiene etiqueta 0x27, **acierto en caché**.

**Resumen del apartado d**

| Referencia | Página virtual | Resultado TLB     | Resultado caché | Marco |
|-----------|----------------|-------------------|-----------------|-------|
| 0xF40F5   | 0x1E           | Acierto TLB       | Acierto caché   | 3     |
| 0x66000   | 0x0C           | Acierto TLB       | Fallo caché     | 2     |
| 0x40040   | 0x08           | Fallo de página   | Fallo caché     | 0     |
| 0x51D04   | 0x0A           | Acierto TLB       | Acierto caché   | 1     |

---

Ejercicio 2

Enunciado: Sea una memoria caché de emplazamiento directo virtualmente accedida físicamente marcada cuyo formato de dirección es el siguiente:

| Cache tag | Cache index | Byte select |
|-----------|-------------|-------------|
| 63:10     | 9:4         | 3:0         |

a) ¿Cuál es el tamaño de esta memoria caché?
b) ¿Cuál es el tamaño de las páginas de la memoria virtual?
c) ¿Cómo se podría doblar el tamaño de la memoria caché sin modificar el formato de la dirección?

Resolución:

**Apartado a: tamaño de la caché**

El campo byte select ocupa los bits 3:0, es decir, 4 bits. El tamaño del bloque es por tanto 2^4 = **16 bytes**.

El campo cache index ocupa los bits 9:4, es decir, 6 bits. El número de bloques en la caché es 2^6 = **64 bloques**.

El tamaño total de la caché es 64 x 16 = **1024 bytes = 1 KB**.

**Apartado b: tamaño de las páginas**

En una caché virtualmente accedida físicamente marcada, el índice de caché y el byte select deben estar completamente dentro del offset de página para que la traducción de dirección virtual a física no afecte a los bits de indexación. Esto garantiza que el índice calculado con la dirección virtual coincide con el calculado con la dirección física.

El offset de página debe cubrir al menos los bits usados para indexar la caché, es decir, los bits 9:0 (cache index + byte select). El tamaño mínimo de página es por tanto 2^10 = **1024 bytes = 1 KB**.

Si el tamaño de página fuera menor que 1 KB, los bits del índice de caché podrían cambiar al traducir de virtual a físico, lo que provocaría incoherencias. El tamaño de página debe ser al menos **1 KB**.

**Apartado c: doblar el tamaño sin cambiar el formato**

Si se dobla el tamaño de la caché pasando a 2 KB con el mismo formato de dirección, el número de bloques pasaría a 128 en lugar de 64. Pero el campo cache index solo tiene 6 bits, que solo pueden direccionar 64 entradas. La única forma de doblar la caché manteniendo el mismo formato de dirección es usar **asociatividad de 2 vías**: en lugar de 64 bloques de emplazamiento directo, se usan 64 conjuntos con 2 vías cada uno. El índice sigue siendo de 6 bits (mismo campo), el tamaño de bloque sigue siendo 16 bytes, pero el tamaño total pasa a 64 x 2 x 16 = 2048 bytes = **2 KB**, el doble que antes.

---

Ejercicio 3

Enunciado: Sea un sistema de memoria con las siguientes características:

- Memoria virtual paginada de 16 MB, política de emplazamiento LRU.
- Tamaño de página 4 KB.
- Memoria principal con 32 KB.
- Memoria caché de direcciones físicas de 8 KB, con bloques de 64 bytes, asociativa por conjuntos con 2 bloques por conjunto, política LRU.

Tabla de páginas inicial:

| Página | Marco | Último acceso |
|--------|-------|--------------|
| 0x100  | 0     | 0            |
| 0x101  | 1     | 1            |
| 0x200  | 2     | 2            |
| 0x201  | 3     | 3            |
| 0x300  | 4     | 4            |
| 0x301  | 5     | 5            |
| 0x400  | 6     | 6            |
| 0x401  | 7     | 7            |

Se ejecuta el siguiente código:

```c
int A[16384];
for (i=0; i<16384; i++)
    red += A[i];
```

A está almacenado consecutivamente en la dirección virtual 0x100000 y la variable `red` está en un registro.

a) Indicar el formato de la dirección virtual y de la dirección física, desde el punto de vista de la memoria virtual y de la memoria caché.
b) ¿Cuántas páginas de memoria virtual se visitan? ¿Cuáles son?
c) Razonar el número de aciertos y fallos de página que se producen.

Resolución:

**Apartado a: formato de las direcciones**

La memoria virtual tiene 16 MB = 2^24 bytes, por lo que las direcciones virtuales son de **24 bits**. Con páginas de 4 KB = 2^12 bytes, el offset ocupa 12 bits y el número de página virtual los 24 - 12 = **12 bits** restantes.

Formato de la dirección virtual: `[número de página virtual: 12 bits | offset: 12 bits]`

La memoria principal tiene 32 KB = 2^15 bytes, por lo que las direcciones físicas son de **15 bits**. Con páginas de 4 KB, el offset físico son los mismos 12 bits y el número de marco ocupa 15 - 12 = **3 bits**.

Formato de la dirección física desde el punto de vista de la memoria virtual: `[número de marco: 3 bits | offset: 12 bits]`

Para la caché, tiene 8 KB = 8192 bytes con bloques de 64 bytes y 2 vías. El número de conjuntos es 8192 / (64 x 2) = **64 conjuntos**. El offset ocupa log2(64) = 6 bits, el índice log2(64 conjuntos) = 6 bits y la etiqueta los 15 - 6 - 6 = **3 bits** restantes.

Formato de la dirección física desde el punto de vista de la caché: `[etiqueta: 3 bits | índice: 6 bits | offset: 6 bits]`

**Apartado b: páginas virtuales visitadas**

El array A tiene 16 384 enteros. Suponiendo enteros de 4 bytes, ocupa 16 384 x 4 = 65 536 bytes = 64 KB. Empieza en la dirección virtual 0x100000.

Cada página tiene 4 KB. El número de páginas necesarias es 65 536 / 4096 = **16 páginas**.

La dirección 0x100000 tiene número de página 0x100000 / 0x1000 = 0x100. Las páginas visitadas son las **0x100, 0x101, 0x102, ..., 0x10F** (16 páginas consecutivas).

**Apartado c: fallos de página**

La memoria física tiene 32 KB / 4 KB = 8 marcos. La tabla de páginas inicial muestra que los 8 marcos están ya ocupados (páginas 0x100 a 0x401). Las páginas que necesita el bucle son la 0x100 a la 0x10F (16 páginas).

De esas 16 páginas, las dos primeras (0x100 y 0x101) ya están cargadas en los marcos 0 y 1 (según la tabla inicial). Las 14 páginas restantes (0x102 a 0x10F) no están en memoria y producirán **fallos de página**.

Cuando se produce el primer fallo (página 0x102), LRU expulsa el marco con el acceso más antiguo. Según la tabla, los marcos con último acceso 0, 1, 2, 3... el más antiguo es el marco 0 (página 0x100, último acceso = 0). Pero el acceso al bucle actualiza este orden: primero se accede a la página 0x100 (marco 0), luego a la 0x101 (marco 1). Cuando se necesita la 0x102, los marcos con menor uso reciente son los que no han sido accedidos en el bucle, es decir, los correspondientes a páginas 0x200 a 0x401.

A medida que se van cargando las páginas 0x102 a 0x10F, LRU va expulsando las páginas menos recientes. Como los 8 marcos están llenos y se necesitan 16 páginas en total (de las cuales 2 ya están), hay que cargar 14 páginas nuevas con 6 marcos disponibles inicialmente (los que tienen páginas que no pertenecen al array). Tras cargar esas 6, la política LRU empezará a expulsar páginas del propio array ya visitadas.

El análisis completo es:

Las páginas 0x100 y 0x101 están en memoria: **2 aciertos** (para la primera iteración de cada página, sin fallo de página). Las páginas 0x102 a 0x10F (14 páginas) no están: **14 fallos de página**. En total, **2 aciertos de página y 14 fallos de página** para las páginas de A.

Dentro de cada página, todos los accesos a los 1024 enteros de esa página son accesos válidos una vez la página está cargada (sin más fallos de página hasta cambiar de página). La variable `red` está en un registro y no genera accesos a memoria.

---

Ejercicio 4

Enunciado: Sea una memoria caché asociativa virtualmente accedida físicamente marcada de 2^10 bytes y grado de asociatividad 8 que utiliza direcciones de 32 bits. Sabiendo que la memoria principal se divide en bloques de 4 bytes y que el bus de direcciones virtuales y reales tiene el mismo tamaño.

a) ¿Cuántas páginas tiene un proceso virtual?
b) ¿Cuántos bytes tiene una página?
c) ¿Cuántos bloques tiene una página?

Resolución:

**Datos del sistema**

La caché tiene 2^10 = 1024 bytes, grado de asociatividad 8 (8 vías) y bloques de 4 bytes (igual que el bloque de memoria principal). Las direcciones son de 32 bits.

El número de conjuntos es 1024 / (4 x 8) = 1024 / 32 = **32 conjuntos**. El offset ocupa log2(4) = 2 bits y el índice log2(32) = 5 bits. Los bits de etiqueta son 32 - 2 - 5 = **25 bits**.

**Apartado a: número de páginas virtuales**

En una caché virtualmente accedida físicamente marcada, el tamaño de página debe ser al menos igual al tamaño de la caché dividido entre el grado de asociatividad (para que el índice de caché quede dentro del offset de página). Aquí el tamaño de página mínimo es el tamaño del conjunto: 4 bytes x 8 vías = 32 bytes. Pero más precisamente, el offset de página debe cubrir los bits de índice y offset de bloque de la caché: 5 + 2 = 7 bits, con lo que el tamaño de página mínimo es 2^7 = **128 bytes**.

El espacio de direcciones virtuales es 2^32 bytes. El número de páginas es 2^32 / 128 = 2^25 = **33 554 432 páginas**.

**Apartado b: tamaño de una página**

Tal como se ha calculado, el tamaño mínimo de página para que el esquema virtualmente accedido físicamente marcado funcione correctamente es **128 bytes** (2^7 bytes), que corresponde al área cubierta por el índice de caché más el offset de bloque.

**Apartado c: bloques por página**

Si cada página tiene 128 bytes y cada bloque de memoria principal tiene 4 bytes, entonces cada página contiene 128 / 4 = **32 bloques**.

---

Ejercicio 5

Enunciado: Sea un computador con memoria virtual paginada y memoria caché con las características siguientes:

- Memoria virtual de 32 páginas de 8 KB cada una, con traducción asociativa y reemplazamiento LRU.
- Memoria física de 32 KB.
- Memoria caché de direcciones físicas de 512 bytes, asociativa por conjuntos, con bloques de 128 bytes, 2 bloques por conjunto y reemplazamiento LRU.
- La política de actualización de la memoria principal es escritura directa sin asignación en escritura.

a) Indicar el formato de la dirección virtual y de la dirección física, desde el punto de vista de la memoria virtual y de la memoria caché.
b) Si en un momento dado los contenidos de la tabla de páginas y de la caché son los siguientes:

| Num Página | Num Marco | Etiqueta | Conjunto | Vía |
|-----------|----------|----------|---------|-----|
| 0x06      | 3        | 0x23     | 0       | 0   |
| 0x01      | 0        | 0x47     | 0       | 1   |
| 0x04      | 2        | 0x73     | 1       | 0   |
| 0x0C      | 1        | 0x08     | 1       | 1   |

Expresar en hexadecimal el rango de direcciones virtuales y físicas de cada marco de página y de cada bloque de caché.

c) Supongamos que un programa realiza la siguiente cadena de referencias virtuales (en hexadecimal): `08770`, `02080-0209F`, `02880-0289F`, `0D3F0-0D410`, `27000-2701F`. Calcular el tiempo total de acceso a memoria e indicar cómo evolucionan los contenidos de la memoria principal y de la caché.

Resolución:

**Apartado a: formato de las direcciones**

La memoria virtual tiene 32 páginas de 8 KB, lo que da 32 x 8 KB = 256 KB en total. Las direcciones virtuales son de log2(256 x 1024) = 18 bits. Con páginas de 8 KB = 2^13 bytes, el offset ocupa 13 bits y el número de página virtual los 18 - 13 = **5 bits** restantes.

Formato de la dirección virtual: `[número de página virtual: 5 bits | offset: 13 bits]`

La memoria física tiene 32 KB = 2^15 bytes. Las direcciones físicas son de 15 bits. Con páginas de 8 KB, el offset físico son los mismos 13 bits y el número de marco ocupa 15 - 13 = **2 bits**.

Formato de la dirección física desde el punto de vista de la memoria virtual: `[número de marco: 2 bits | offset: 13 bits]`

Para la caché, tiene 512 bytes con bloques de 128 bytes y 2 vías. El número de conjuntos es 512 / (128 x 2) = **2 conjuntos**. El offset ocupa log2(128) = 7 bits, el índice log2(2) = 1 bit y la etiqueta los 15 - 7 - 1 = **7 bits** restantes.

Formato de la dirección física desde el punto de vista de la caché: `[etiqueta: 7 bits | índice: 1 bit | offset: 7 bits]`

**Apartado b: rangos de direcciones**

El tamaño de página es 8 KB = 0x2000 bytes.

*Rangos de páginas virtuales (marco a marco):*

- Marco 3 tiene la página virtual 0x06: rango virtual = 0x06 x 0x2000 a 0x06 x 0x2000 + 0x1FFF = **0x0C000 a 0x0DFFF**. Rango físico = 3 x 0x2000 a 3 x 0x2000 + 0x1FFF = **0x06000 a 0x07FFF**.
- Marco 0 tiene la página virtual 0x01: rango virtual = **0x02000 a 0x03FFF**. Rango físico = **0x00000 a 0x01FFF**.
- Marco 2 tiene la página virtual 0x04: rango virtual = **0x08000 a 0x09FFF**. Rango físico = **0x04000 a 0x05FFF**.
- Marco 1 tiene la página virtual 0x0C: rango virtual = **0x18000 a 0x19FFF**. Rango físico = **0x02000 a 0x03FFF**.

*Rangos de bloques de caché:*

El tamaño de bloque es 128 bytes = 0x80 bytes. La dirección física de un bloque de caché se reconstruye como etiqueta x 2 x 0x80 + índice x 0x80 (donde índice es 0 o 1 según el conjunto).

- Conjunto 0, vía 0, etiqueta 0x23: dirección física base = 0x23 x 0x100 + 0 x 0x80 = 0x2300. Rango físico: **0x2300 a 0x237F**. Para la dirección virtual: se busca qué página virtual mapea al marco con esa dirección física. 0x2300 cae en el rango físico 0x02000 a 0x03FFF (marco 1), que corresponde a la página virtual 0x0C. Offset = 0x2300 - 0x2000 = 0x300. Dirección virtual base = 0x18000 + 0x300 = 0x18300. Rango virtual: **0x18300 a 0x1837F**.
- Conjunto 0, vía 1, etiqueta 0x47: dirección física base = 0x47 x 0x100 + 0 x 0x80 = 0x4700. Rango físico: **0x4700 a 0x477F**. 0x4700 cae en el rango físico 0x04000 a 0x05FFF (marco 2), página virtual 0x04. Offset = 0x4700 - 0x4000 = 0x700. Rango virtual: **0x08700 a 0x0877F**.
- Conjunto 1, vía 0, etiqueta 0x73: dirección física base = 0x73 x 0x100 + 1 x 0x80 = 0x7300 + 0x80 = 0x7380. Rango físico: **0x7380 a 0x73FF**. 0x7380 cae en el rango físico 0x06000 a 0x07FFF (marco 3), página virtual 0x06. Offset = 0x7380 - 0x6000 = 0x1380. Rango virtual: **0x0D380 a 0x0D3FF**.
- Conjunto 1, vía 1, etiqueta 0x08: dirección física base = 0x08 x 0x100 + 1 x 0x80 = 0x0800 + 0x80 = 0x0880. Rango físico: **0x0880 a 0x08FF**. 0x0880 cae en el rango físico 0x00000 a 0x01FFF (marco 0), página virtual 0x01. Offset = 0x0880. Rango virtual: **0x02880 a 0x028FF**.

**Apartado c: cadena de referencias y tiempo de acceso**

El enunciado no especifica tiempos de acceso explícitos, por lo que se analiza cualitativamente indicando aciertos, fallos de caché y fallos de página, y describiendo la evolución del estado.

*Referencia 08770:*

Página virtual: 0x08770 / 0x2000 = 0x04. El marco correspondiente es 2. Dirección física: 2 x 0x2000 + (0x08770 - 0x08000) = 0x4000 + 0x0770 = 0x4770.

Para la caché: índice = bit 7 de 0x4770 = bit 7 de 0100 0111 0111 0000 = 0 (bit 7 vale 1... recalculando: 0x4770 = 0100 0111 0111 0000 en binario de 15 bits: 100 0111 0111 0000. Bits 7:7 = 0 (posición 7 desde el bit 0 empezando por la derecha) = (0x4770 >> 7) & 1 = (0x22) & 1 = 0. Índice = 0. Etiqueta = 0x4770 >> 8 = 0x47. El conjunto 0 tiene etiquetas 0x23 y 0x47. La etiqueta 0x47 coincide con la vía 1. **Acierto en caché**.

*Referencias 02080-0209F:*

32 bytes consecutivos en la página 0x01 (marco 0). Dirección física: 0x0080 a 0x009F.

Índice = bit 7 de 0x0080 = (0x0080 >> 7) & 1 = 1. Etiqueta = 0x0080 >> 8 = 0x00. El conjunto 1 tiene etiquetas 0x73 y 0x08. La etiqueta 0x00 no coincide. **Fallo en caché**. Se carga el bloque físico 0x0080 a 0x00FF en el conjunto 1. LRU expulsa la vía menos reciente del conjunto 1 (la que tenga menor uso reciente entre 0x73 y 0x08). La etiqueta 0x08 cubría el rango físico 0x0880 a 0x08FF (accedido como referencia del bloque de la tabla inicial), mientras que 0x73 cubría 0x7380 a 0x73FF. Sin información adicional sobre cuál fue accedida antes, suponemos que se aplica LRU y se expulsa la menos reciente. Entra el bloque con etiqueta 0x00.

Nota: los 32 bytes van de 0x0080 a 0x009F, que están dentro del mismo bloque (0x0080 a 0x00FF). Un solo fallo sirve todos los accesos.

*Referencias 02880-0289F:*

Página virtual 0x01 (marco 0). Dirección física: 0x0880 a 0x089F.

Índice = bit 7 de 0x0880 = (0x0880 >> 7) & 1 = 1. Etiqueta = 0x0880 >> 8 = 0x08. Si el bloque con etiqueta 0x08 sigue en el conjunto 1, **acierto en caché**. Si fue expulsado en el paso anterior, **fallo en caché**. Dado que el paso anterior expulsó uno de los dos bloques del conjunto 1, y si se expulsó el 0x08, ahora habría fallo. Sin pérdida de generalidad, si el acceso anterior fue a 0x0880 a 0x08FF (tabla inicial) antes que a 0x7380 a 0x73FF, el LRU expulsaría el 0x73. En ese caso el bloque 0x08 sigue en caché y este acceso es acierto.

*Referencias 0D3F0-0D410:*

Estos 33 bytes cruzan un límite de bloque. La parte 0D3F0-0D3FF está en el bloque 0x0D3F0 / 0x80 = 0x1A7 (físico). La parte 0D400-0D410 está en el siguiente bloque.

Página virtual: 0x0D3F0 / 0x2000 = 0x06 (marco 3). Dirección física de 0x0D3F0: 3 x 0x2000 + (0x0D3F0 - 0x0C000) = 0x6000 + 0x13F0 = 0x73F0. Índice = (0x73F0 >> 7) & 1 = (0x39F) & 1 = 1. Etiqueta = 0x73F0 >> 8 = 0x73. El conjunto 1 tiene (tras los accesos anteriores) los bloques con etiquetas 0x73 y 0x00 (o 0x08 según lo anterior). Si el 0x73 sigue en caché, **acierto** para 0x0D3F0 a 0x0D3FF.

La dirección 0x0D400: página virtual 0x06, dirección física = 0x6000 + 0x1400 = 0x7400. Índice = (0x7400 >> 7) & 1 = 0. Etiqueta = 0x7400 >> 8 = 0x74. El conjunto 0 tiene etiquetas 0x23 y 0x47 (o lo que haya entrado antes). La etiqueta 0x74 no coincide. **Fallo en caché**. LRU expulsa la vía menos reciente del conjunto 0 y entra el bloque con etiqueta 0x74.

*Referencias 27000-2701F:*

Página virtual: 0x27000 / 0x2000 = 0x13. Esta página no está en la tabla de páginas (que solo tiene las páginas 0x01, 0x04, 0x06 y 0x0C). **Fallo de página**. Se debe cargar la página virtual 0x13 en memoria física. LRU expulsa el marco menos recientemente usado. Con escritura directa, las páginas expulsadas no necesitan ser escritas a disco. Después de cargar la página y resolver la traducción, se resuelve también el acceso a la caché (que resultará en fallo de caché porque el bloque no estará cargado).

---

Ejercicio 6

Enunciado: Sea un sistema de memoria con las siguientes características:

- Memoria virtual paginada de 4 GB, política de emplazamiento LRU.
- Tamaño de página 2 KB.
- Memoria principal con 13 bits para la dirección.
- Memoria caché de direcciones físicas de 2048 bytes, con bloques de 256 bytes, asociativa por conjuntos con 4 bloques por conjunto, política LRU.

a) Indicar el formato de la dirección virtual y de la dirección física, desde el punto de vista de la memoria virtual y de la memoria caché.
b) Se desea acceder a las direcciones: `55118FF8`, `0A000000` y `11281276`. Indicar si se produce algún fallo y los valores de las tablas al finalizar el acceso. Indicar en los casos de fallo de página los bloques de caché que se invalidarían. Las tablas previas al acceso son:

| Num Página | Marco | Edad |
|-----------|-------|------|
| 0x000001  | 0     | 0    |
| 0x1B0000  | 1     | 1    |
| 0x0AA231  | 2     | 3    |
| 0x100000  | 3     | 2    |

| Conjunto | Marco | Etiqueta | Edad |
|---------|-------|----------|------|
| 0       | 0     | 0xF      | 0    |
| 0       | 1     | 0x0      | 2    |
| 0       | 2     | 0xB      | 3    |
| 0       | 3     | 0x1      | 1    |
| 1       | 0     | 0xA      | 1    |
| 1       | 1     | 0x2      | 3    |
| 1       | 2     | 0x1      | 2    |
| 1       | 3     | 0xB      | 0    |

c) Si el computador accede a la dirección `FFA34500`, indicar el rango de su página en memoria virtual, el rango de su marco en memoria principal, el rango de su bloque en memoria principal y el rango de su marco de bloque en caché.

Resolución:

**Apartado a: formato de las direcciones**

La memoria virtual tiene 4 GB = 2^32 bytes, por lo que las direcciones virtuales son de **32 bits**. Con páginas de 2 KB = 2^11 bytes, el offset ocupa 11 bits y el número de página virtual los 32 - 11 = **21 bits** restantes.

Formato de la dirección virtual: `[número de página virtual: 21 bits | offset: 11 bits]`

La memoria principal tiene 13 bits de dirección, por lo que tiene 2^13 = 8192 bytes = 8 KB. Con páginas de 2 KB, el número de marcos es 8192 / 2048 = **4 marcos** y el número de marco ocupa 13 - 11 = **2 bits**.

Formato de la dirección física desde el punto de vista de la memoria virtual: `[número de marco: 2 bits | offset: 11 bits]`

Para la caché, tiene 2048 bytes con bloques de 256 bytes y 4 vías. El número de conjuntos es 2048 / (256 x 4) = **2 conjuntos**. El offset ocupa log2(256) = 8 bits, el índice log2(2) = 1 bit y la etiqueta los 13 - 8 - 1 = **4 bits** restantes.

Formato de la dirección física desde el punto de vista de la caché: `[etiqueta: 4 bits | índice: 1 bit | offset: 8 bits]`

**Apartado b: accesos a las tres direcciones**

*Dirección 0x55118FF8:*

Página virtual: 0x55118FF8 >> 11 = 0x55118FF8 / 0x800 = 0xAA231. La tabla de páginas tiene la página 0x0AA231 en el marco 2. **No hay fallo de página**. La dirección física es 2 x 0x800 + (0x55118FF8 & 0x7FF) = 0x1000 + 0x7F8 = 0x17F8.

Para la caché: índice = bit 8 de 0x17F8 = (0x17F8 >> 8) & 1 = 1. Etiqueta = 0x17F8 >> 9 = 0xB. El conjunto 1 tiene las etiquetas 0xA, 0x2, 0x1, 0xB. La etiqueta 0xB coincide con el marco 3 del conjunto 1 (edad 0, el más antiguo). **Acierto en caché**. Se actualiza la edad del bloque accedido (pasa a ser el más reciente).

Tras el acceso, las edades del conjunto 1 se actualizan: el bloque con etiqueta 0xB pasa de edad 0 a la más reciente. Los demás incrementan su "antigüedad" relativa.

*Dirección 0x0A000000:*

Página virtual: 0x0A000000 >> 11 = 0x0A000000 / 0x800 = 0x14000. Esta página no está en la tabla de páginas (que tiene 0x000001, 0x1B0000, 0x0AA231, 0x100000). **Fallo de página**. Se debe cargar la página 0x14000 en memoria. Los 4 marcos están ocupados. LRU expulsa el marco con edad más alta (el más antiguo). La tabla indica edades 0, 1, 3, 2, por lo que el más antiguo es el marco 0 (edad 0, página 0x000001). Se expulsa la página 0x000001 del marco 0 y entra la página 0x14000.

Los bloques de caché que deben invalidarse son los que pertenecían al marco 0 (antiguo contenido), es decir, los bloques con etiqueta correspondiente al marco 0. Las etiquetas de caché para el marco 0 son las que tienen los 2 bits más significativos de la dirección física = 00 (ya que el marco 0 ocupa las direcciones físicas 0x0000 a 0x07FF). Los bits de etiqueta son los 4 bits más altos de la dirección física de 13 bits: para el marco 0 serían las etiquetas 0x0 y 0x1 (ya que 0x0000 a 0x07FF cubre etiquetas 0x0 y parte de 0x1 en la indexación de la caché). Concretamente, las entradas de caché con etiqueta 0x0 (conjunto 0, marco 1) y 0x1 (conjunto 0, marco 3 y conjunto 1, marco 2) que correspondan al marco 0 se deben invalidar.

La dirección física de la nueva página 0x14000 es: nuevo marco 0 (se asigna al marco liberado). Dirección física = 0 x 0x800 + (0x0A000000 & 0x7FF) = 0x0000. Para la caché: índice = bit 8 de 0x0000 = 0. Etiqueta = 0x0000 >> 9 = 0x0. El conjunto 0 tiene etiqueta 0x0 en el marco 1 (si no fue invalidada). Si hay coincidencia, **acierto de caché** (poco probable si se ha invalidado). Si se invalida el bloque 0x0 del conjunto 0, entonces hay **fallo de caché** y se carga el nuevo bloque.

La tabla de páginas se actualiza: sale la página 0x000001 (marco 0) y entra la página 0x14000 en el marco 0. Las edades se actualizan: la página recién cargada tiene la edad más reciente.

*Dirección 0x11281276:*

Página virtual: 0x11281276 >> 11 = 0x11281276 / 0x800 = 0x22502. Esta página no está en la tabla de páginas (que ahora tiene 0x14000, 0x1B0000, 0x0AA231, 0x100000). **Fallo de página**. LRU expulsa el marco más antiguo. Tras los accesos anteriores, el orden de edades ha cambiado. El marco más antiguo es ahora uno de los que no han sido accedidos en los pasos 1 y 2. Sin perder generalidad, suponemos que el marco 1 (página 0x1B0000, que no ha sido accedida) es el más antiguo y se expulsa. Entra la página 0x22502 en el marco 1.

Los bloques de caché correspondientes al marco 1 (direcciones físicas 0x0800 a 0x0FFF, etiquetas que empiezan por 0x1) se invalidan. Concretamente, las entradas con etiqueta 0x1 en el conjunto 0 (marco 3) y en el conjunto 1 (marco 2) que pertenezcan al marco 1. Se invalidan esas entradas.

La dirección física de 0x11281276 en el nuevo marco: 1 x 0x800 + (0x11281276 & 0x7FF) = 0x800 + 0x276 = 0xA76. Para la caché: índice = bit 8 de 0xA76 = (0xA76 >> 8) & 1 = 0. Etiqueta = 0xA76 >> 9 = 0x5. El conjunto 0 no tiene la etiqueta 0x5 (tiene 0xF, 0x0, 0xB, 0x1). **Fallo de caché**. LRU expulsa la entrada más antigua del conjunto 0 (la de mayor edad, que es el bloque con etiqueta 0xB, edad 3) y entra el nuevo bloque con etiqueta 0x5.

**Apartado c: dirección FFA34500**

Dirección virtual: 0xFFA34500.

Número de página virtual: 0xFFA34500 >> 11 = 0xFFA34500 / 0x800 = 0x1FF468 (redondeando: 0xFFA34500 / 0x800 = 0x1FF468A, tomando la parte entera = 0x1FF468). Offset dentro de la página = 0xFFA34500 & 0x7FF = 0x500.

Rango de la página en memoria virtual: la página comienza en 0x1FF468 x 0x800 = 0xFFA34000 y termina en 0xFFA34000 + 0x7FF = **0xFFA34000 a 0xFFA347FF**.

Suponiendo que esta página está mapeada a algún marco (con las tablas del estado final del apartado b), el marco asignado le da la dirección física. Si la página 0x1FF468 no está en la tabla, habría fallo de página. Suponiendo que está mapeada al marco m, la dirección física es m x 0x800 + 0x500.

Rango del marco en memoria principal: si el marco es m, el rango es m x 0x800 a m x 0x800 + 0x7FF.

Rango del bloque en memoria principal: el bloque de caché contiene 256 bytes. La dirección física se redondea al múltiplo de 256 más cercano hacia abajo. Si la dirección física es 0xMAP + 0x500, el bloque comienza en (0xMAP + 0x500) & ~0xFF y tiene longitud 256 bytes.

Rango del marco de bloque en caché (conjunto): el índice de caché determina el conjunto. Si el índice es 0, el conjunto 0 alberga el bloque; si es 1, el conjunto 1. El rango del conjunto es el conjunto completo (4 vías x 256 bytes = 1024 bytes de capacidad en el conjunto, aunque no se corresponde directamente con un rango contiguo de direcciones físicas).

---

Ejercicio 7

Enunciado: Sea un computador con memoria virtual paginada, cada página tiene 1024 palabras, en memoria virtual tiene 8 páginas y la memoria física 4 marcos de página.

a) Indicad el tamaño en bits de la dirección de memoria virtual y de la dirección de memoria física.
b) Si la jerarquía tiene una memoria caché de 1024 palabras de acceso directo con 4 bloques de caché de direcciones virtuales, indicad qué tamaño en bits tiene la etiqueta de caché.
c) ¿Qué valores de etiqueta de caché indicarían que algún bloque de la página virtual cero está en caché?

Tabla de páginas:

| Página Virtual | Hit | Marco de Página |
|---------------|-----|----------------|
| 0             | 1   | 3              |
| 1             | 1   | 1              |
| 2             | 0   | 0              |
| 3             | 0   | 0              |
| 4             | 1   | 2              |
| 5             | 0   | 0              |
| 6             | 1   | 0              |
| 7             | 0   | 0              |

Resolución:

**Apartado a: tamaño de las direcciones**

La memoria virtual tiene 8 páginas de 1024 palabras cada una. La dirección virtual necesita identificar la página (3 bits para 8 páginas) y la palabra dentro de la página (10 bits para 1024 palabras). Total: **13 bits de dirección virtual**.

La memoria física tiene 4 marcos de página de 1024 palabras cada uno. La dirección física necesita identificar el marco (2 bits para 4 marcos) y la palabra dentro del marco (10 bits). Total: **12 bits de dirección física**.

**Apartado b: tamaño de la etiqueta de caché virtual**

La caché tiene 1024 palabras con 4 bloques de emplazamiento directo. Cada bloque contiene 1024 / 4 = 256 palabras. El índice de bloque ocupa log2(4) = 2 bits y el offset de bloque log2(256) = 8 bits. La etiqueta ocupa los bits restantes de la dirección virtual: 13 - 2 - 8 = **3 bits**.

**Apartado c: valores de etiqueta para la página virtual cero**

La dirección virtual de la página 0 va de 0x0000 a 0x03FF (las primeras 1024 palabras del espacio virtual). Con un formato de 13 bits dividido en [etiqueta 3b | índice 2b | offset 8b], los bits de etiqueta son los 3 más significativos de la dirección virtual.

Para las direcciones de la página 0 (0x0000 a 0x03FF), los 3 bits más significativos de una dirección de 13 bits siempre son **000**, ya que 0x0000 a 0x03FF corresponde al rango 0 0000 0000 0000 a 0 0011 1111 1111 en binario de 13 bits (bits 12:10 = 000 en todos los casos).

Por tanto, el valor de etiqueta que indica que un bloque de la página virtual 0 está en caché es **0x0** (etiqueta igual a cero). Si cualquiera de los 4 bloques de la caché tiene etiqueta 0, significa que contiene datos de la página virtual 0.

---

### Problemas de Rendimiento

---

Ejercicio R1

Enunciado: Sea un sistema con las siguientes características:

Procesador:

- CPI ideal de 1.
- 35% de las instrucciones son de acceso a memoria.

Caché L1:

- 64 KB, unificada, emplazamiento directo, postescritura, bit sucio y asignación en escritura.
- Líneas de 8 bytes.
- 25% de las líneas modificadas.
- Tasa de fallos de 0,021.
- Direcciones físicas.

Memoria principal:

- Latencia de 60 ciclos.
- Tasa de transferencia de bloques: 4 bytes por ciclo.

TLB:

- Tasa de fallos: 0,03.
- Penalización: 7 ciclos.

a) Calcular el CPI real del sistema.
b) Calcular el nuevo CPI real añadiendo una caché L2 con las siguientes características:

- 1 MB, unificada, asociativa por conjuntos con E = 2, escritura directa sin asignación en escritura.
- Latencia de acceso a L2: 20 ciclos.
- Líneas de 64 bytes.
- Tasa de transferencia con MP: 16 bytes por ciclo.
- Tasa de transferencia con L1: 4 bytes por ciclo.
- Tasa de fallos local: 0,2.
- Del 80% de las instrucciones que llegan a L2 son de lectura y el 20% de escritura.
- Las cachés usan direccionamiento virtual con los mismos valores de TLB.
c) Indicar cuál de las dos organizaciones es mejor.

Resolución:

**Apartado a: CPI real sin L2**

El CPI real se calcula como:

CPI = CPI_ideal + penalizaciones_por_fallos_TLB + penalizaciones_por_fallos_caché

**Penalización por fallos de TLB:**

Cada instrucción de acceso a memoria (35% del total) consulta la TLB. Una fracción de 0,03 de esas consultas falla. La penalización es 7 ciclos.

Penalización TLB = 0,35 x 0,03 x 7 = **0,0735 ciclos por instrucción**.

**Penalización por fallos de caché L1:**

El tiempo de penalización por fallo en L1 es el tiempo de traer un bloque de 8 bytes desde MP. Con latencia de 60 ciclos y tasa de transferencia de 4 bytes por ciclo, el tiempo para traer 8 bytes es 60 + 8/4 = 60 + 2 = **62 ciclos**.

En postescritura con bit sucio, cuando se expulsa una línea modificada hay que escribirla en MP. El 25% de las líneas son modificadas, por lo que el coste adicional de escritura por fallo es 0,25 x (tiempo de escritura en MP). El tiempo de escritura de 8 bytes es también 8/4 = 2 ciclos (más la latencia, total 62 ciclos). Este coste se añade al fallo de lectura cuando se expulsa una línea sucia.

Penalización por fallo = 62 + 0,25 x 62 = 62 x 1,25 = **77,5 ciclos**.

El 35% de las instrucciones acceden a memoria, y la tasa de fallos es 0,021.

Penalización caché = 0,35 x 0,021 x 77,5 = **0,5696 ciclos por instrucción**.

**CPI real:**

CPI = 1 + 0,0735 + 0,5696 = **1,643 ciclos por instrucción**.

**Apartado b: CPI real con L2**

Con L2, los fallos de L1 se resuelven primero en L2 (si están allí) antes de ir a MP.

**Penalización por fallos de TLB:** igual que antes, 0,0735 ciclos (se mantiene el mismo valor).

**Penalización por fallos en L1 resueltos en L2:**

El tiempo de transferencia de un bloque de L1 (8 bytes) desde L2 con tasa de 4 bytes/ciclo es 8/4 = 2 ciclos, más la latencia de acceso a L2 de 20 ciclos. Total: **22 ciclos** por fallo de L1 que acierta en L2.

**Penalización por fallos en L1 que también fallan en L2 (van a MP):**

El bloque de L2 es de 64 bytes. El tiempo de traer 64 bytes desde MP con tasa de 16 bytes/ciclo es 64/16 = 4 ciclos, más la latencia de 60 ciclos. Total: **64 ciclos** para traer el bloque de MP a L2. Luego se transfiere el bloque (o la parte relevante) a L1.

La tasa de fallos local de L2 es 0,2, lo que significa que el 20% de los accesos que llegan a L2 fallan en L2. Los que aciertan en L2 (80%) pagan 22 ciclos; los que fallan en L2 (20%) pagan 22 + 64 = 86 ciclos.

Penalización media por fallo de L1 = 0,8 x 22 + 0,2 x 86 = 17,6 + 17,2 = **34,8 ciclos**.

Para las escrituras en L2 (escritura directa sin asignación): el 20% de los accesos que llegan a L2 son de escritura. Con escritura directa, las escrituras que fallan en L2 van directamente a MP sin cargar el bloque en L2. Para esas escrituras la penalización es la latencia de escritura en MP.

Penalización escrituras L2 = 0,35 x 0,021 x 0,20 x 60 = **0,0882 ciclos** (penalización adicional por escrituras que van a MP directamente).

Penalización total por caché = 0,35 x 0,021 x 34,8 = **0,256 ciclos por instrucción**.

**CPI real con L2:**

CPI = 1 + 0,0735 + 0,256 = **1,330 ciclos por instrucción**.

**Apartado c: comparativa**

La organización con L2 es mejor: el CPI baja de 1,643 a 1,330, una mejora de aproximadamente el 19%. La L2 absorbe la mayoría de los fallos de L1 a un coste mucho menor que ir a MP, reduciendo significativamente la penalización media por fallo.

---

Ejercicio R2

Enunciado: Sea una memoria principal de 1 MB dividida en bloques de 2 palabras, siendo cada palabra de 1 byte. El sistema tiene una memoria caché asociativa por conjuntos de 2^8 bytes con un grado de asociatividad de E = 4 y un tiempo de acceso de 10 ns.

a) Número de bits del bus de direcciones.
b) Número de bloques por conjunto de la memoria caché.
c) Formato de las direcciones de memoria caché.
d) Calcular el tiempo medio de acceso a memoria y el ancho de banda sabiendo lo siguiente:

- Tasa de fallos de la caché: 3%.
- El bus y la memoria son de 1 palabra y el acceso es secuencial.
- 10 ns en enviar la dirección, 80 ns en acceder al dato, 10 ns en enviar el dato.
e) Para implementar memoria virtual con procesador de 64 bits, ¿cuál sería el formato de la dirección virtual para una caché virtualmente accedida físicamente marcada?

Resolución:

**Apartado a: bits del bus de direcciones**

La memoria principal tiene 1 MB = 2^20 bytes. El bus de direcciones necesita **20 bits**.

**Apartado b: bloques por conjunto**

La caché tiene 2^8 = 256 bytes con bloques de 2 palabras = 2 bytes. El número total de bloques en la caché es 256 / 2 = **128 bloques**. Con un grado de asociatividad E = 4, el número de conjuntos es 128 / 4 = **32 conjuntos**. Cada conjunto tiene **4 bloques**.

**Apartado c: formato de las direcciones**

El offset de bloque ocupa log2(2) = **1 bit** (tamaño de bloque = 2 bytes). El índice de conjunto ocupa log2(32) = **5 bits**. La etiqueta ocupa los 20 - 1 - 5 = **14 bits** restantes.

Formato: `[etiqueta: 14 bits | índice: 5 bits | offset: 1 bit]`

**Apartado d: tiempo medio de acceso y ancho de banda**

El tiempo de acceso en acierto es simplemente el tiempo de acceso a la caché: **10 ns**.

En caso de fallo, hay que traer el bloque desde MP. El acceso secuencial con bus de 1 palabra (1 byte) y bloque de 2 palabras significa que se transfieren 2 bytes uno a uno. Para cada byte: 10 ns (enviar dirección) + 80 ns (acceder al dato en MP) + 10 ns (enviar dato) = 100 ns. Para el bloque completo de 2 bytes de forma secuencial: 2 x 100 ns = **200 ns de penalización**.

Tiempo medio de acceso = T_acierto + tasa_fallos x penalización = 10 + 0,03 x 200 = 10 + 6 = **16 ns**.

Ancho de banda: en el caso de un acierto se transfiere 1 byte en 10 ns, lo que da 1/10 = 0,1 bytes/ns = **100 MB/s**. Con la tasa de fallos, el tiempo efectivo medio por acceso es 16 ns, lo que da un ancho de banda efectivo de 1/16 = 0,0625 bytes/ns = **62,5 MB/s**.

**Apartado e: formato de la dirección virtual para caché VI/FM**

Con una caché virtualmente accedida físicamente marcada y el mismo formato de caché (14 bits de etiqueta, 5 bits de índice, 1 bit de offset), la dirección virtual mantiene los mismos campos de índice y offset (que deben coincidir con los bits de la dirección física para que el sistema funcione correctamente).

El procesador genera direcciones de 64 bits. Los bits de índice y offset (6 bits en total: bits 5:0) pertenecen al offset de página y no cambian en la traducción. Los bits de etiqueta en la dirección virtual son los 64 - 6 = **58 bits superiores**, aunque en la práctica la etiqueta física que se usa para la comparación en la caché es la que proviene de la traducción.

Formato de la dirección virtual: `[número de página virtual: 58 bits | offset de página: 6 bits]`, donde el offset de página cubre los 5 bits de índice de caché más el 1 bit de offset de bloque.

---

Ejercicio R3

Enunciado: Sea una memoria principal de 2^64 bytes y una memoria caché de 2^6 bytes. La memoria principal tiene tiempos de acceso de 150 ns, la memoria caché de 8 ns y la MP se divide en 2^62 bloques.

a) Formato de la dirección caché para emplazamiento directo.
b) Formato para emplazamiento totalmente asociativo.
c) Formato para emplazamiento asociativo por conjuntos de 4 vías.
d) Tiempo medio de acceso y ancho de banda con tasa de fallos del 8%, bus y memoria de 1 byte, acceso secuencial: 25 ns enviar dirección, 100 ns acceder al dato, 25 ns enviar el dato.
e) Tiempo medio de acceso y ancho de banda con tasa de fallos del 8%, bus y memoria de 4 bytes, acceso en paralelo.
f) Tiempo medio de acceso y ancho de banda con tasa de fallos del 8%, memoria en 4 módulos de 1 palabra entrelazados, acceso en paralelo.
g) Formato de la dirección virtual con páginas de 32K palabras.
h) Número de posiciones de memoria para la tabla de páginas directa con proceso de tamaño máximo.

Resolución:

**Datos iniciales**

La MP tiene 2^64 bytes y la caché 2^6 = 64 bytes. La MP se divide en 2^62 bloques, por lo que el tamaño de cada bloque es 2^64 / 2^62 = **4 bytes** (2 bits de offset de bloque). Las direcciones de MP son de **64 bits**.

**Apartado a: emplazamiento directo**

El offset ocupa 2 bits (bloque de 4 bytes). La caché tiene 64 / 4 = 16 bloques, por lo que el índice ocupa log2(16) = 4 bits. La etiqueta ocupa los 64 - 2 - 4 = **58 bits** restantes.

Formato: `[etiqueta: 58 bits | índice: 4 bits | offset: 2 bits]`

**Apartado b: totalmente asociativo**

Con acceso totalmente asociativo no hay campo de índice. Todo lo que no es offset es etiqueta.

Formato: `[etiqueta: 62 bits | offset: 2 bits]`

**Apartado c: asociativo por conjuntos de 4 vías**

Con 16 bloques totales y 4 vías, hay 16 / 4 = 4 conjuntos. El índice ocupa log2(4) = 2 bits y la etiqueta 64 - 2 - 2 = **60 bits**.

Formato: `[etiqueta: 60 bits | índice: 2 bits | offset: 2 bits]`

**Apartado d: acceso secuencial, bus y memoria de 1 byte**

El bloque tiene 4 bytes. Con bus de 1 byte y acceso secuencial, se transfieren 4 bytes uno a uno. Cada byte cuesta 25 + 100 + 25 = 150 ns. Para el bloque completo: 4 x 150 = **600 ns de penalización**.

Tiempo medio = 8 + 0,08 x 600 = 8 + 48 = **56 ns**.

Ancho de banda efectivo: en cada acceso se transfiere 1 byte en 56 ns (tiempo medio). Ancho de banda = 1 / 56 ns = 0,0179 bytes/ns = **17,9 MB/s**.

**Apartado e: bus y memoria de 4 bytes, acceso en paralelo**

Con bus de 4 bytes y bloque de 4 bytes, el bloque entero se transfiere en un solo ciclo: 25 (dirección) + 100 (acceso a dato) + 25 (envío) = **150 ns de penalización**.

Tiempo medio = 8 + 0,08 x 150 = 8 + 12 = **20 ns**.

Ancho de banda: se transfieren 4 bytes en 20 ns. Ancho de banda = 4 / 20 ns = 0,2 bytes/ns = **200 MB/s**.

**Apartado f: 4 módulos entrelazados, acceso en paralelo**

Con 4 módulos entrelazados de 1 palabra cada uno, los 4 bytes del bloque se acceden en paralelo en los 4 módulos simultáneamente. El tiempo de acceso al bloque completo es el mismo que para 1 palabra: 25 (dirección) + 100 (acceso en paralelo a los 4 módulos) + 25 (envío de los 4 bytes en paralelo) = **150 ns de penalización**.

El resultado es el mismo que el apartado e en términos de tiempo de penalización, pero el entrelazado permite además solapes en accesos consecutivos.

Tiempo medio = 8 + 0,08 x 150 = **20 ns**. Ancho de banda = **200 MB/s**.

**Apartado g: formato de la dirección virtual**

Si el tamaño de la dirección virtual es el mismo que el de la dirección real (64 bits) y una página tiene 32K = 2^15 palabras (palabras de 1 byte, por lo que la página tiene 2^15 bytes = 32 KB), el offset de página ocupa 15 bits y el número de página virtual los 64 - 15 = **49 bits** restantes.

Formato: `[número de página virtual: 49 bits | offset de página: 15 bits]`

**Apartado h: posiciones para la tabla de páginas directa**

Con un proceso de tamaño máximo (que usa todo el espacio de direcciones virtuales de 64 bits), el número máximo de páginas virtuales es 2^49. La tabla de páginas directa tiene una entrada por página virtual, por lo que necesita **2^49 posiciones de memoria**.

---

Ejercicio R4

Enunciado: Sea la siguiente jerarquía de memoria:

- Memoria caché: tiempos de acceso de 20 ns, bloques de 8 bytes y tasa de fallos del 1%.
- Memoria principal: tiempos de acceso total para 4 bytes de 200 ns y tasa de fallos del 0,001%.
- El ancho de bus entre caché y MP es de 4 bytes.
- Memoria secundaria: tiempo promedio de acceso a una posición de 1 millón de ns y tiempo de acceso a un byte de 10 ns.
- Sistema de memoria virtual paginada con páginas de 512 bytes.

a) Tiempo de acceso medio a memoria caché.
b) Tiempo medio de acceso a la memoria virtual.
c) Tiempo medio de acceso de la jerarquía.

Resolución:

**Apartado a: tiempo de acceso medio a memoria caché**

En caso de acierto, el tiempo de acceso es directamente el tiempo de la caché: **20 ns**.

En caso de fallo, hay que traer el bloque desde MP. El bloque tiene 8 bytes y el bus transfiere 4 bytes a la vez, por lo que se necesitan 2 transferencias. El tiempo de acceso a MP para 4 bytes es 200 ns, y para el bloque completo de 8 bytes son 2 x 200 = 400 ns de penalización.

Tiempo medio de acceso a caché = T_acierto + tasa_fallos x penalización = 20 + 0,01 x 400 = 20 + 4 = **24 ns**.

**Apartado b: tiempo medio de acceso a la memoria virtual**

La memoria virtual añade la posibilidad de fallo de página: si la página no está en MP, hay que traerla de la memoria secundaria.

El tiempo de acceso total al dato cuando hay fallo de página incluye el tiempo de localizar la página en disco (1 millón de ns para posicionar la cabeza) más el tiempo de leer la página entera. La página tiene 512 bytes, y con 10 ns por byte el tiempo de lectura es 512 x 10 = 5120 ns. El tiempo total de un fallo de página es 1 000 000 + 5120 ≈ **1 005 120 ns**.

La tasa de fallos de MP (0,001%) es la probabilidad de que al acceder a MP la página no esté (fallo de página). Esta tasa es 0,00001.

Tiempo medio de acceso a memoria virtual = T_MP + tasa_fallos_MP x T_fallo_pagina = 24 + 0,00001 x 1 005 120 = 24 + 10,05 ≈ **34,05 ns**.

Nota: aquí T_MP es el tiempo medio de acceso a la caché calculado en el apartado a, ya que la jerarquía incluye la caché como primer nivel.

**Apartado c: tiempo medio de acceso de la jerarquía**

El tiempo medio de acceso de la jerarquía completa combina los tres niveles: caché, MP y disco. Siguiendo la fórmula de tiempo medio de acceso en jerarquías:

T_jerarquía = T_caché + tasa_fallos_caché x (T_MP + tasa_fallos_MP x T_disco)

donde T_disco es el tiempo de acceso al dato en disco cuando hay fallo de página.

T_caché = 20 ns (tiempo de acceso si acierta en caché).
Penalización por fallo en caché (traer bloque de 8 bytes desde MP) = 2 x 200 = 400 ns.
Penalización por fallo de página (traer página desde disco y luego bloque desde MP) = 1 005 120 + 400 ns.

T_jerarquía = 20 + 0,01 x (400 + 0,00001 x (1 005 120 + 400))

T_jerarquía = 20 + 0,01 x (400 + 0,00001 x 1 005 520)

T_jerarquía = 20 + 0,01 x (400 + 10,06)

T_jerarquía = 20 + 0,01 x 410,06

T_jerarquía = 20 + 4,10 = **24,10 ns**.

El resultado es muy próximo al tiempo medio de acceso a la caché porque la tasa de fallos de página es extremadamente baja (0,001%), lo que hace que la penalización del disco tenga un impacto muy reducido en el tiempo medio total.
