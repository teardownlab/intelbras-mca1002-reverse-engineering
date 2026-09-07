# Identificação de hardware — Intelbras MCA 1002

Este documento consolida a identificação dos componentes principais da PCB do MCA 1002, com status **CONFIRMED**, **INFERRED** ou **UNKNOWN** para cada afirmação.

## Correção de identificação (2026-09-05)

Uma versão anterior deste projeto identificava o módulo secundário (não-Zigbee) como **Tuya CB3S**, baseado em SoC **Beken BK7231N**.

**Essa identificação estava ERRADA e é considerada retirada.** Nenhuma conclusão anterior baseada nela (protocolo, compatibilidade com OpenBeken, arquitetura assumida) deve ser tratada como válida.

A identificação correta, por serigrafia do módulo, é:

```text
RE761-N4P
```

Status: **CONFIRMED** apenas a marcação serigráfica `RE761-N4P`. O chipset interno, fabricante do SoC, função exata na placa (Wi-Fi? outro rádio? bridge?) e sua interface com o REX3B21 são **UNKNOWN**.

Não presumir Tuya/Beken/CB3S/BK7231N para o RE761-N4P sem evidência nova e específica (ex.: FCC filing, datasheet, dump de firmware analisado).

## Módulo Zigbee — Rexense REX3B21S

Status: **CONFIRMED**

Identificação visual (serigrafia):

```text
REXENSE
REX3B21S
U-V5B1
```

| Item | Valor | Status |
|---|---|---|
| Fabricante do módulo | Rexense | CONFIRMED |
| Modelo do módulo | REX3B21S | CONFIRMED |
| SoC | Silicon Labs EFR32MG21 | CONFIRMED (via SWD, ver [`swd.md`](swd.md)) |
| CPU core | ARM Cortex-M33 | CONFIRMED (CPUID lido via SWD = `0x410FD213`, r0p3) |
| Protocolo rádio suportado pelo SoC | Zigbee 3.0 (capacidade do EFR32MG21) | INFERRED (capacidade do chip; não confirma o firmware atualmente gravado) |
| Firmware atualmente gravado no EFR32MG21 | — | UNKNOWN (ainda não lido/analisado) |

Pinout do módulo relevante para debug (correlacionado com J3 por continuidade — ver [`measurements.md`](measurements.md)):

| Pino REX3B21 | Função | Status |
|---:|---|---|
| 5 | 3,3 V / VCC / VTref | CONFIRMED |
| 7 | GND | CONFIRMED |
| 19 | RESET | CONFIRMED |
| 17 | PA2 / SWDIO | CONFIRMED |
| 15 | PA1 / SWCLK | CONFIRMED |
| 3 | PA5 / TXD | INFERRED (da documentação do módulo; não testado por continuidade ainda) |
| 4 | PA6 / RXD | INFERRED (da documentação do módulo; não testado por continuidade ainda) |
| 18 | GND | INFERRED (da documentação do módulo; não testado por continuidade ainda) |

## Módulo secundário — RE761-N4P

Status do módulo: **CONFIRMED** (identificação visual)
Status da função/firmware: **CONFIRMED** (via captura passiva de UART, 2026-09-07 — ver [`experiments.md`](experiments.md))
Status do SoC/fabricante do silício: **CONFIRMED** (2026-09-07, marcação física lida diretamente no chip após remoção da blindagem metálica).

| Item | Valor | Status |
|---|---|---|
| Marcação serigráfica (módulo) | RE761-N4P (SN: NMPB00300310) | CONFIRMED |
| Fabricante do silício | **Shenzhen iComm Semiconductor Co., Ltd.** (南方硅谷半导体) | CONFIRMED (marcação física + datasheet oficial) |
| SoC interno (part number real) | **SV32WB06** — marcação física lida no chip: `SV32WB06 / TAC2411 / 0PW07` (após remoção da blindagem) | **CONFIRMED** — bate exatamente com datasheet público da iComm Semi (`SV32WB0xx Datasheet V1.1`) |
| SoC interno (codinome de boot, obsoleto) | ~~`TurismoE 6020B`~~ — string de boot real, mas não é o nome do fabricante nem do part number; hipótese de "Bouffalo Lab BL808" baseada só em fingerprint técnico (clock 480MHz + combo WiFi/BT) **estava ERRADA** — coincidência de arquitetura, não o mesmo chip | REFUTED em 2026-09-07 — ver correção abaixo |
| Função na placa | Controlador Wi-Fi / gateway principal (roda a stack de aplicação do hub, fala com REX3B21 via UART e com a nuvem via Wi-Fi) | CONFIRMED |
| Firmware | Baseado em SDK/framework **Dahua/IMOU** (símbolos `IMOU_sysEnvRead`, `IMOU_sceneLinkage`, `ZigbeeAdapt_Rex.c`) | CONFIRMED |
| Build identificado | `Project Name: GateWay`, `PackName: General_GateWay_IOT-ZG2-IB_SV32WB0X_V2.4.628243.R.26014`, `Build File: product.gw-ZG2-IB.svr32wbx.cfg`, `Git Commit: 7f9a6ef6f` | CONFIRMED (string de boot, texto claro) |
| Backend de nuvem (DRS) | `iotaccess.easy4ipcloud.com` — plataforma IoT "Easy4ip" da Dahua Technology | CONFIRMED (string de boot, texto claro) |
| MAC deste chip | `98-2A-0A-D2-CC-7B` (e um segundo, `...CC-7C`, provavelmente a interface BT/BLE do mesmo SoC) | CONFIRMED |
| Zigbee SDK usado internamente para falar com o REX3B21 | `zigbee max_num=800`, versão `1.2.3-0.4a3e46e`, arquivo `ZigbeeAdapt_Rex.c` | CONFIRMED |
| Armazenamento (flash) próprio deste chip | ~19,6 MB total, ~1,4 MB usado (fora da partição de OTA); contém `/ota.bin` (13,3 MB), `/Back_zigbeelib`, `/Back_wifikey`, `/Back_wifissid`, `/Back_gateway`, etc. — sistema de arquivos próprio, separado da flash de 512 KiB do EFR32 | CONFIRMED (listagem de boot) |
| Interface com REX3B21 | UART (não SPI/I2C) | CONFIRMED |

