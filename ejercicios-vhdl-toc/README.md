# Ejercicios TOC

## Mario González García

---

## Lección 1 – Modelado y diseño de hardware con VHDL

### 1. Codifica en VHDL

En todo el ejercicio he omitido el importe de la biblioteca:

```vhdl
library ieee;
use ieee.std_logic_1164.all;
```

#### XOR con dos entradas de 4 bits

```vhdl
entity xor_gate is
  port (
    e1: in std_logic_vector(3 downto 0);
    e2: in std_logic_vector(3 downto 0);
    s: out std_logic_vector(3 downto 0)
  );
end xor_gate;

architecture rtl of xor_gate is
begin 
  s <= e1 xor e2;
end architecture rtl;
```

#### Multiplexor 4:1

He decidido que las entradas sean de **4 bits** como en el XOR

```vhdl
entity mul is
  port (
    e0: in std_logic_vector(3 downto 0);
    e1: in std_logic_vector(3 downto 0);
    e2: in std_logic_vector(3 downto 0);
    e3: in std_logic_vector(3 downto 0);
    select: in std_logic_vector(1 downto 0);
    s: out std_logic_vector(3 downto 0)
  );
end mul;

architecture rtl of mul is
begin
  sel_entry: process (select)
  begin
    case select is
      when "00" => s <= e0;
      when "01" => s <= e1;
      when "10" => s <= e2;
      when "11" => s <= e3;
      when others => s <= "0000";
    end case;
  end process sel_entry;
end architecture rtl;
```

#### Decodificador 3:8

He decidido que las entradas sean de **4 bits** como en el XOR

```vhdl
entity dec is
  port (
    e: in std_logic_vector(2 downto 0);
    o: out std_logic_vector(7 downto 0)
  );
end dec;

architecture rtl of dec is
begin
  decodify: process (e)
  begin
    case e is
      when "000" => o <= "00000001";
      when "001" => o <= "00000010";
      when "010" => o <= "00000100";
      when "011" => o <= "00001000";
      when "100" => o <= "00010000";
      when "101" => o <= "00100000";
      when "110" => o <= "01000000";
      when "111" => o <= "10000000";
      when others => o <= "00000000";
    end case;
  end process decodify;
end architecture rtl;
```

#### Codificador 8:3

He decidido que las entradas sean de **4 bits** como en el XOR

```vhdl
entity cod is
  port (
    e: in std_logic_vector(7 downto 0);
    o: out std_logic_vector(2 downto 0)
  );
end cod;

architecture rtl of cod is
begin
  codify: process (e)
  begin
    if e(7) = '1' then o <= "111";
    elsif e(6) = '1' then o <= "110";
    elsif e(5) = '1' then o <= "101";
    elsif e(4) = '1' then o <= "100";
    elsif e(3) = '1' then o <= "011";
    elsif e(2) = '1' then o <= "010";
    elsif e(1) = '1' then o <= "001";
    elsif e(0) = '1' then o <= "000";
    else o <= "000";
    end if;
  end process codify;
end architecture rtl;
```

#### Comparador de 2 num (4 bits)

He decidido que las entradas sean de **4 bits** como en el XOR

```vhdl
entity comp is
  port (
    e1: in std_logic_vector(3 downto 0);
    e2: in std_logic_vector(3 downto 0);
    s: out std_logic
  );
end comp;

architecture rtl of comp is
begin
  comparate: process (e1, e2)
  begin
    if e1 = e2 then
      s <= '1';
    else
      s <= '0';
    end if;
  end process comparate;
end architecture rtl;
```

### 2. Codifica en VHDL un sistema combinatorio

* La entrada es un número entre 0 y 15
* La salida (Z) es 1 si:
  * La entrada es un número primo
  * La entrada es inferior a 4 y par
  * La entrada es mayor a 8 e impar

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity comb is 
  port (
    e: std_logic_vector(3 downto 0);
    Z: std_logic
  );
end comb;

