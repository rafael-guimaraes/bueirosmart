# Circuito BueiroSmart

O circuito do BueiroSmart foi desenvolvido para simular, em lógica digital, um sistema de monitoramento preventivo de bueiros urbanos. Ele recebe três grupos de entradas digitais, processa essas informações por meio de comparadores, portas lógicas, flip-flop e blocos aritméticos, e gera saídas visuais, sonoras e numéricas.

O funcionamento geral pode ser dividido em duas partes principais:

1. **Lógica de decisão do estado do bueiro**
2. **Cálculo do Índice Local de Risco — IR**

---

## 1. Entradas do circuito

O circuito possui três entradas principais:

```text
N[4:0] = nível de água
P[4:0] = chuva local
T[3:0] = inclinação/deslocamento da tampa
```

No simulador, essas entradas são representadas por interruptores digitais. Elas simulam as palavras binárias que seriam obtidas a partir dos sensores do sistema.

### Nível de água — N[4:0]

A entrada `N[4:0]` representa o nível de água dentro do bueiro.

```text
N = 00000 até 11111
N = 0 até 31
```

Quanto maior o valor de `N`, maior o nível da água.

### Chuva local — P[4:0]

A entrada `P[4:0]` representa a intensidade da chuva local.

```text
P = 00000 até 11111
P = 0 até 31
```

Quanto maior o valor de `P`, maior a intensidade de chuva detectada.

### Tampa — T[3:0]

A entrada `T[3:0]` representa a inclinação ou deslocamento da tampa.

```text
T = 0000 até 1111
T = 0 até 15
```

Quanto maior o valor de `T`, maior a inclinação ou alteração física da tampa.

---

## 2. Comparadores de limiar

Após as entradas, o circuito utiliza comparadores para transformar os valores binários em sinais lógicos intermediários.

Esses sinais indicam se determinada condição foi atingida:

```text
NA = nível de água em atenção
NC = nível de água crítico
CF = chuva forte
CB = chuva baixa
TD = tampa deslocada
TC = tampa crítica
```

### NA — nível de atenção

```text
NA = N >= 16
```

Esse sinal indica que o nível da água entrou em uma faixa de atenção.

Como `N` possui 5 bits, qualquer valor de 16 a 31 tem o bit mais significativo `N4 = 1`. Por isso, essa condição pode ser obtida diretamente por:

```text
NA = N4
```

### NC — nível crítico

```text
NC = N >= 22
```

Esse sinal indica que o nível da água está alto o suficiente para participar de situações críticas.

### CF — chuva forte

```text
CF = P >= 12
```

Esse sinal indica chuva forte.

### CB — chuva baixa

```text
CB = P <= 5
```

Esse sinal indica chuva baixa ou ausente.

### TD — tampa deslocada

```text
TD = T >= 3
```

Esse sinal indica deslocamento moderado da tampa.

### TC — tampa crítica

```text
TC = T >= 8
```

Esse sinal indica inclinação severa ou possível remoção da tampa.

Como `T` possui 4 bits, qualquer valor de 8 a 15 tem o bit mais significativo `T3 = 1`. Por isso:

```text
TC = T3
```

---

## 3. Lógica de decisão

Depois dos comparadores, o circuito usa portas lógicas para decidir o estado do bueiro.

### Suspeita de obstrução

```text
OBSTRUCAO = NC AND CB
```

Essa condição ocorre quando o nível da água está crítico, mas a chuva está baixa ou ausente.

Interpretação:

```text
água alta + pouca chuva = possível obstrução
```

### Risco por chuva

```text
RISCO_CHUVA = NC AND CF
```

Essa condição ocorre quando o nível da água está crítico e há chuva forte.

Interpretação:

```text
água alta + chuva forte = risco por evento de chuva
```

### Violação crítica da tampa

```text
VIOLACAO_CRITICA = TC
```

Essa condição ocorre quando a tampa está em inclinação severa ou possível remoção.

Esse evento é crítico independentemente do nível de água ou da chuva.

---

