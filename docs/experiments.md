# Log cronológico de experimentos — Intelbras MCA 1002

Registro cronológico de cada experimento/medição realizado. Datas de entradas anteriores a 2026-09-05 foram reconstruídas a partir das datas dos commits originais do repositório (`git log`); a data exata do experimento em si pode ter sido ligeiramente anterior à do commit.

---

## 2026-09-03 — Continuidade de J3 com GND

**Objetivo:** identificar qual(is) pad(s) do header J3 são GND.

**Conexão:** nenhuma alimentação; placa totalmente desligada. Multímetro em modo continuidade, ponta de referência na blindagem metálica externa do conector micro-USB.

**Comando:** medição manual de continuidade, pad a pad, J3-1 a J3-6.

**Resultado:**

| Pad | Continuidade com GND |
|---|---|
| J3-1 | Não |
| J3-2 | Sim |
| J3-3 | Não |
| J3-4 | Não |
| J3-5 | Não |
| J3-6 | Não |

**Interpretação:** J3-2 é GND.

**Status:** CONFIRMED.

**Próximo passo:** medir tensão DC dos demais pads com a placa energizada, usando J3-2 como referência.

---

## 2026-09-03 — Tensão DC de J3 (placa energizada) e continuidade com pinos do REX3B21

**Objetivo:** identificar 3,3 V, possíveis linhas de sinal, e mapear J3 diretamente aos pinos do módulo REX3B21.

**Conexão:** placa energizada normalmente pela própria fonte. Multímetro: ponta preta em J3-2 (GND), ponta vermelha em cada pad. Em paralelo, com a placa desligada, testada continuidade direta entre cada pad de J3 e os pinos do REX3B21 (5, 7, 15, 17, 19).

**Comando:** medição manual de tensão DC e continuidade ponto a ponto.

**Resultado — tensão DC:**

| Pad | Tensão medida |
|---|---:|
| J3-1 | 3,355 V |
| J3-2 | 0 V / GND |
| J3-3 | 3,295 V |
| J3-4 | 3,337 V |
| J3-5 | 3,340 V |
| J3-6 | ~1,2 V |

**Resultado — continuidade com REX3B21:**

| J3 | Pino REX3B21 | Função |
|---|---:|---|
| J3-1 | 5 | 3,3 V / VCC / VTref |
| J3-2 | 7 | GND |
| J3-3 | 19 | RESET |
| J3-4 | 17 | PA2 / SWDIO |
| J3-5 | 15 | PA1 / SWCLK |
| J3-6 | — | não identificado |

**Interpretação:** J3 é o header de debug/programação SWD do módulo REX3B21/EFR32MG21 (VTref, GND, RESET, SWDIO, SWCLK). J3-6 permanece não identificado — sem continuidade com os pinos testados, ~1,2 V quando energizado (pode ser SWO, UART, outro GPIO, ou sinal condicionado).

**Status:** CONFIRMED (pinout J3-1 a J3-5). UNKNOWN (J3-6).

**Próximo passo:** conectar um debugger SWD compatível com 3,3 V (VTref/GND/RESET/SWDIO/SWCLK) e tentar apenas identificação do MCU, sem erase/write/unlock. Não conectar J3-6 até identificá-lo.

---

## 2026-09-05 — Primeiro contato SWD e identificação do core via OpenOCD

**Objetivo:** estabelecer conexão SWD real com o EFR32MG21 e confirmar a identidade do core, usando apenas operações read-only.

**Conexão:**

```text
Pico GP2 -> J3-5 (SWCLK)
Pico GP3 -> J3-4 (SWDIO)
Pico GND -> J3-2 (GND)
```

RESET (J3-3) não conectado. J3-1 (VTref) não conectado ao Pico. J3-6 não conectado. MCA 1002 alimentado pela própria fonte; Pico alimentado pelo USB do computador.