architecture rtl of comb is
begin
  calc: process (e)
  begin
    if (to_integer(unsigned(e)) > 8) and (e(0) = '1') 
      Z <= '1';
    elsif (e = "0000") or (e = "0010") or (e = "0011") or (e = "0101") or (e="0111")
      Z <= '1';
    else
      Z <= '0';
    end if;
  end process calc;
end architecute rtl;
```

### 3. Codifica en VHDL un sistema combinatorio

* **Objetivo**: Multiplicador por 3
* La entrada es un entero positivo entre 0 y 7
* La salida (Z) es un entero positivo entre  0 y 15
* Existe una flag (D) para detectar overflow

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity comb is
  port (
    e: in std_logic_vector(2 downto 0);
    Z: out std_logic_vector(3 downto 0);
    D: out std_logic
  );
end comb;

architecture rtl of comb is
begin
  triplicate: process (e)
    variable temp : unsigned(4 downto 0);
  begin
    temp := 3 * unsigned(e);
    Z <= std_logic_vector(temp(3 downto 0));
    D <= temp(4);
  end process triplicate;
end architecture rtl;
```

### 4. Codifica en VHDL un sistema combinatorio

* **Objetivo**: Implementa este boceto

![Diagrama del ejercicio 4](./public/ej4.png)

He decidido que las entradas sean de **4 bits** como en el **ejercicio 1**.

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity xor_gate is
  port (
    e1: in std_logic_vector(3 downto 0);
    e2: in std_logic_vector(3 downto 0);
    s: out std_logic_vector(3 downto 0)
  );
end xor_gate;

architecture rtl of xor_gate is
begin 
  s <= e1 xor e2;
end architecture rtl;

entity comb is
  port (
    A0: in std_logic_vector(3 downto 0);
    A1: in std_logic_vector(3 downto 0);
    A2: in std_logic_vector(3 downto 0);
    A3: in std_logic_vector(3 downto 0);
    extra: out std_logic_vector(3 downto 0)
  );
end comb;

architecture rtl of comb is
  component xor_gate
    port (
      e1: in std_logic_vector(3 downto 0);
      e2: in std_logic_vector(3 downto 0);
      s: out std_logic_vector(3 downto 0)
    );
  end component;

  signal temp1, temp2: std_logic_vector(3 downto 0);

begin
  xor1: xor_gate port map (
    e1 => A0,
    e2 => A1,
    s => temp1
  );

  xor2: xor_gate port map (
    e1 => A2,
    e2 => A3,
    s => temp2
  );

  xor3: xor_gate port map (
    e1 => temp1,
    e2 => temp2,
    s => extra
  );

end architecture rtl;
```

### 5. Codifica en VHDL un sistema secuencial y completa el cronograma

* **Objetivo**: Implementa el sistema para que la salida (z) cumpla:

$$
z(t) =
\begin{cases}
1 & \text{si } x(t-2, t-1, t) = 'aaa' \text{ o } 'bbb' \\
0 & \text{en otro caso}
\end{cases}
$$

![Maquina Mealy de estados del ejercicio 5](./public/ej5_1.png)

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity seq_detector is
  port (
    clk : in std_logic;
    reset : in std_logic;
    x : in std_logic;
    z : out std_logic
  );
end seq_detector;

architecture rtl of seq_detector is
  type state_type is (EI, S0, S1, S2, S3);
  signal state, next_state : state_type;

begin
  change_state : process (clk, reset)
  begin
    if reset = '1' then
      state <= EI;
    elsif rising_edge(clk) then
      state <= next_state;
    end if;
  end process change_state;

  calc_state_out : process (state, x)
  begin
    next_state <= state;
    z <= '0';

    case state is
      when EI =>
        if x = '0' then
          next_state <= S0;
        elsif x = '1' then
          next_state <= S1;
        end if;
      
      when S0 =>
        if x = '0' then
          next_state <= S2;
        elsif x = '1' then
          next_state <= S1;
        end if;

      when S1 =>
        if x = '0' then
          next_state <= S0;
        elsif x = '1' then
          next_state <= S3;
        end if;

      when S2 =>
        if x = '0' then
          next_state <= S2;
          z <= '1';
        elsif x = '1' then
          next_state <= S1;
        end if;

      when S3 =>
        if x = '0' then
          next_state <= S0;
        elsif x = '1' then
          next_state <= S3;
          z <= '1';
        end if;

      when others =>
        next_state <= EI;
    end case;
  end process calc_state_out;
end architecture rtl;
```

