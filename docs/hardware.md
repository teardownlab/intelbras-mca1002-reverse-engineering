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

Status do módulo: **CONFIRMED** (identificação visual apenas)
Status do chipset/função: **UNKNOWN**

| Item | Valor | Status |
|---|---|---|
| Marcação serigráfica | RE761-N4P | CONFIRMED |
| Fabricante | — | UNKNOWN |
| SoC interno | — | UNKNOWN |
| Função na placa | — | UNKNOWN (hipótese: controlador Wi-Fi, não confirmado) |
| Interface com REX3B21 | — | UNKNOWN (hipótese: serial/UART, não confirmado) |

Próximos passos de identificação sugeridos (não executados ainda):
- pesquisar `RE761-N4P` em bases de FCC ID / certificação, caso exista marcação de certificação próxima ao módulo na PCB;
- fotografar o módulo em alta resolução para procurar outras marcações (datecode, FCC ID, SoC package markings);
- verificar se há pontos de teste/headers próprios do RE761-N4P ainda não mapeados.

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
