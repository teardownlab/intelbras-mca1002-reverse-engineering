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

**Decisão do responsável do projeto:** ler `EUI-64` a seguir.

---

## 2026-09-05 — Leitura de DEVINFO_EUI64L/H

**Objetivo:** ler o EUI-64 do rádio Zigbee para checar se o OUI bate com um prefixo IEEE conhecido da Silicon Labs/Ember, como teste independente da hipótese de "campos de conveniência não populados pela Rexense".

**Pesquisa prévia:** offsets confirmados no RM oficial (extração raw, seções 6.4.2.13/6.4.2.14): `DEVINFO_EUI64L`=`0x048` (`UNIQUEL`[31:0], 4 bytes menos significativos), `DEVINFO_EUI64H`=`0x04C` (`OUI64`[31:8], 3 bytes do OUI IEEE; `UNIQUEH`[7:0], byte mais significativo do identificador único).

**Comando:**

```tcl
mdw 0x0FE0E048 2
```

Classificação: READ-ONLY / SAFE (mesmo `mdw`, via AP0).

**Resultado:**

```text
0x0fe0e048: 40004018 0000000b
```

**Interpretação:** `EUI64L`=`0x40004018` (`UNIQUEL`), `EUI64H`=`0x0000000B` → `OUI64`=`0x000000`, `UNIQUEH`=`0x0B`. **EUI-64 completo = `00:00:00:0B:40:00:40:18`.** O OUI `00:00:00` **não é um OUI real da Silicon Labs/Ember**. Um rádio Zigbee em produção precisa de um EUI-64 único e válido para operar na rede — este valor parece não-provisionado ou incorreto.

Isso é o **quarto** campo seguido (depois de `FAMILY`/`FAMILYNUM`, `MSIZE`, `MODULENAME`) a vir com conteúdo implausível/não reconhecível, enquanto apenas `DEVINFO_INFO` (campo interno de CRC/versionamento, não configurável por integrador terceiro) pareceu correto. Isso **enfraquece** a hipótese de "a Rexense não preenche campos de conveniência" — EUI-64 não é opcional para um rádio Zigbee funcionar, então não deveria estar ausente num dispositivo em produção.

**Status:** CONFIRMED (leitura reproduzível de bytes reais). UNKNOWN — causa da discrepância entre conteúdo esperado e lido permanece em aberto; hipótese de "campos não populados" enfraquecida mas não descartada.

**Próximo passo:** ler o vetor de interrupções do Cortex-M33 em `0x00000000` (Initial Stack Pointer e Reset Handler) — teste universal, independente de tabelas específicas da Silicon Labs, que serve tanto de diagnóstico adicional (os valores devem parecer um endereço de RAM e um endereço de código Thumb válidos) quanto de primeiro passo real rumo ao mapeamento do firmware atual.

---

## 2026-09-05 — Leitura do vetor de interrupções em 0x00000000 — firmware real confirmado

**Objetivo:** ler o início do vetor de interrupções do Cortex-M33 (flash principal, `0x00000000`) como teste universal, independente da Silicon Labs, e primeiro passo real de mapeamento do firmware.

**Comando:**

```tcl
mdw 0x00000000 2
```

Classificação: READ-ONLY / SAFE (mesmo `mdw`, via AP0).

**Resultado:**

```text
0x00000000: 20001338 00003219
```

**Interpretação:** word 0 (Initial Stack Pointer) = `0x20001338` — dentro da faixa de RAM (`0x20000000`+), exatamente como esperado. Word 1 (Reset Handler) = `0x00003219` — bit 0 setado (modo Thumb), endereço plausível dentro da flash. **Isso é um vetor de interrupções ARM Cortex-M válido e coerente**, bem diferente da região DEVINFO. Confirma que a leitura via SWD/AHB-AP funciona corretamente em geral, e que **há firmware real e válido gravado na flash principal** — a região DEVINFO parece ter um problema específico dela (causa ainda não identificada), não um problema geral de leitura.

**Status:** CONFIRMED — firmware presente e vetor de interrupções válido. Boa notícia para a prioridade de preservação do firmware original.

**Próximo passo:** ler os primeiros 16 words de `0x00000000` (Stack Pointer + as 15 exceções padrão do Cortex-M: NMI, HardFault, MemManage, BusFault, UsageFault, SVCall, DebugMonitor, PendSV, SysTick) para começar a montar o mapa real do firmware atual.

---

## 2026-09-05 — Leitura dos primeiros 16 words do vetor de interrupções

**Comando:** `mdw 0x00000000 16` — Classificação: READ-ONLY / SAFE.

**Resultado:**

```text
0x00000000: 20001338 00003219 0000030f 00000c25 00000e13 00000e1b 000019bf 00000000
0x00000020: 00000000 00000000 00000040 00001f33 00002083 00000074 000020e7 0000281b
```