* **Objetivo**: Completar el cronograma:

![Cronograma del ejercicio 5](./public/ej5_2.png)

### 6. Sobre una FSM Mealy

![FSM Mealy del ejercicio 6](./public/ej6_1.png)

* **Objetivo**: Reconoce el patrón detectado

La única combinación que resulta en '1' es aquella cuya en el **S3** recibe una **entrada de valor A**.

Para llegar ahí se necesita:

S0 - B -> S1 - A -> S2 - B -> S3 - A -> S2

Es decir: BABA. Sin embargo para volver a obtener un 1 necesitaríamos:

S2 - B -> S3 - A -> S2

Por tanto, el patrón es "BABA"

* **Objetivo**: Implementar en VHDL

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity seq_detector is
  port (
    clk: in std_logic;
    rst: in std_logic;
    x: in std_logic;
    z: out std_logic
  );
end seq_detector;

architecture rtl of seq_detector is
  type state_type is (S0, S1, S2, S3);
  signal state, next_state : state_type;
  
begin
  change_state : process (clk, rst)
  begin
    if rst = '1' then
      state <= S0;
    elsif rising_edge(clk) then
      state <= next_state;
    end if;
  end process change_state;

  calc_state_out : process (state, x)
  begin
    next_state <= state;
    z <= '0';

    case state is
      when S0 =>
        if x = '0' then
          next_state <= S0;
        elsif x = '1' then
          next_state <= S1;
        end if;

      when S1 =>
        if x = '0' then
          next_state <= S2;
        elsif x = '1' then
          next_state <= S1;
        end if;

      when S2 =>
        if x = '0' then
          next_state <= S0;
        elsif x = '1' then
          next_state <= S3;
        end if;
      
      when S3 =>
        if x = '0' then
          next_state <= S2;
          z <= '1':
        elsif x = '1' then
          next_state <= S1;
        end if;

      when others =>
        next_state <= S0;
    end case;
  end process calc_state_out;
end architecture rtl;
```

* **Objetivo**: Completar el cronograma:

![Cronograma del ejercicio 6](./public/ej6_2.png)

### 7. Codifica un desplazador en VHDL

* **Restricción**: No utilizar la instrucción de VHDL SLL

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity shifter is 
  port (
    clk : in  std_logic;
    rst : in  std_logic;
    ctrl : in  std_logic_vector(2 downto 0);
    x : in  std_logic_vector(7 downto 0);
    z : out std_logic_vector(7 downto 0)
  );
end shifter;

architecture rtl of shifter is
  signal temp : std_logic_vector(7 downto 0);
begin
  shift : process(clk, rst)
  begin
    if rst = '1' then
      temp <= (others => '0');
    elsif rising_edge(clk) then
      case ctrl is
        when "000" => temp <= x;
        when "001" => temp <= x(6 downto 0) & '0';
        when "010" => temp <= x(5 downto 0) & "00";
        when "011" => temp <= x(4 downto 0) & "000";
        when "100" => temp <= x(3 downto 0) & "0000";
        when "101" => temp <= x(2 downto 0) & "00000";
        when "110" => temp <= x(1 downto 0) & "000000";
        when others => temp <= x(0) & "0000000";
      end case;
    end if;
  end process shift;
  z <= temp;
end architecture rtl;
```

### 8. Dibuja una descripción RTL equivalente del circuito en VHDL

![Estructura del ejercicio 8](./public/ej8_1.png)

![Diagrama RTL del ejercicio 8](./public/ej8_2.png)

### 9. Codifica una FSM sintetizable en VHDL

* **Objetivo**: Adaptar el siguiente código para crear una FSM equivalente y sintetizable
* He decidido implementar este sistema como una máquina de Moore (la salida "solo" depende del estado)

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity even_detector is
  port (
    clk: in std_logic;
    rst: in std_logic;
    x: in std_logic;
    start: in std_logic;
    even: out std_logic;
  );
