# Coprocessador-Grafico-Programavel
Desenvolvimento de um coprocessador gráfico programável em Verilog para FPGA DE1-SoC, com ISA de 32 bits, interface ARM-FPGA via MMIO, fila de instruções e integração dos motores gráficos de background, sprites e rasterização de polígonos.


### 1. ISA e interface de software

A arquitetura utiliza uma *Instruction Set Architecture* (ISA) de 32 bits para controlar as operações gráficas do coprocessador. Cada instrução possui um código de operação (*opcode*) nos bits `[31:27]`, enquanto os 27 bits restantes contêm os parâmetros específicos de cada comando. Foram definidas nove instruções, responsáveis pela configuração e movimentação de elementos gráficos, manipulação da paleta e sincronização com a VGA.

| Instrução | Opcode | Finalidade |
|---|---|---|
| `LOAD_REG` | `00000` | Armazenar um valor em um registrador do coprocessador. |
| `ESPERA_VSYNC` | `00001` | Aguardar a condição de sincronismo vertical implementada na Unidade de Controle. |
| `PALE_DEFINIR` | `00010` | Definir uma cor em uma posição da paleta. |
| `SPR_CRIAR` | `00011` | Configurar um sprite. |
| `SPR_MOVER` | `00100` | Movimentar um sprite em uma direção. |
| `BG_TILE` | `00101` | Selecionar um *tilemap* de plano de fundo. |
| `BG_MOVER` | `00110` | Movimentar o plano de fundo. |
| `POLI_DEFINIR` | `00111` | Configurar um polígono. |
| `POLI_MOVER` | `01000` | Movimentar um polígono em uma direção. |

A definição da ISA estabelece o formato binário dos comandos e permite que a Unidade de Controle identifique a operação solicitada a partir do *opcode*, interpretando os demais campos de acordo com o tipo de instrução.

#### 1.1. Arquivo `gpu_isa.h`

O arquivo `gpu_isa.h` declara as funções disponíveis para os programas em C. Seus protótipos definem os nomes, os tipos e a ordem dos argumentos esperados, permitindo que a aplicação utilize as instruções sem implementar diretamente a codificação binária.

Um exemplo é a declaração:

```c
void asm_load_reg(
    volatile uint32_t *mmio,
    uint32_t reg_dest,
    uint32_t valor
);
```

O ponteiro `mmio` identifica a região de endereços utilizada para acessar o coprocessador endereçado; `reg_dest` indica o registrador de destino do coprocessador; e `valor` corresponde ao dado a ser armazenado. O tipo `uint32_t` representa um inteiro sem sinal de 32 bits.

Esse arquivo .h não contém a implementação das instruções, pois é de responsabilidade do arquivo `gpu_driver.S`, definido em Assembly ARM que disponibiliza as funções declaradas para uso da aplicação.

#### 1.2. Programa em C

O programa em C define a sequência de comandos que compõe cada cena gráfica e fornece os parâmetros necessários às operações. As chamadas das funções declaradas em `gpu_isa.h` permitem configurar registradores, cores, sprites, plano de fundo e polígonos, além de solicitar a sincronização com a VGA.

Por exemplo:

```c
asm_load_reg(mmio, REG_XY, empacotar_xy(x, y));
asm_spr_criar(mmio, REG_XY, indice, textura, 1, 0, 0);
```

Nesse exemplo, a primeira chamada armazena as coordenadas empacotadas no registrador `REG_XY`. A segunda solicita a configuração de um sprite, utilizando esse registrador como fonte das coordenadas e fornecendo o índice, a textura, a visibilidade e as opções de espelhamento.

Dessa forma, o C define a operação desejada e seus parâmetros, enquanto o driver em Assembly ARM realiza a codificação e o envio das instruções ao hardware.

### 2. Caminho da instrução até a FIFO

O envio das instruções utiliza o mecanismo de entrada e saída mapeadas em memória (*Memory-Mapped Input/Output* — MMIO). Essa abordagem permite que o processador ARM acesse registradores do periférico por meio de operações de leitura e escrita em endereços de memória.

