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