end even_detector;

architecture rtl of even_detector is
  type state_type is (EVEN, ODD);
  signal state, next_state: state_type;
begin
  change_state : process (clk, rst)
  begin
    if rst = '1' then
      state <= EVEN;
    elsif rising_edge(clk) then
      state <= next_state;
    end if;
  end process change_state;

  calc_state_out : process (state, x, start)
  begin
    next_state <= state;

    if start = '1' then
      case state is
        when EVEN =>
          if x = '1' then
            next_state <= ODD;
          else
            next_state <= EVEN;
          end if;

        when ODD =>
          if x = '1' then
            next_state <= EVEN;
          else
            next_state <= ODD;
          end if;
      end case;
    end if;

    if state = EVEN then
      even <= '1';
    else
      even <= '0';
    end if;
  end process calc_state_out;
end architecture rtl;
```

### 10. Codifica un sistema secuencial en VHDL

* **Objetivo**: Implementar en VHDL este diagrama de una máquina Mealy

![FSM Mealy del ejercicio 10](./public/ej10.png)

```vhdl
entity seq_detector is
  port (
    clk: in std_logic;
    rst: in std_logic;
    x: in std_logic;
    z: out std_logic
  );
end seq_detector;

architecture rtl of seq_detector is
  type state_type is (S0, S1, S2, S3);
  signal state, next_state : state_type;
begin
  change_state: process(clk, rst)
  begin
    if rst = '1' then
      state <= S0;
    elsif rising_edge(clk) then
      state <= next_state;
    end if;
  end process change_state;

  calc_state_out: process(state, x)
  begin
    next_state <= state;
    z <= '0';

    case state is
      when S0 =>
        if x = '1' then
          next_state <= S0;
        else
          next_state <= S1;
        end if;
      
      when S1 =>
        if x = '1' then
          next_state <= S0;
        else
          next_state <= S2;
        end if;

      when S2 =>
        if x = '1' then
          next_state <= S3;
        else
          next_state <= S2;
        end if;

      when S3 =>
        if x = '1' then
          next_state <= S0;
          z <= '1';
        else
          next_state <= S1;
        end if;

      when others =>
        next_state <= S0;
    end case;
  end process calc_state_out;
end architecture rtl;
```

### 11. Diseña una lavadora en VHDL

* **Objetivo**: Codificar en VHDL una lavadora con las siguientes especificaciones:

1. clk: señal de reloj
2. rst: señal de reset a bajo nivel
3. dos entradas: inicio/parada (1bit) y ciclo rápido (1bit)
4. entrada: start/stop para arrancar y detener la máquina
5. ciclo rápido indica modo de lavado: rápido o lento
6. cinco salidas: entrada de agua, calentar agua, mover paletas, secar y abrir detergente
7. en estado inicial todas las salidas están a 0
8. desde cualquier estado se vuelve al inicial con inicio/parada a 0 hasta que pase a 1
9. el lavado tiene tres etapas: lavado ((2 rápido / 4 lento) ciclos), enjuague ((1 rápido / 2 lento) ciclo(s)) y secado (1 ciclo)
10. tras el secado se regresa al inicio
11. en lavado: entrada de agua, calentar agua en el primer ciclo
12. en lavado: abrir detergente en el segundo ciclo
13. en todos las etapas: mover paletas
14. en secado: secar

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity dishwasher_ctrl is
  port (
    clk, rst_n : in std_logic;
    start_stop : in std_logic;
    fast_cycle : in std_logic;
    fill_water : out std_logic;
    heat_water : out std_logic;
    move_blades : out std_logic;
    open_detergent : out std_logic;
    dry : out std_logic
  );
end dishwasher_ctrl;

architecture rtl of dishwasher_ctrl is
  type state_type is (S0, S1, S2, S3, S4, S5, S6, S7);
  signal state, next_state : state_type;
begin
  change_state: process(clk, rst_n)
  begin
    if rst_n = '0' then
      state <= S0;
    elsif rising_edge(clk) then
      state <= next_state;
    end if;
  end process change_state;

  process(state, start_stop, fast_cycle)
  begin
    next_state <= state;
    case state is
      when S0 =>
        if start_stop = '1' then
          next_state <= S1;
        end if;

      when S1 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S2;
        end if;

      when S2 =>
        if start_stop = '0' then
          next_state <= S0;
        elsif fast_cycle = '1' then
          next_state <= S5;
        else
          next_state <= S3;
        end if;

      when S3 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S4;
        end if;

      when S4 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S5;
        end if;

      when S5 =>
        if start_stop = '0' then
          next_state <= S0;
        elsif fast_cycle = '1' then
          next_state <= S7;
        else
          next_state <= S6;
        end if;

      when S6 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S7;
        end if;

      when S7 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S0;
        end if;

      when others =>
        next_state <= S0;
    end case;
  end process;

  process(state)
  begin
    fill_water <= '0';
    heat_water <= '0';
    move_blades <= '0';
    open_detergent <= '0';
    dry <= '0';

    case state is
      when S1 =>
        fill_water <= '1';
        heat_water <= '1';
        move_blades <= '1';
      when S2 =>
        move_blades <= '1';
        open_detergent <= '1';
      when S3 | S4 | S5 | S6 =>
        move_blades <= '1';
      when S7 =>
        dry <= '1';
      when others =>
        null;
    end case;
  end process;

end architecture rtl;
```