No programa em C, a região é acessada por meio de `open()` e `mmap()`. A constante `MMIO_BASE`, definida como `0xFF200000`, representa o endereço físico utilizado pelo programa, que deve corresponder ao mapeamento configurado para o periférico no *Platform Designer*. O resultado de `mmap()` é convertido em um ponteiro `volatile uint32_t *`, utilizado nas chamadas do driver para acessar a região mapeada.

#### 2.1. Codificação em Assembly ARM

O arquivo `gpu_driver.S` implementa as funções declaradas em `gpu_isa.h`. Cada função recebe os argumentos da operação, utiliza instruções ARM para posicionar os campos definidos pela ISA e monta a palavra binária de 32 bits correspondente ao comando.

Operações como `AND`, `LSL` e `ORR` são utilizadas para coletar, deslocar e combinar os campos da instrução. Ao final da montagem, a palavra completa fica disponível no registrador `R1` para ser enviada ao periférico. O registrador `R0` mantém o ponteiro `mmio`.

O envio da instrução binária final é feita no label `_send_fifo`, compartilhada pelas funções do driver:

```asm
_send_fifo:
1:
    LDR R2, [R0, #4]
    AND R2, R2, #1
    CMP R2, #1
    NOP
    BEQ 1b

    STR R1, [R0, #0]
    BX LR
```

Essa função consulta inicialmente o registrador de estado, acessado pelo deslocamento de quatro bytes em relação à base. A operação `AND` isola o bit menos significativo, que representa o indicador de fila cheia. Se o resultado for `1`, a instrução `BEQ 1b` retorna ao início do laço, repetindo a consulta. Quando a fila não está cheia, `STR R1, [R0, #0]` escreve a palavra de 32 bits no registrador destinado ao recebimento das instruções, e `BX LR` retorna à função que realizou a chamada.

Esse procedimento implementa uma espera ocupada (*busy-waiting*), ou seja, o ARM aguarda a disponibilidade de espaço antes de realizar a escrita, evitando enviar novos comandos quando a fila está cheia.

#### 2.2. Fluxo da instrução

O fluxo completo, desde a chamada da função em C até o armazenamento da instrução na FIFO, é apresentado na Figura 1.

![Fluxo da instrução do programa em C até a FIFO](docs/figuras/fluxo_instrucao_fifo.png)

*Figura 1 — Caminho da instrução desde a aplicação em C até a fila do coprocessador.*

O fluxograma representa a participação do programa em C, do driver em Assembly ARM, da interface MMIO e da fila de instruções. Após o envio, a instrução permanece armazenada até ser solicitada pela Unidade de Busca, que a encaminha para o registrador de instrução e, posteriormente, para a Unidade de Controle.

### 3. Interface MMIO e fila de instruções

A comunicação entre o processador ARM e a fila exigiu uma interface dedicada porque as operações realizadas pelo barramento Avalon-MM não correspondem diretamente aos sinais necessários para controlar a FIFO. O barramento transporta endereço, dados e sinais de leitura ou escrita; a fila, por sua vez, necessita de um sinal de escrita, de uma palavra de 32 bits e de um sinal de leitura. A `interface_mmio` realiza essa adaptação, decodificando os acessos do ARM e convertendo-os em sinais utilizados pela fila.

#### 3.1. Módulo `interface_mmio`

A interface recebe os sinais do barramento, identifica o registrador acessado e encaminha as informações de acordo com a operação solicitada. No projeto, foram definidos dois endereços locais para a comunicação:

| Endereço local | Deslocamento utilizado | Operação | Finalidade |
|---|---|---|---|
| `00` | `0x00` | Escrita | Receber a instrução de 32 bits. |
| `01` | `0x04` | Leitura | Retornar o indicador de fila cheia. |

Essa correspondência deve ser compatível com a configuração de endereçamento da interface no *Platform Designer*.