Probe: Raspberry Pi Pico com firmware `debugprobe_on_pico.uf2` (Raspberry Pi Debug Probe), enumerado como CMSIS-DAPv2 (VID:PID `2e8a:000c`, firmware `2.0.0`).

Software: xPack OpenOCD `0.12.0+dev-02228-ge5888bda3-dirty` (build 2025-10-04).

**Comando:**

```tcl
swd newdap efr32 cpu -expected-id 0x6ba02477
dap create efr32.dap -chain-position efr32.cpu
target create efr32.cpu cortex_m -dap efr32.dap -ap-num 0
```

Classificação: READ-ONLY / SAFE (configuração de enumeração do DAP/target; nenhuma escrita no dispositivo).

**Resultado:**

```text
SWD DPIDR = 0x6BA02477
Cortex-M33 r0p3 processor detected
8 breakpoints
4 watchpoints
Examination succeed
```

**Interpretação:** comunicação SWD funcional com o Debug Port do EFR32MG21. O DPIDR lido bate com o esperado para a família EFR32MG21 (`-expected-id 0x6ba02477` validado pelo próprio OpenOCD). O core foi corretamente enumerado como Cortex-M33 r0p3.

**Status:** CONFIRMED.

**Próximo passo:** ler um registrador padrão ARM para confirmar independentemente a identidade do core.

---

## 2026-09-05 — Leitura de CPUID via AHB-AP

**Objetivo:** confirmar de forma independente (via registrador padrão ARM, fora do enumeration do OpenOCD) a identidade do core, e validar que a leitura de memória via AHB-AP está funcional.

**Conexão:** mesma da entrada anterior (Pico -> J3-5/J3-4/J3-2, RESET não conectado).

**Comando:**

```tcl
mdw 0xE000ED00 1
```

Classificação: READ-ONLY / SAFE. `mdw` lê uma palavra de 32 bits via o AP configurado (AHB-AP, ap-num 0); não escreve nada. `0xE000ED00` é o registrador `CPUID`, no espaço de sistema ARMv8-M (PPB), padrão em toda a família Cortex-M — não é um endereço específico do EFR32.

**Resultado:**

```text
0xE000ED00: 410FD213
```

**Interpretação:** CPUID = `0x410FD213` decodifica como Implementer = ARM (`0x41`), PartNo = Cortex-M33 (`0xD21`), Revision = r0p3. Consistente com o que o OpenOCD já havia detectado na etapa de exame do core. Confirma também que o AHB-AP permite leitura de memória mapeada do sistema (ao menos no espaço PPB) — não indica ainda nada sobre acesso à flash principal ou SRAM, nem sobre o estado de debug/security lock.

**Status:** CONFIRMED.

**Próximo passo:** pesquisar documentação oficial Silicon Labs para o EFR32MG21 (Series 2) e determinar o endereço correto de DEVINFO para essa série (não extrapolar de Series 1), como próximo passo para identificar part number exato, flash size e RAM size — antes de qualquer leitura, verificar também se há indicação de debug/security lock.

---

## 2026-09-05 — Enumeração de Access Ports: leitura do IDR do AP0

**Objetivo:** enumerar/confirmar os Access Ports do DAP, começando pelo AP0 já usado para leitura de CPUID, usando um comando documentado oficialmente como somente leitura.

**Conexão:** mesma de sempre (Pico -> J3-5/J3-4/J3-2, RESET não conectado). Desta vez o OpenOCD foi executado em modo batch (não interativo): `openocd -f interface/cmsis-dap.cfg -c "transport select swd" -c "adapter speed 1000" -c "swd newdap efr32 cpu -expected-id 0x6ba02477" -c "dap create efr32.dap -chain-position efr32.cpu" -c "target create efr32.cpu cortex_m -dap efr32.dap -ap-num 0" -c "init" -c "efr32.dap apid 0" -c "shutdown"`.

**Comando:**

```tcl
efr32.dap apid 0
```