### 12. Diseña un sistema de lavado en VHDL

* **Objetivo**: Codificar en VHDL un sistema de lavado con las siguientes especificaciones:

1. clk: señal de reloj
2. rst: señal de reset a bajo nivel
3. dos entradas: inicio/parada (1bit) y encerado (1bit)
4. entrada: arranque/parada
5. la entrada de encerado permite la opción de encerar
6. cinco salidas: riego, aire, rodillo, jabón y encerado
7. en estado inicial todas las salidas están a 0
8. desde cualquier estado se vuelve al inicial con arranque/parada a 0 hasta que pase a 1
9. el lavado tiene cuatro etapas: enjabonado (1 ciclo), rodillos (2 ciclos), enjuague (1 ciclo), secado (1 ciclo)
10. si hay encerado, se pasará por 2 ciclos antes del final, si no, regresa al estado inicial

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity carwash_ctrl is
  port (
    clk, rst_n : in std_logic;
    start_stop : in std_logic;
    wax_option : in std_logic;
    soap : out std_logic;
    rollers : out std_logic;
    water : out std_logic;
    air : out std_logic;
    wax : out std_logic
  );
end carwash_ctrl;

architecture rtl of carwash_ctrl is
  type state_type is (S0, S1, S2, S3, S4, S5, S6, S7);
  signal state, next_state : state_type;
begin
  change_state: process(clk, rst_n)
  begin
    if rst_n = '0' then
      state <= S0;
    elsif rising_edge(clk) then
      state <= next_state;
    end if;
  end process change_state;

  calc_state: process(state, start_stop, wax_option)
  begin
    next_state <= state;

    case state is
      when S0 =>
        if start_stop = '1' then
          next_state <= S1;
        end if;

      when S1 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S2;
        end if;

      when S2 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S3;
        end if;

      when S3 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S4;
        end if;

      when S4 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S5;
        end if;

      when S5 =>
        if start_stop = '0' then
          next_state <= S0;
        elsif wax_option = '1' then
          next_state <= S6;
        else
          next_state <= S0;
        end if;

      when S6 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S7;
        end if;

      when S7 =>
        if start_stop = '0' then
          next_state <= S0;
        else
          next_state <= S0;
        end if;

      when others =>
        next_state <= S0;
    end case;
  end process calc_state_out;

  calc_out: process(state)
  begin
    soap <= '0';
    rollers <= '0';
    water <= '0';
    air <= '0';
    wax <= '0';

    case state is
      when S1 =>
        soap <= '1';
      when S2 | S3 =>
        rollers <= '1';
      when S4 =>
        water <= '1';
      when S5 =>
        air <= '1';
      when S6 | S7 =>
        wax <= '1';
      when others =>
        null;
    end case;
  end process calc_out;

end architecture rtl;
```
