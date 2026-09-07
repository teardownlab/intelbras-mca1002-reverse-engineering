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
Status do SoC/fabricante do silício: **candidato encontrado** (nome de codinome visto em log de boot), mas não resolvido a um fabricante/part number público — ver abaixo.

| Item | Valor | Status |
|---|---|---|
| Marcação serigráfica | RE761-N4P (SN: NMPB00300310) | CONFIRMED |
| Fabricante do silício | — | UNKNOWN |
| SoC interno (codinome de boot) | `TurismoE 6020B` (string `"< TurismoE 6020B SoC BT1M Rx DC calibration...done"`, confirmada byte-a-byte em captura de 2026-09-07) | CONFIRMED a string; UNKNOWN o mapeamento para fabricante/part number real — sem resultado em busca web (`TurismoE` não é nome público de nenhum fabricante conhecido; provavelmente codinome interno do fornecedor de silício pra Dahua/IMOU) |
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
| J1 (demais pinos) | não testados individualmente após a renumeração acima | UNKNOWN |
| J2 | suspeitos anteriores (pad 35/pad 8 do chip) não re-confirmados com a numeração corrigida | UNKNOWN — revisar |

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