Classificação: READ-ONLY / SAFE. Confirmado contra a documentação oficial do OpenOCD (`doc/openocd.texi`): `apid` apenas exibe o IDR do AP indicado; não existe variante de escrita para este comando.

**Resultado:**

```text
Info : SWD DPIDR 0x6ba02477
Info : [efr32.cpu] Cortex-M33 r0p3 processor detected
Info : [efr32.cpu] Examination succeed
0x84770001
```

**Interpretação:** IDR decodificado como Revision=`0x8`, JEP106 continuation=`0x4`, JEP106 identity=`0x3B` (ARM Limited), Class=`0x8` (MEM-AP), Variant=`0x0`, Type=`0x1` (AHB-AP). Confirma que AP0 é um MEM-AP AHB-AP válido da ARM — consistente com as leituras de CPUID já feitas via esse mesmo AP.

**Status:** CONFIRMED.

**Próximo passo:** verificar se existem outros Access Ports além do AP0 (tentar `apid` em outros números), e/ou avançar para verificar o estado de debug/security lock sem alterá-lo, e para identificação de DEVINFO após confirmar o endereço correto no Reference Manual oficial.

---

## 2026-09-05 — Enumeração de AP1: identificado como DCI/AAP (Silicon Labs) — ⚠️ achado de segurança


**Objetivo:** continuar a enumeração de Access Ports, testando AP1.

**Conexão:** mesma configuração de sempre (Pico -> J3-5/J3-4/J3-2, RESET não conectado), OpenOCD em modo batch, mesmo bring-up de sempre.

**Comando:**

```tcl
efr32.dap apid 1
```

Classificação: READ-ONLY / SAFE (comando `apid`, sem variante de escrita).

**Resultado:**

```text
0x54770002
```

**Interpretação:** decodificado como Class=MEM-AP, Type=APB-AP, ARM. Pesquisa contra o AN1190 oficial da Silicon Labs ("Series 2 Secure Debug") indica que esse segundo AP corresponde à **Debug Challenge Interface (DCI)**, também chamada de Authentication Access Port (AAP) em fontes de comunidade — o mecanismo oficial de debug lock/unlock **e mass erase** (`Erase Device`: "Performs a device mass erase and resets the debug configuration to its initial unlocked state."). Isso confirma fisicamente, no barramento SWD deste dispositivo, o mecanismo sobre o qual já havíamos sido alertados como potencialmente destrutivo.

**Status:** CONFIRMED (AP1 existe, é APB-AP ARM). INFERRED, alta confiança, pendente de AN1303 (corresponde à DCI/AAP). UNKNOWN (protocolo exato de registradores).

**Próximo passo:** ler o AN1303 (Silicon Labs, oficial) — "Programming Series 2 Devices using the Debug Challenge Interface (DCI) and Serial Wire Debug (SWD)" — para entender o protocolo de registradores da DCI antes de qualquer outro comando em AP1. Nenhum `apreg`/`dpreg`/`mdw` em AP1 até essa pesquisa estar completa.

---

## 2026-09-05 — Pesquisa do protocolo DCI (sucessor web do AN1303) + execução de `Read Lock Status` (0x4311)

**Objetivo:** entender o protocolo de registradores da DCI (item pendente da entrada anterior) e, com isso, verificar o estado de debug/security lock do EFR32MG21 sem alterá-lo.

**Pesquisa (sem tocar hardware):** o PDF do AN1303 está deprecated desde Simplicity SDK Suite 2025.12.0 (contém só uma capa redirecionando para `docs.silabs.com`). Consultadas as páginas web oficiais sucessoras: "Debug Challenge Interface (DCI)" e "SE Command List" (`docs.silabs.com/connect-stack/latest/efr32-dci-swd-programming/...`). Detalhes completos e tabela de comandos com classificação de risco documentados em [`swd.md`](swd.md).

