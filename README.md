[README.md](https://github.com/user-attachments/files/33548071/README.md)
# PBL03 — Processador RISC-V com jogo Genius e verificação UVM

Projeto acadêmico de implementação e verificação funcional de um **processador RISC-V de 32 bits**, descrito em **SystemVerilog** e organizado por meio do **Block Design do AMD Vivado**. O processador executa um programa do jogo **Genius** (memória de sequência de cores), utilizando entradas de chaves e botões direcionais e saídas para quatro LEDs.

A validação é realizada por simulação com **Vivado XSim** e um ambiente de testes baseado em **UVM (Universal Verification Methodology)**, que aplica estímulos ao processador, monitora as respostas e verifica automaticamente o comportamento do jogo.

## Objetivos

- Integrar os blocos principais de um processador RISC-V com caminho de dados de ciclo único.
- Executar um programa armazenado em memória de instruções, com acesso a RAM e periféricos por **MMIO** (*Memory-Mapped I/O*).
- Simular o funcionamento do jogo Genius, incluindo níveis de dificuldade e modos de operação.
- Verificar sequências de LEDs, entradas do jogador, condições de erro, reinicialização e temporização usando UVM.

## Tecnologias

- **Linguagem:** SystemVerilog
- **Ambiente:** AMD Vivado 2026.1
- **Simulador:** XSim
- **Metodologia de verificação:** UVM 1.2
- **Arquitetura:** RISC-V de 32 bits, com suporte a um subconjunto de instruções RV32I
- **Dispositivo configurado no projeto Vivado:** `xc7z010clg400-1`

> O foco do material disponibilizado é a implementação RTL e a **simulação funcional**. Não há, neste repositório, comprovação de validação em placa física.

## Arquitetura do processador

O processador é construído a partir de módulos independentes conectados no diagrama `RISCVPROJECT.bd`:

| Módulo | Função |
| --- | --- |
| `PC.sv` | Armazena o endereço da instrução atual. |
| `pc_plus4.sv` | Calcula o endereço sequencial `PC + 4`. |
| `instr_mem.sv` | Carrega e disponibiliza as instruções de `genius.mem`. |
| `decoder_control.sv` | Decodifica a instrução e produz os sinais de controle. |
| `regfile.sv` | Implementa o banco de 32 registradores de 32 bits, com `x0` fixo em zero. |
| `immediate_extended.sv` | Gera os imediatos para diferentes formatos de instrução. |
| `alu.sv` | Executa operações aritméticas, lógicas, comparações e deslocamentos. |
| `data_memory.sv` | Implementa RAM de 16 KiB, acessos `LB/LW/SB/SW` e periféricos MMIO. |
| `pc_target.sv` | Calcula endereços de desvio e salto. |
| `mux_pc.sv`, `mux_alu_a.sv`, `mux_alu_src.sv` | Selecionam os operandos e o próximo endereço do programa. |
| `write2.sv` | Seleciona o resultado a ser escrito no banco de registradores. |

O decodificador contempla operações aritméticas/lógicas, acesso à memória, desvios condicionais e saltos, além de `LUI` e `AUIPC`. **Não é uma implementação completa de toda a ISA RISC-V**: por exemplo, `ECALL` e deslocamentos aritméticos à direita (`SRA/SRAI`) não estão implementados no decodificador apresentado.

### Mapeamento de entradas e saídas (MMIO)

| Endereço hexadecimal | Periférico | Operação |
| --- | --- | --- |
| `0xF0000000` | LED 0 | Leitura/escrita |
| `0xF0000004` | LED 1 | Leitura/escrita |
| `0xF0000008` | LED 2 | Leitura/escrita |
| `0xF000000C` | LED 3 | Leitura/escrita |
| `0xF0000100` | Chaves (`switches_in`) | Leitura |
| `0xF0000200` | D-Pad: cima | Leitura |
| `0xF0000204` | D-Pad: baixo | Leitura |
| `0xF0000208` | D-Pad: esquerda | Leitura |
| `0xF000020C` | D-Pad: direita | Leitura |
| `0xF0000300` | Contador de ciclos (`rtc_counter`) | Leitura |

A RAM de dados ocupa a faixa `0x00000000` a `0x00003FFF` (16 KiB). O contador em `0xF0000300` é incrementado a cada ciclo de clock; **não representa diretamente tempo em segundos**.

## Funcionamento do Genius

O programa apresenta uma sequência de cores por meio dos quatro LEDs. O jogador deve reproduzir a sequência utilizando o D-Pad. A cada rodada concluída corretamente, o programa acrescenta uma nova cor à sequência; uma resposta incorreta encerra a partida atual.

As configurações são enviadas por `switches_in_0`:

| Bit | Função |
| --- | --- |
| `SW[1:0]` | Dificuldade: `00` fácil, `01` média, `10` difícil. |
| `SW[2]` | Seleção de modo rápido. |
| `SW[3]` | Modo manual (`Mando eu`). |
| `SW[4]` | Comando de início (`START`). |

O testbench utiliza `dpad_in_0[0:3]` para inserir as quatro cores. As saídas RGB esperadas na verificação são:

| LED | Cor | Valor esperado |
| --- | --- | --- |
| `led0_out_0` | Vermelho | `0x00FF0000` |
| `led1_out_0` | Verde | `0x0000FF00` |
| `led2_out_0` | Azul | `0x000000FF` |
| `led3_out_0` | Amarelo | `0x00FFFF00` |

## Ambiente de verificação UVM

O arquivo `genius_uvm_tb.sv` reúne os componentes do ambiente de verificação:

- **Sequence item e sequences:** representam comandos de início, entrada de cor e reset.
- **Sequencer:** organiza o envio das transações.
- **Driver:** converte as transações em sinais de `switches_in_0`, `dpad_in_0` e `reset_0`.
- **Monitor:** observa os LEDs, detecta novos eventos e identifica ativação simultânea indevida de mais de um LED.
- **Scoreboard:** registra as cores observadas e confere os valores RGB esperados.
- **Test:** conduz os cenários, compara sequências, verifica as condições esperadas e contabiliza falhas.
- **Cobertura funcional:** coleta amostras referentes a cores, dificuldade, velocidade e modo de jogo.

A sequência pseudorrandômica utilizada nas verificações do modo **Siga** é reproduzida pelo testbench a partir de um **LFSR de 8 bits**, com semente `0xA5`, para comparar a saída observada com a sequência esperada.

### Cenários automatizados

| Caso | Cenário |
| --- | --- |
| 1 | Modo Siga: fácil e lento. |
| 2 | Modo Siga: médio e rápido. |
| 3 | Modo Siga: difícil e rápido, incluindo verificação das quatro cores. |
| 4 | Resposta incorreta no meio da sequência e reinício por `START`. |
| 5 | Reset durante a partida, enquanto o processador aguarda a resposta do jogador. |
| 6 | Modo manual: LED 0. |
| 7 | Modo manual: LED 1. |
| 8 | Modo manual: LED 2. |
| 9 | Modo manual: LED 3. |

Além dos testes por cenário, o ambiente verifica a correspondência das cores RGB, a presença das quatro cores ao longo da suíte, a ausência de múltiplos LEDs ativos simultaneamente e a relação esperada entre as durações dos pulsos nos modos rápido e lento.

### Modos de execução

O testbench dispõe de dois modos:

| Modo | Rodadas: fácil / médio / difícil | Seleção |
| --- | --- | --- |
| Rápido | `3 / 4 / 6` | Padrão |
| Completo | `8 / 16 / 32` | Argumento de simulação `GENIUS_FULL` |

O modo completo é selecionado quando o simulador recebe a opção de tempo de execução correspondente a **`+GENIUS_FULL`** (ou seu equivalente como *test plusarg* no XSim). Ambos os modos também executam os cenários de erro, reset e operação manual.

## Estrutura dos arquivos principais

```text
PBL03-Giovanni/
├── PBL03-Giovanni.xpr
├── PBL03-Giovanni.srcs/
│   ├── sources_1/
│   │   ├── new/
│   │   │   ├── *.sv                 # Módulos do processador
│   │   │   └── genius.mem           # Programa carregado na memória de instruções
│   │   └── bd/RISCVPROJECT/
│   │       └── RISCVPROJECT.bd     # Diagrama de blocos do processador
│   └── sim_1/
│       ├── imports/Jogo_Genius/
│       │   └── genius_uvm_tb.sv    # Ambiente UVM e top da simulação
│       └── new/
│           ├── tb_riscv.sv         # Testbench funcional alternativo
│           └── genius.txt          # Arquivo auxiliar de instruções
└── PBL03-Giovanni.sim/
    └── sim_1/behav/xsim/
        ├── data.mem                # Inicialização da RAM (presente no pacote)
        └── simulate.log            # Registro de simulação incluído
```

Os diretórios `.cache`, `.gen`, `.runs` e `.sim` contêm, em grande parte, artefatos produzidos pelo Vivado. Para um repositório Git reproduzível, é preferível versionar as **fontes RTL, a descrição do Block Design, os arquivos de memória e os testes**, evitando depender exclusivamente de arquivos gerados.

## Como abrir e simular

1. Instale o **AMD Vivado 2026.1**, com suporte ao **XSim** e **UVM**.
2. Abra `PBL03-Giovanni.xpr` no Vivado.
3. Confira se todos os arquivos de código e de memória estão presentes no projeto. Se aparecerem referências quebradas, remova os caminhos inválidos e adicione novamente os arquivos correspondentes pela interface do Vivado.
4. Verifique se `genius.mem` está disponível para o `$readmemh` do módulo `instr_mem.sv`.
5. **Atenção ao `data.mem`:** no ZIP recebido, ele aparece em `PBL03-Giovanni.sim/sim_1/behav/xsim/data.mem`, mas o projeto `.xpr` também aponta para um caminho externo à pasta do projeto. Para reproduzir a simulação em outro computador, copie esse arquivo para uma pasta de fontes versionada, adicione-o ao projeto e garanta sua disponibilidade no diretório de execução da simulação.
6. Em **Simulation Sources**, selecione `genius_uvm_tb` como **Simulation Top**.
7. Execute **Run Simulation → Run Behavioral Simulation**.
8. Analise as mensagens `CASE_PASS`, `CASE_FAIL`, `UVM_ERROR`, `UVM_FATAL` e o resumo `FINAL`, quando a suíte terminar. Também é possível acompanhar os sinais de depuração `dbg_case`, `dbg_stage`, `dbg_round`, `dbg_step`, `dbg_checks` e `dbg_failures` no Waveform Viewer.

O clock de simulação é gerado com período de **10 ns** (100 MHz). O arquivo `tb_riscv.sv` oferece um teste funcional mais simples, mas o teste selecionado no projeto recebido é `genius_uvm_tb`.

### Estado da verificação no pacote recebido

O arquivo `simulate.log` incluído no ZIP apresenta **execução parcial da suíte rápida**: seis casos chegaram a registrar `CASE_PASS`, mas existe um `UVM_ERROR` na verificação da velocidade dos pulsos (`lento=0`, `rápido=340000`, nos valores registrados pelo simulador). O log fornecido termina antes do resumo final da suíte.


## Observações

- O `RISCVPROJECT_wrapper` é o topo do hardware; `genius_uvm_tb` é o topo da simulação UVM.
- A memória de instruções usa `genius.mem`; a memória de dados é inicializada a partir de `data.mem`.
- O teste com reset em andamento ocorre durante a espera pela entrada do jogador, sem LEDs ativos. Ele **não verifica** reset durante um pulso de LED aceso.
- A versão do arquivo `.xpr` contém algumas referências antigas a diretórios externos, como `PBL03-Cimatec` e uma pasta do OneDrive. Esses caminhos devem ser corrigidos ao abrir o projeto em outra máquina.

---

**Contexto:** PBL03 — desenvolvimento de processador RISC-V e verificação funcional de uma aplicação embarcada com SystemVerilog e UVM.