Na operação de escrita, o módulo verifica se `avs_escrever` está ativo, se estava inativo no ciclo anterior e se `avs_endereco` corresponde a `00`. Quando as condições são satisfeitas, a instrução presente em `avs_instrucao` é armazenada em `coe_fila_instrucao`, e o sinal `coe_fila_escrever` é ativado por um ciclo de clock. O registrador `avs_escrever_anterior` permite identificar a nova borda de subida do sinal de escrita.

Na operação de leitura, quando `avs_ler` está ativo e o endereço local corresponde a `01`, a interface retorna o estado de `coe_fila_cheia` no bit menos significativo de `avs_SinalFila`, mantendo os outros bits em zero. Essa informação é utilizada pelo driver em Assembly ARM para determinar quando pode enviar uma nova instrução.

Portanto, a interface não realiza a interpretação do comando gráfico nem armazena uma sequência de instruções. Sua função é transformar as operações de acesso ao periférico do coprocessador em sinais adequados à comunicação com a FIFO e disponibilizar o estado da fila ao ARM.

#### 3.2. Módulo `fila_fifo`

A FIFO (*First-In, First-Out*) armazena temporariamente as instruções recebidas do ARM, mantendo a ordem de chegada dos comandos até que sejam consumidos pela Unidade de Busca. Sua capacidade padrão é de 16 posições, cada uma com largura de 32 bits.

O armazenamento é implementado por um vetor `fila`, enquanto os registradores `ponteiro_escrever` e `ponteiro_ler` controlam as posições de entrada e saída. Ambos possuem quatro bits, permitindo endereçar as posições de 0 a 15. Ao avançarem além da última posição, retornam naturalmente a zero, formando uma estrutura circular. O registrador `quantidade`, de cinco bits, contabiliza os elementos armazenados e permite identificar os estados de fila cheia e vazia.

A escrita ocorre quando `fila_escrever` está ativo e `fila_cheia` está inativo. A instrução fornecida em `arm_instrucao` é armazenada na posição indicada por `ponteiro_escrever`, que avança após a operação. A leitura ocorre quando `unidade_ler` está ativo e `fila_vazia` está inativo. Nesse caso, o ponteiro de leitura avança para a próxima posição.

A saída `instrucao_saida` é combinacional e acompanha diretamente a posição indicada por `ponteiro_ler`. Dessa forma, a instrução localizada no começo da fila já está disponível na saída, sem depender de uma operação de leitura para que seu valor seja apresentado.

Os sinais `fila_cheia` e `fila_vazia` são derivados de `quantidade`: a fila está cheia quando contém 16 instruções e vazia quando não contém nenhuma. Em condições normais, se uma leitura e uma escrita forem solicitadas simultaneamente, com a fila nem cheia nem vazia, os dois ponteiros avançam e a quantidade permanece constante, pois uma instrução entra enquanto outra é lida.

A escrita é controlada pela interface MMIO, enquanto a leitura é solicitada pela Unidade de Busca por meio de `valid_leitura_fifo`, conectado ao sinal `unidade_ler` da fila. Essa separação permite que o ARM envie comandos sem controlar diretamente o instante em que o coprocessador os processa.

#### 3.3. Integração da comunicação

A integração entre o driver, a interface MMIO e a FIFO estabelece o caminho de transferência das instruções entre o processador ARM e o coprocessador gráfico. O driver verifica o estado da fila antes de escrever, a interface converte a operação de barramento em um pulso de escrita e a FIFO armazena a palavra recebida. No sentido da leitura, a Unidade de Busca solicita a instrução disponível e a FIFO avança seu ponteiro, disponibilizando o próximo comando.

Essa organização separa as responsabilidades do software, da interface de comunicação e do armazenamento temporário. Assim, o programa em C não precisa conhecer os ponteiros ou a estrutura interna da fila, enquanto a Unidade de Busca obtém as instruções na ordem em que foram enviadas. A instrução disponibilizada é armazenada no registrador de instrução e entregue à Unidade de Controle para decodificação e execução da operação correspondente.