**Ponto de decisão:** `Read Lock Status` (Command ID `0x4311`) é documentado pela Silicon Labs como consulta somente leitura, sempre disponível, sem efeito colateral persistente — mas mecanicamente exige uma escrita de registrador (`apreg`) no mailbox volátil do Secure Engine (via AP1), tocando a letra da regra de "nenhuma escrita" desta fase mesmo não tocando seu espírito. Apresentado ao responsável pelo projeto, que autorizou explicitamente a execução.

**Conexão:** mesma de sempre (Pico -> J3-5/J3-4/J3-2, RESET não conectado). OpenOCD em modo batch.

**Comando:** sequência completa de `efr32.dap apreg 1 ...` implementando o protocolo DCI oficial (CSW, TAR->DCI_STATUS/DCI_WDATA/DCI_RDATA, DRW) para enviar o comando `Read Lock Status` e ler a resposta — sequência completa documentada em [`swd.md`](swd.md#read-lock-status-0x4311-executado--resultado`).

Classificação: cada `apreg` individual é uma leitura ou uma escrita em um registrador de mailbox volátil do Secure Engine — não escreve em flash, NVM, nem em nenhuma configuração persistente do dispositivo. Autorizado explicitamente pelo responsável do projeto como exceção a esta fase (ver acima).

**Resultado:**

```text
--STATUS before--
0x00000000
--WRITE word0 (length=8)--
--STATUS after word0--
0x00000000
--WRITE word1 (cmd=0x4311 Read Lock Status)--
--STATUS poll 1--
0x00000100
--READ response word0 (len+code)--
0x00000008
--STATUS poll 4--
0x00000100
--READ response word1 (payload)--
0x00000002
```

**Interpretação:** resposta = comprimento 8 bytes, código `SE_RESPONSE_OK` (0). Payload = `0x00000002` → apenas bit 1 (`Device erase enabled`) setado; bit 0 (`Debug lock`), bit 2 (`Secure debug lock`) e bit 5 (`Debug lock hardware status`) todos zero. **O EFR32MG21 está em "Standard Debug Unlock": debug port aberto, sem lock ativo, secure debug desabilitado, comando `Erase Device` habilitado (mas não executado).** Isso confirma e explica por que todas as leituras via SWD feitas até agora funcionaram sem qualquer restrição, e reforça por que `Erase Device` (`0x430F`) deve permanecer permanentemente fora de cogitação.

**Status:** CONFIRMED.

**Próximo passo:** com o estado de debug/lock confirmado como totalmente aberto, avançar para identificar a variante exata do EFR32MG21 via DEVINFO (part number, flash size, RAM size) — usando o AHB-AP (AP0), após confirmar o endereço correto de DEVINFO no Reference Manual oficial (Series 2). Continuar sem tocar novamente em AP1 além do que já foi feito.

---

## 2026-09-05 — Leitura de DEVINFO_PART/MEMINFO/MSIZE — resultado ambíguo, documentado com honestidade

**Objetivo:** identificar part number, flash size e RAM size do EFR32MG21 via DEVINFO.

**Pesquisa prévia:** baixado o EFR32xG21 Wireless Gecko Reference Manual oficial (Rev 1.0, `silabs.com`). Confirmado `FLASH_DEVINFO` base = `0x0FE0E000` via Figure 4.1 (System Address Space). Corrigido um erro de fonte de comunidade: um endereço `0x0FE08000` encontrado antes numa pesquisa era na verdade `FLASH_USERDATA`, não `DEVINFO` — descartado. Offsets confirmados em 6.4.1/6.4.2 (checados em duas extrações independentes do PDF, "layout" e "raw", que bateram exatamente uma com a outra e com um header CMSIS gerado automaticamente pela própria Silicon Labs, usado só como checagem cruzada): `DEVINFO_PART`=`0x004`, `DEVINFO_MEMINFO`=`0x008`, `DEVINFO_MSIZE`=`0x00C`.

**Conexão:** mesma de sempre (Pico -> J3-5/J3-4/J3-2, RESET não conectado), via AP0 (AHB-AP) — AP1/DCI não tocado nesta entrada.

**Comando:**

```tcl
mdw 0x0FE0E004 3
```

Classificação: READ-ONLY / SAFE (mesmo mecanismo de leitura já usado para CPUID).

**Resultado:**

```text
0x0fe0e004: 00000015 40024024 02010013
```

**Interpretação:**

`DEVINFO_PART` = `0x00000015`: `FAMILY`[29:24]=`0`, `FAMILYNUM`[21:16]=`0`, `DEVICENUM`[15:0]=`21`. `DEVICENUM=21` bate exatamente com "MG**21**", mas `FAMILY` deveria ser `1` (MG) e `FAMILYNUM` deveria ser `21` segundo a convenção documentada — ambos vieram `0`.

`DEVINFO_MSIZE` = `0x02010013`: `SRAM`[26:16]=`513` KB, `FLASH`[15:0]=`19` KB. **`SRAM`=513 KB é fisicamente impossível** — o datasheet oficial do EFR32MG21 especifica RAM máxima de 96 KB. `FLASH`=19 KB também não corresponde a nenhum SKU conhecido (512/768/1024 KB).

**Status:** UNKNOWN / INCONCLUSIVO. `DEVICENUM=21` é um indício forte (mas não prova) de EFR32MG21. Os campos `FAMILY`/`FAMILYNUM`/`MSIZE` não decodificam para valores fisicamente plausíveis com o layout de bits confirmado (checado 3x contra o RM oficial e um header CMSIS independente — a decodificação de bits está correta; o que está em aberto é por que os valores lidos não fazem sentido). Hipóteses não verificadas: (a) esses campos legados podem não ser preenchidos da forma esperada em módulos de terceiros (Rexense) vs. SoCs "nus" da Silicon Labs; (b) pode haver algum efeito de leitura ainda não identificado específico dessa região de flash. **Não tratar `SRAM=513KB`/`FLASH=19KB` como fatos sobre o hardware.**

**Próximo passo:** ler `DEVINFO_MODULENAME0`..`MODULENAME6` (offsets `0x130`–`0x148`, confirmados no RM oficial) — região que armazena até 28 caracteres ASCII do nome do módulo (4 caracteres por word, `0xFF`=não escrito). Deve identificar o módulo de forma direta e inequívoca (ex.: "REX3B21S"), evitando a ambiguidade encontrada nos campos numéricos de FAMILY/MSIZE.

---

## 2026-09-05 — Leitura de DEVINFO_MODULENAME0..6 — resultado inesperado, e teste de reprodutibilidade

**Objetivo:** ler o nome do módulo em ASCII (`DEVINFO_MODULENAME0`..`6`, offsets `0x130`-`0x148`) para identificação direta, e depois verificar se as leituras estranhas de DEVINFO são reprodutíveis.

**Conexão:** mesma de sempre (Pico -> J3-5/J3-4/J3-2, RESET não conectado), via AP0.

**Comando 1:**

```tcl
mdw 0x0FE0E130 7
```

**Resultado 1:**

```text
0x0fe0e130: a80200d8 00001300 a8020110 00000549 a80200ac 16d55534 a80180ac
```

**Interpretação 1:** isso **não é ASCII** (bytes como `D8 00 02 A8` não são caracteres imprimíveis) e **não é o padrão `0xFF`** (não escrito) previsto pela especificação para essa região. Padrão lembra dados/ponteiros (repetição de `0xA802` no byte alto de vários words), não uma string de nome de módulo.

**Comando 2 (reprodutibilidade):** repetição, em nova conexão SWD independente (novo processo OpenOCD), de `mdw 0x0FE0E004 3` e `mdw 0x0FE0E130 7`.

**Resultado 2:**

```text
0x0fe0e004: 00000015 40024024 02010013
0x0fe0e130: a80200d8 00001300 a8020110 00000549 a80200ac 16d55534 a80180ac
```

**Interpretação 2:** valores **idênticos, byte a byte**, em relação às leituras anteriores. Isso descarta ruído de barramento/falha transitória de leitura como explicação — é conteúdo real e estável de memória nesses endereços. O que permanece sem explicação é por que esse conteúdo não corresponde ao que o Reference Manual descreve para essas posições (nem os valores de `FAMILY`/`MSIZE` fazem sentido físico, nem `MODULENAME` parece ASCII ou "não escrito").

**Status:** CONFIRMED (leituras são estáveis/reprodutíveis). UNKNOWN (por que o conteúdo não corresponde à especificação genérica da Silicon Labs para essas posições — hipóteses em aberto: endianness/mapeamento de página específico deste die, diferença de revisão do EFR32MG21 usada neste módulo Rexense vs. a documentada no RM Rev 1.0, ou possibilidade de os campos `DEVINFO_MODULENAME` simplesmente não serem usados por este fabricante de módulo).

**Próximo passo:** ler `DEVINFO_INFO` (offset `0x000`, "DI Page Version", campo `DEVINFOREV`) como teste de sanidade adicional — é um campo simples, documentado como "initially 1", que serve de referência de baixo risco para saber se o problema está na região de `0x004` em diante ou se afeta a página inteira desde o primeiro word.

---

## 2026-09-05 — Leitura de DEVINFO_INFO (offset 0x000) — reavaliação da hipótese

**Objetivo:** ler `DEVINFO_INFO` como teste de sanidade adicional para a página DEVINFO.

**Comando:**

```tcl
mdw 0x0FE0E000 1
```

**Resultado:**

```text
0x0fe0e000: 40028010
```

**Interpretação:** decodificado por bits confirmados no RM oficial (seção 6.4.2.1, extração "raw" sem ambiguidade): `DEVINFOREV`[31:24]=`0x40`(64), `PRODREV`[23:16]=`0x02`(2), `CRC`[15:0]=`0x8010`. `PRODREV`=2 é um valor pequeno e plausível; `CRC`=`0x8010` tem exatamente a cara de um CRC-16 real (deve parecer "aleatório" por natureza — isso bate). `DEVINFOREV`=64 é alto frente à narrativa do manual ("initially 1"), mas essa frase descreve a origem histórica do esquema de versionamento da página, não necessariamente o valor esperado hoje — não é uma contradição física como o `SRAM`=513KB encontrado antes.

**Reavaliação:** este resultado sugere que a leitura de DEVINFO **está funcionando corretamente em geral** (o CRC parece genuíno). A explicação mais provável para as estranhezas anteriores (`FAMILY`/`FAMILYNUM`=0 em `PART`, `MSIZE` fisicamente implausível, `MODULENAME` não-ASCII) passa a ser: **esses são campos de conveniência da Silicon Labs (OPN/família/nome de módulo) que a Rexense, como integradora terceira, pode não preencher** — em vez de um problema de endereço base ou de mecanismo de leitura via SWD. Isso não está provado, mas é agora a hipótese mais provável.

**Status:** CONFIRMED — leitura de DEVINFO funcional e reproduzível em geral. INFERRED — campos de conveniência (FAMILY/FAMILYNUM/MSIZE/MODULENAME) provavelmente não populados por este fabricante de módulo, em vez de indicar erro de leitura. UNKNOWN — confirmação definitiva dessa hipótese; flash size e RAM size reais deste chip continuam não confirmados por nenhuma via até agora.

**Próximo passo:** ponto de decisão apresentado ao responsável do projeto — continuar insistindo nos campos numéricos da DEVINFO (ex.: `DEVINFO_PKGINFO`, `DEVINFO_EUI64`) ou mudar de prioridade (ex.: mapear o início da flash principal em `0x00000000`, ou buscar `EUI-64`/calibração via outros campos, ou seguir para NVM3/bootloader).