**Conclusão importante para o objetivo do projeto:** o MCA 1002 é, por baixo, um hub Dahua/IMOU rebrandeado pela Intelbras (Mibo). A nuvem "Intelbras/Mibo" é provavelmente white-label da plataforma Easy4ip da Dahua. Isso não muda o plano de usar o EFR32/REX3B21 como coordenador Zigbee local — mas explica a origem do protocolo/arquitetura original e é relevante caso se queira, no futuro, investigar o RE761-N4P mais a fundo (ex.: extrair `/ota.bin` via alguma interface de atualização, já que ele contém o firmware Dahua/IMOU completo).

**Achado de segurança (não relacionado ao silício, mas relevante):** o RE761-N4P grava a senha de Wi-Fi doméstico do usuário em texto claro em variáveis internas expostas no próprio log de boot (`Connect AP: ssid=...` / `Connect AP: Password=...`). Não foi investigado se o arquivo `/Back_wifikey` na flash também guarda em claro, mas é provável. **Nunca commitar capturas brutas de UART deste chip no repositório público** — ver aviso em [`experiments.md`](experiments.md).

Interface física confirmada (numeração do header **J1: pino 1 a 4, contado de fora da placa para dentro**):

| Header/pino | Função | Status |
|---|---|---|
| J1 pino 2 | TX do RE761-N4P (saída, 115200 8N1) | CONFIRMED (2026-09-07, captura de boot completa via UART) |
| J1 pino 1 | RX do RE761-N4P (entrada — console de comandos aceita entrada aqui) | CONFIRMED (2026-09-07, ver seção de console abaixo) |
| J1 (pinos 3 e 4) | não testados | UNKNOWN |

### Console de comandos via UART (CONFIRMED, 2026-09-07)

Com J1-1 (RX) + J1-2 (TX) + GND dedicado ligados a um adaptador USB-TTL, existe um **console de comandos interativo** na mesma UART do log de boot, 115200 8N1. Comandos testados sem necessidade de senha (prompt `?>`):

| Comando | Resultado |
|---|---|
| `?` | Lista comandos: `meminfo`, `sysinfo`, `cmd_log`, `cmd_tag`, `cmd_show` |
| `meminfo` | `total SRAM: 512K` + tabela de uso ILM/DLM/Bus, `psram not exist` |
| `sysinfo` | `mcu clk 480000000`, `xtal clk 26000000`, `bus clk 160000000`, `xip mode 2`, lista de tasks RTOS (`isr`, `cli`, `IDLE`, `Tmr Svc`, `Radio_Receive_T`, `Radio_Tx_Task`, `tcpip_task`, `WdtIdle`, `comTask`, `VoicePlay`, `msgDealPool`, `ZigbeeComm`, `SmartLink`, `sta connect tas`, `scan task`) |
| `cmd_log` / `cmd_tag` / `cmd_show` | Comandos de teste/debug de log, funcionais mas de baixo valor de identificação |

Qualquer outro comando (`help`, `ps`, `version`, `AT`, tentativas de senha `admin`/`12345678`/`888888`/`password`/`1234`/`0000`/número de série) aparenta ser silenciosamente ignorado. Um bloco `[password]:password is wrong / Enter the password,Please` aparece intercalado no log, mas com timing inconsistente com os comandos enviados — hipótese: é ruído de um subsistema não relacionado (ex.: provisionamento BLE) escrevendo na mesma UART compartilhada, não uma resposta real às nossas tentativas. Não confirmado.