**Interpretação:** mapeando pelos índices padrão do Cortex-M (0=SP, 1=Reset, 2=NMI, 3=HardFault, 4=MemManage, 5=BusFault, 6=UsageFault, 7–9=Reservado, 10=Reservado, 11=SVCall, 12=DebugMonitor, 13=Reservado, 14=PendSV, 15=SysTick): todos os handlers implementados (`Reset`,`NMI`,`HardFault`,`MemManage`,`BusFault`,`UsageFault`,`SVCall`,`DebugMonitor`,`PendSV`,`SysTick`) são endereços ímpares (Thumb) plausíveis dentro de uma flash pequena/modesta; os slots reservados 7, 8 e 9 são `0x00000000` como esperado. Os slots 10 e 13 (reservados) têm valores pequenos não-nulos (`0x40`, `0x74`) — não são ponteiros de código, possivelmente uso específico do GSDK/bootloader da Silicon Labs (ex.: informação de aplicação/checksum); não é motivo de preocupação, é padrão comum em imagens Gecko SDK.

**Status:** CONFIRMED — vetor de interrupções completo e coerente, mais uma confirmação de firmware real e válido.

---

## 2026-09-05 — ApplicationProperties_t: bootloader identificado, e localização da aplicação Zigbee real

**Objetivo:** usar a convenção oficial da Silicon Labs (Simplicity Commander grava um ponteiro para uma struct `ApplicationProperties_t` na palavra 13 do vetor de interrupções) para identificar definitivamente o tipo de firmware presente, evitando a ambiguidade da DEVINFO.

**Pesquisa prévia:** código-fonte oficial obtido via `gh api` do repositório `SiliconLabs/simplicity_sdk` (`platform/bootloader/api/application_properties.h`), não apenas resumos de busca. Magic de 16 bytes confirmado: `13 b7 79 fa c9 25 dd b7 ad f3 cf e0 f1 b6 14 b8`. Bits de `app.type`: `APPLICATION_TYPE_ZIGBEE`=bit0 (`0x01`), `APPLICATION_TYPE_BOOTLOADER`=bit6 (`0x40`). Layout do struct (80 bytes/20 words): magic(16)+structVersion(4)+signatureType(4)+signatureLocation(4)+app.type(4)+app.version(4)+app.capabilities(4)+app.productId(16)+cert*(4)+longTokenSectionAddress*(4)+decryptKey(16).

**Comando 1** (struct apontada por word 13 do vetor em `0x00000000`, endereço `0x00000074`):

```tcl
mdw 0x00000074 20
```

**Resultado 1:**

```text
0x00000074: fa79b713 b7dd25c9 e0cff3ad b814b6f1 00000101 00000000 000032b4 00000040
0x00000094: 01080000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0x000000b4: 33524645 00000032 00000001 00000002
```

**Interpretação 1:** magic bate **exatamente** (4 primeiros words idênticos ao esperado) — struct real confirmada. `structVersion`=1.1. `signatureType`=0 (não assinado). `signatureLocation`=`0x32b4` (~12,7 KB — dentro da região reservada). **`app.type`=`0x00000040`=bit6=`APPLICATION_TYPE_BOOTLOADER`** — este é o **Gecko Bootloader**, não a aplicação Zigbee. `app.version`=`0x01080000`→ decodifica como versão "1.8.0.0" do bootloader. Byte final ("decryptKey", reaproveitado) contém ASCII `"EFR32"` seguido de inteiros pequenos — mais uma confirmação independente da plataforma, direto do firmware.

**Status:** CONFIRMED — região `0x00000000`–`~0x4000` é o **Gecko Bootloader** (Silicon Labs), versão 1.8.

**Comando 2** (pesquisa oficial: no EFR32xG21, o bootloader reserva 16 KB e a aplicação começa em `0x00004000` — confirmado via `docs.silabs.com`, seção "Memory Space For Bootloading"):

```tcl
mdw 0x00004000 16
```

**Resultado 2:**

```text
0x00004000: 200070b0 0003267d 0000da29 000324c5 010a0aa7 00004200 ac0f1804 01a767a0
0x00004020: 00000000 00000000 00000000 00000000 00000000 00032c6c 00000000 00000000
```

**Interpretação 2:** Stack Pointer (`0x200070b0`) e Reset Handler (`0x0003267d`) plausíveis para uma aplicação bem maior que o bootloader. Word 13 (offset `0x34`, endereço `0x00004034`) = `0x00032c6c` — ponteiro para a `ApplicationProperties_t` da aplicação principal. Alguns handlers de exceção (`MemManage`, `UsageFault`, e o vetor em `0x401c`) vieram com valores grandes não-óbvios como ponteiro de código Thumb — não investigado a fundo (baixa prioridade frente ao resultado principal).

**Comando 3** (struct da aplicação principal, com leitura estendida para capturar strings adjacentes):

```tcl
mdw 0x00032c6c 32
```

**Resultado 3:**

```text
0x00032c6c: fa79b713 b7dd25c9 e0cff3ad b814b6f1 00000101 00000000 ffffffff 00000001
0x00032c8c: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0x00032cac: 2e6d6f63 00000063 45584552 5f45534e 435f4148 535f4f4f 37366b74 4d5f3031
0x00032ccc: 35313247 372e315f 0000332e 5f584552 004f4f43 45584552 0045534e 31323131
```