## 4. Sinal crítico instantâneo — CRITICO_RAW

Os três eventos críticos são reunidos em uma porta OR:

```text
CRITICO_RAW = OBSTRUCAO OR RISCO_CHUVA OR VIOLACAO_CRITICA
```

Ou seja:

```text
CRITICO_RAW = (NC AND CB) OR (NC AND CF) OR TC
```

Esse sinal representa uma condição crítica detectada naquele momento.

Ele é chamado de `CRITICO_RAW` porque ainda não está memorizado. Se a condição crítica desaparecer, esse sinal pode voltar para 0.

---

## 5. Latch crítico com Flip-Flop D

Para evitar que o alarme crítico oscile ou desapareça rapidamente, o circuito utiliza um Flip-Flop tipo D configurado como latch.

A ideia é separar dois sinais:

```text
CRITICO_RAW = crítico instantâneo
CRITICO = crítico memorizado
```

O sinal `CRITICO` é a saída `Q` do Flip-Flop.

A entrada D do Flip-Flop é definida como:

```text
D = CRITICO_RAW OR CRITICO
```

Isso cria uma realimentação.

### Funcionamento

Se uma condição crítica acontecer:

```text
CRITICO_RAW = 1
```

então:

```text
D = 1 OR CRITICO
D = 1
```

No clock, o Flip-Flop salva esse valor e a saída vira:

```text
CRITICO = 1
```

Mesmo que depois `CRITICO_RAW` volte para 0, a realimentação mantém:

```text
D = 0 OR 1
D = 1
```

Assim, o estado crítico permanece ativo até o reset.

O reset limpa o Flip-Flop e retorna:

```text
CRITICO = 0
```

---

## 6. Saídas de estado

O circuito possui três estados principais:

```text
NORMAL
ATENCAO
CRITICO
```

A prioridade é:

```text
1. Crítico
2. Atenção
3. Normal
```

### Estado crítico

```text
LED_VERMELHO = CRITICO
BUZZER = CRITICO
```

Quando `CRITICO = 1`, o LED vermelho acende e o buzzer é acionado.

### Estado de atenção

Primeiro, o circuito gera uma atenção intermediária:

```text
ATENCAO_RAW = NA OR TD
```

Ou seja, existe atenção quando há nível de água em faixa intermediária ou deslocamento moderado da tampa.

Mas a atenção só deve aparecer se não houver crítico ativo:

```text
ATENCAO = ATENCAO_RAW AND NOT(CRITICO)
```

A saída correspondente é:

```text
LED_BRANCO = ATENCAO
```

### Estado normal

O estado normal só fica ativo quando não há crítico nem atenção:

```text
NORMAL = NOT(CRITICO) AND NOT(ATENCAO)
```

A saída correspondente é:

```text
LED_VERDE = NORMAL
```

---

# Cálculo do Índice Local de Risco — IR

Além das saídas de estado, o circuito calcula um valor numérico chamado `IR`.

O IR resume a condição geral do bueiro em uma palavra de 5 bits.

A fórmula usada é:

```text
IR = (N + P + 4T) >> 2
```

Essa expressão combina:

```text
N = nível de água
P = chuva local
T = inclinação da tampa
```

---

## 7. Normalização da entrada T

A entrada `T` possui 4 bits e varia de 0 a 15.

Já `N` e `P` possuem 5 bits e variam de 0 a 31.

Para deixar `T` em escala equivalente, considera-se:

```text
Tn = 2T
```

Como a fórmula usa `2Tn`, temos:

```text
2Tn = 2(2T)
2Tn = 4T
```

Por isso o circuito implementa:

```text
IR = (N + P + 4T) >> 2
```

---

## 8. Bloco SHIFT_T_X4

O bloco `SHIFT_T_X4` realiza:

```text
T << 2
```

Deslocar `T` duas posições para a esquerda equivale a multiplicar por 4.

Exemplo:

```text
T = 15
T = 1111

T << 2 = 111100
T << 2 = 60
```

---

## 9. Bloco SOMADOR_NP