### Identificação definitiva do SoC: iComm Semiconductor SV32WB06 (CONFIRMED, 2026-09-07)

A blindagem metálica sobre o chip principal do RE761-N4P foi removida, revelando a marcação física direta: **`SV32WB06 / TAC2411 / 0PW07`**. Busca web confirmou: **SV32WB06** é um SoC real, documentado publicamente pela **Shenzhen iComm Semiconductor Co., Ltd.** — datasheet oficial: `SV32WB0xx Datasheet V1.1` (https://www.icomm-semi.com/Uploads/Temp/files/2022-01-21/SV32WB0xx%20Datasheet%20V1.1.pdf).

**Isso invalida a hipótese anterior de Bouffalo Lab BL808** — a semelhança (clock alto, combo WiFi+BT) era coincidência de arquitetura entre fornecedores diferentes, não o mesmo chip. Toda referência anterior a BL808/GPIO39/PU_CHIP neste documento e nos demais (`experiments.md`, `references.md`) deve ser tratada como **obsoleta/refutada**.

**Especificações confirmadas (batem com nossos achados via `sysinfo`/`meminfo`):**
- WiFi 802.11 b/g/n (single spatial stream) + Bluetooth 5.0
- Encapsulamento: **QFN60** (confirmado visualmente — contagem de pinos bate com o datasheet)
- 128KB ROM + até 512KB SRAM — bate com `total SRAM: 512K` do `meminfo`
- Flash integrado no encapsulamento (até 32Mb) — explica por que não achamos um chip de flash externo separado na PCB

**Pinout QFN60 confirmado no datasheet oficial (Tabela 20/Figura 14, específico pra variante SV32WB06):**

| Sinal | Pino físico (QFN60) | Função |
|---|---|---|
| **GPIO13** | **Pino 22** | **Strap de boot**: nível 0 = boot normal da flash (padrão); **nível 1 = modo IAP** (gravação/programação) |
| **LDO_EN** | **Pino 21** | **Reset completo do chip** — nível baixo por ≥500µs reseta tudo; a nota de fábrica diz "após de-assert, SV32WB0xx fica em modo OFF aguardando comunicação do host" |
| GPIO00 | Pino 8 | UART Rx de gravação (usado em modo IAP) |
| GPIO01 | Pino 9 | UART Tx de gravação (usado em modo IAP) |

Nota da Tabela 23 do datasheet: *"Use GPIO00/GPIO01 as UART Rx/Tx to program the flash for SV32WB01x/SV32WB06"* — confirma explicitamente essa combinação de pinos para a nossa variante exata.

**Vantagem prática:** os pinos 21 (LDO_EN) e 22 (GPIO13) ficam **fisicamente adjacentes** no encapsulamento — convenientes para soldar os dois de uma vez.

**Bloqueio atual:** a blindagem já foi removida (acesso físico direto ao chip existe agora), mas ainda falta **localizar o pino 1 fisicamente na foto/chip real** para poder contar corretamente até os pinos 21/22 (e 8/9). Isso exige uma foto bem nítida e ampliada do marcador de pino 1 (ponto/chanfro num dos cantos do QFN) com boa iluminação.

**Status:** CONFIRMED — identidade do SoC (SV32WB06, iComm Semiconductor) e localização exata dos pinos de boot/reset no datasheet oficial. UNKNOWN (próximo passo) — correlação entre a numeração do datasheet e a orientação física real do chip soldado na placa.

Próximos passos de identificação sugeridos (não executados ainda):
- pesquisar `RE761-N4P` em bases de FCC ID / certificação, caso exista marcação de certificação próxima ao módulo na PCB;
- localizar fisicamente o chip de flash SPI externo (~19,6 MB) na PCB — provavelmente um SOIC-8 próximo ao RE761-N4P, ainda não mapeado;
- opcionalmente, tentar extrair `/ota.bin` (13,3 MB) por algum canal de atualização/depuração, para análise offline do firmware Dahua/IMOU completo.

## Interface J3 (resumo)

Ver detalhes completos e histórico de medição em [`measurements.md`](measurements.md).

| J3 | Função | Status |
|---|---|---|
| J3-1 | 3,3 V / VCC / VTref | CONFIRMED |
| J3-2 | GND | CONFIRMED |
| J3-3 | RESET | CONFIRMED |
| J3-4 | SWDIO | CONFIRMED |
| J3-5 | SWCLK | CONFIRMED |
| J3-6 | desconhecido (~1,2 V energizado) | UNKNOWN |

**Regra de segurança:** J3-6 não deve ser conectado a nada até ser identificado.

## Acesso de debug — SWD

Status: **CONFIRMED**

J3 é o header SWD do REX3B21/EFR32MG21. Conexão SWD estabelecida com sucesso via CMSIS-DAP (Raspberry Pi Pico com firmware Debug Probe) e OpenOCD. Detalhes completos, comandos e saídas em [`swd.md`](swd.md).
