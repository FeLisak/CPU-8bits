# CPU de Arquitetura Simplificada de 8 Bits

O objetivo do projeto foi construir uma CPU de arquitetura simplificada de 8 bits, implementando uma ALU com as operações aritméticas e lógicas clássicas, um circuito de controle capaz de gerenciar os ciclos de busca e execução, e um conjunto de memórias para armazenar instruções e variáveis.

O projeto é dividido em dois arquivos: `CPU_com_barramento.dig`, que contém o circuito principal com o PC, memórias, controle e barramento, e `ALU.dig`, que é a Unidade Lógica e Aritmética instanciada como subcircuito dentro da CPU.

---

## Visão Geral da Arquitetura

A CPU segue um modelo von Neumann simplificado, com barramento único compartilhado entre todos os módulos. As principais características são:

| Característica                           | Detalhe                        |
| ---------------------------------------- | ------------------------------ |
| Largura do barramento de dados           | 8 bits                         |
| Largura do PC                            | 8 bits                         |
| Espaço endereçável de dados              | 256 posições                   |
| Espaço de programa (ROM)                 | 8 posições                     |
| Registrador acumulador (AC)              | 8 bits                         |
| Registrador quociente/multiplicador (MQ) | 8 bits                         |
| Opcode                                   | 3 bits (8 operações possíveis) |

---

## Componentes Principais

### Program Counter (PC)

O PC é um contador de 8 bits que mantém o endereço da próxima instrução. Ele recebe o clock global e o sinal de `reset`. O reset é disparado automaticamente por um comparador quando o PC atinge o valor 5, fazendo o programa rodar em loop contínuo entre os endereços 0 e 4. O valor atual do PC é distribuído pelo sinal `PC_out` para as memórias.

---

### Memória de Instruções (ROM)

A ROM armazena os opcodes do programa, com 3 bits de endereço e 3 bits de dados. O endereço é fornecido pelos 3 bits menos significativos do `PC_out`, extraídos via Splitter. O conteúdo configurado é `7*0, 5` — as 7 primeiras posições com valor `0` e a última com valor `5`, que corresponde à instrução de divisão e armazenamento. A saída da ROM alimenta tanto a ALU (como sinal de seleção de operação) quanto o circuito de controle.

---

### Memória de Dados (EEPROM Dual-Port)

A memória de dados é uma EEPROM de porta dupla com 8 bits de endereço e 8 bits de dados, iniciada com os valores `2, 4, 6, 8, 10`. A porta de leitura é endereçada pelo `PC_out` e coloca o operando no barramento. A porta de escrita é controlada pelo sinal `Storage`, que habilita a gravação do resultado da ALU de volta à memória. As duas operações são sincronizadas pelo clock global.

---

### Circuito de Controle

O circuito de controle determina quando cada módulo pode acessar o barramento e quando a EEPROM deve ser escrita. Um comparador de 3 bits verifica se o OpCode atual é igual a `5`. Quando for, uma porta AND combina esse sinal com o clock para gerar um pulso de escrita sincronizado. Esse pulso é distribuído pelo sinal `Storage`, que habilita o Driver de escrita da EEPROM e desabilita, via inversores NOT, os Drivers de leitura — evitando conflito no barramento.

---

## Barramento da CPU

O barramento é um fio compartilhado de 8 bits que conecta todos os módulos. Para que múltiplos componentes possam estar ligados ao mesmo fio sem conflito, o acesso é controlado por **Drivers tristate** — portões que colocam um módulo em alta impedância quando ele não está em uso.

O circuito conta com três Drivers de 8 bits:

| Driver   | Função                                                                  |
| -------- | ----------------------------------------------------------------------- |
| Driver 1 | Leva os dados da EEPROM até o barramento                                |
| Driver 2 | Leva os dados do barramento até a entrada B da ALU                      |
| Driver 3 | Leva o resultado do acumulador (AC) de volta ao barramento para escrita |

O controle de quem está ativo é feito pelo sinal `Storage`. Quando `Storage = 0`, os Drivers 1 e 2 estão habilitados e os dados fluem da EEPROM para a ALU. Quando `Storage = 1`, eles são desabilitados pelos inversores NOT e o Driver 3 assume, colocando o resultado da ALU no barramento para ser gravado na EEPROM.

Os sinais internos são distribuídos por **tunnels** — fios nomeados que conectam pontos distantes do circuito sem precisar traçar uma linha física. Os principais são `Clock`, `reset`, `PC_out`, `OpCode` e `Storage`.

---

## ALU

### Visão Geral

A ALU recebe dois operandos de 8 bits (A e B), um sinal de seleção de 3 bits e o clock. Ela produz dois resultados: o **AC** (acumulador), que guarda o resultado principal, e o **MQ** (quociente/multiplicador), que guarda resultados secundários como o resto de divisões ou a parte baixa de multiplicações.

### Operações Suportadas

| Select | Operação      | Descrição                                                          |
| ------ | ------------- | ------------------------------------------------------------------ |
| `000`  | Soma          | A + B                                                              |
| `001`  | Subtração     | A − B                                                              |
| `010`  | Multiplicação | A × B — resultado de 16 bits, parte alta em AC e parte baixa em MQ |
| `011`  | NAND          | NOT(A AND B), bit a bit                                            |
| `100`  | XOR           | A XOR B, bit a bit                                                 |
| `101`  | Divisão       | A ÷ B — quociente em AC, resto em MQ                               |
| `110`  | Shift Left    | Deslocamento lógico à esquerda                                     |
| `111`  | Shift Right   | Deslocamento lógico à direita                                      |

Cada operação é um subcircuito independente. Dois multiplexadores de 8 entradas selecionam qual resultado vai para os registradores AC e MQ, conforme o sinal `Select`.

### Registradores AC e MQ

Os resultados dos multiplexadores são capturados em registradores de 8 bits no flanco do clock. O AC armazena o resultado principal e o MQ armazena o resultado secundário. A saída do MQ é realimentada internamente para operações que precisam preservar o estado anterior.

---

## Correção da ALU — Unificação do MQ

Na versão anterior da ALU, existiam dois pinos de saída distintos para o MQ: um vindo da multiplicação (`MQ0`, parte baixa do produto de 16 bits) e outro vindo da divisão (`MQ1`, resto da divisão). Isso gerava dois pinos `MQ` separados na interface do subcircuito, o que causava conflito quando a ALU era conectada à CPU — o circuito externo não tinha como saber qual dos dois consumir.

A correção foi adicionar um segundo multiplexador exclusivo para o MQ, controlado pelo mesmo sinal `Select`. Com isso:

- Quando `Select = 010` (multiplicação), o MUX seleciona `MQ0`
- Quando `Select = 101` (divisão), o MUX seleciona `MQ1`
- Para as demais operações, o MUX realimenta o próprio registrador MQ, preservando o valor anterior

Agora a ALU expõe um único pino `MQ`, eliminando a ambiguidade e tornando a integração com a CPU direta e sem conflitos.

---

## Mapa de Memórias

### ROM

| Endereço | OpCode | Operação                |
| -------- | ------ | ----------------------- |
| 0–6      | `000`  | Soma                    |
| 7        | `101`  | Divisão / Armazenamento |

### EEPROM

| Endereço | Valor |
| -------- | ----- |
| 0        | 2     |
| 1        | 4     |
| 2        | 6     |
| 3        | 8     |
| 4        | 10    |
| 5–255    | 0     |

## Vídeo de Demonstração

Clique [aqui](https://youtu.be/HmslYmr_9FY) para acessar o vídeo de demonstração e explicação do funcionamento da CPU de 8 bits.