O bloco `SOMADOR_NP` soma as entradas `N` e `P`:

```text
SOMADOR_NP = N + P
```

Exemplo com valores máximos:

```text
N = 31
P = 31

N + P = 62
```

---

## 10. Bloco SOMADOR_IR

O bloco `SOMADOR_IR` soma o resultado de `N + P` com `4T`:

```text
SOMADOR_IR = (N + P) + (T << 2)
```

Ou seja:

```text
SOMADOR_IR = N + P + 4T
```

Exemplo com valores máximos:

```text
N = 31
P = 31
T = 15

N + P = 62
4T = 60

SOMADOR_IR = 122
```

---

## 11. Bloco DIV4_IR

O bloco `DIV4_IR` realiza o deslocamento à direita por duas posições:

```text
DIV4_IR = SOMADOR_IR >> 2
```

Esse deslocamento equivale à divisão inteira por 4.

Exemplo:

```text
IR = 122 >> 2
IR = 30
```

Em binário:

```text
30 = 11110
```

Então:

```text
IR[4:0] = 11110
```

---

## 12. Codificador COD_IR_HEX

O bloco `COD_IR_HEX` prepara o valor de `IR` para ser mostrado em dois displays de 7 segmentos no formato hexadecimal.

Como o IR possui 5 bits:

```text
IR[4:0]
```

o codificador separa esse valor em dois dígitos hexadecimais:

```text
HI = 000IR4
LO = IR3 IR2 IR1 IR0
```

### Exemplo com IR máximo

```text
IR = 30 decimal
IR = 11110 binário
```

Separando:

```text
HI = 0001 = 1
LO = 1110 = E
```

Portanto, os displays mostram:

```text
1E
```

Esse valor representa `30` em decimal, mas exibido em hexadecimal.

---

## 13. Decodificador hexadecimal para 7 segmentos

Depois do `COD_IR_HEX`, cada dígito hexadecimal passa por um decodificador para display de 7 segmentos.

O decodificador recebe 4 bits:

```text
0000 até 1111
```

e aciona os segmentos necessários para mostrar:

```text
0, 1, 2, 3, 4, 5, 6, 7, 8, 9, A, B, C, D, E, F
```

Exemplo:

```text
HI = 0001 -> display mostra 1
LO = 1110 -> display mostra E
```

Resultado visual:

```text
1E
```

---

## 14. Displays de 7 segmentos

Os displays mostram o valor do IR em hexadecimal.

```text
Display esquerdo = dígito mais significativo
Display direito = dígito menos significativo
```

Exemplo:

```text
IR = 30 decimal
IR = 1E hexadecimal
```

Então:

```text
Display esquerdo = 1
Display direito = E
```

---

# Resumo geral do circuito

O circuito possui dois caminhos principais.

## Caminho da decisão lógica

```text
Entradas N, P, T
        ↓
Comparadores de limiar
        ↓
NA, NC, CF, CB, TD, TC
        ↓
Lógica combinatória
        ↓
CRITICO_RAW, ATENCAO_RAW
        ↓
Latch crítico
        ↓
CRITICO, ATENCAO, NORMAL
        ↓
LEDs e buzzer
```

## Caminho do cálculo do IR

```text
Entradas N, P, T
        ↓
N + P
        ↓
T << 2
        ↓
N + P + 4T
        ↓
>> 2
        ↓
IR[4:0]
        ↓
Codificador hexadecimal
        ↓
Decodificadores de 7 segmentos
        ↓
Displays
```

---

# Conclusão

O circuito do BueiroSmart combina lógica combinatória, memória por latch e operação aritmética. Os comparadores identificam condições importantes dos sensores, as portas lógicas classificam o estado do bueiro, o Flip-Flop memoriza eventos críticos e o bloco aritmético calcula o Índice Local de Risco.

Com isso, o sistema consegue indicar três estados principais:

```text
Normal
Atenção
Crítico
```

e também exibir um valor numérico de risco em formato hexadecimal nos displays de 7 segmentos.