**Interpretação 3:** magic bate exatamente de novo. `structVersion`=1.1, `signatureType`=0 (não assinado), `signatureLocation`=`0xFFFFFFFF` (não definido — consistente com "sem assinatura"). **`app.type`=`0x00000001`=bit0=`APPLICATION_TYPE_ZIGBEE`** ✅ — **esta é a aplicação Zigbee real**, confirmada pelo próprio firmware, não por inferência.

Decodificando os bytes finais (região "decryptKey" reaproveitada + dados adjacentes) como ASCII, aparecem strings legíveis separadas por bytes nulos:

```text
REXENSE_HA_COO_Stk6710_MG215_1.7.3
REX_COO
REXENSE
1121
```

Interpretação destas strings (INFERRED, alta confiança, mas não documentação oficial — são identificadores internos da Rexense, não têm especificação pública):
- `REXENSE` — fabricante do módulo, confirma a marcação física.
- `HA` — provavelmente "Home Automation".
- **`COO`** — muito provavelmente **"Coordinator"** (papel Zigbee). Se correto, **este firmware específico roda como Coordenador Zigbee**, não apenas NCP/Router genérico — diretamente relevante para o objetivo do projeto.
- `Stk6710` — provável referência à versão da stack EmberZNet/GSDK usada no build (formato sugestivo de "6.7.1.0" ou similar).
- `MG215` — referência à plataforma (EFR32MG21 + sufixo/variante "5", não totalmente esclarecido).
- `1.7.3` — versão do firmware da aplicação.
- `REX_COO` — forma curta do identificador (Rexense Coordinator).
- `1121` — não esclarecido (pode ser build/data code, ex.: nov/2021).

**Status:** CONFIRMED — aplicação principal é `APPLICATION_TYPE_ZIGBEE` (via `app.type`, direto do firmware). CONFIRMED — strings de identificação legíveis presentes no firmware, apontando para Rexense como origem. INFERRED (alta confiança) — papel de **Coordinator** Zigbee, versão de firmware 1.7.3, stack ~6.7.1.0. UNKNOWN — significado exato de `MG215` e `1121`.

**Próximo passo:** documentar isso com destaque no README (é a descoberta mais relevante para o objetivo do projeto até agora). Considerar buscar mais strings próximas (versão do EmberZNet, EUI-64 real usado pela aplicação — pode não ser o de `DEVINFO`), e eventualmente montar um mapa de memória: bootloader (`0x0`–`0x4000`), aplicação (`0x4000`–`?`).

---

## 2026-09-05 — Sondagem empírica do tamanho real da flash

**Objetivo:** como a via `DEVINFO_MSIZE` falhou (deu valor fisicamente impossível), tentar determinar o tamanho real da flash empiricamente, testando a última palavra de cada tamanho de flash conhecido do EFR32MG21 (512/768/1024 KB — SKUs reais: `...F512...`, `...F768...`, `...F1024...`).

**Pesquisa prévia:** NVM3 (armazenamento de dados de rede Zigbee, chaves etc.) fica por padrão no **final da flash principal**, com tamanho padrão de **40 KB** em dispositivos Series 2 (`docs.silabs.com`, "Memory Layout", Gecko Platform).

**Comando:**

```tcl
mdw 0x0007FFFC 1
mdw 0x000BFFFC 1
mdw 0x000FFFFC 1
```

Classificação: READ-ONLY / SAFE (mesmo `mdw`, via AP0). Uma leitura fora da flash real resultaria, na pior hipótese, em erro de barramento reportado pelo OpenOCD — não é uma operação destrutiva.

**Resultado:**

```text
0x0007fffc: ffffffff   (limite de 512 KB)
0x000bfffc: 00000000   (limite de 768 KB)
0x000ffffc: 00000000   (limite de 1024 KB)
```

**Interpretação:** nenhuma das três leituras resultou em erro de barramento (todas "tiveram sucesso" do ponto de vista do OpenOCD), então isso não é 100% conclusivo por si só. Porém, o padrão é sugestivo: `0xFFFFFFFF` (padrão de flash apagada/não escrita, típico de NOR flash real) no limite de 512 KB, contra `0x00000000` (padrão comum de barramento não mapeado em muitos SoCs ARM) nos limites de 768 KB e 1024 KB. Isso é consistente com **flash real de 512 KB** (SKU tipo `EFR32MG21xxxF512xxx`), com a região de NVM3 (~40 KB, padrão) ocupando o final dessa flash, ainda majoritariamente apagada/vazia (`0xFFFFFFFF`) na posição testada.

**Status:** INFERRED (não CONFIRMED — não é um teste definitivo, pois nenhuma leitura gerou erro explícito). Flash provavelmente de 512 KB. Recomenda-se não tratar como fato até confirmação adicional (ex.: encontrar o cabeçalho real de uma página NVM3 formatada, ou uma leitura que gere erro de barramento claro acima de 512 KB).

**Próximo passo:** procurar o início da região NVM3 (se flash=512KB, região padrão seria aproximadamente os últimos 40 KB, por volta de `0x00076000`–`0x00080000`) e buscar por um cabeçalho de página NVM3 real (formato terá padrão reconhecível), em vez de assumir o tamanho da flash como fato.
