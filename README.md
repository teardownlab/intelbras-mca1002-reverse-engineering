# Intelbras MCA 1002 – Reverse Engineering para Home Assistant

Este repositório documenta a investigação do **Intelbras MCA 1002** para entender sua arquitetura interna e avaliar se o hardware pode ser reaproveitado como gateway/coordenador Zigbee local para **Home Assistant**, preferencialmente com **ZHA** ou **Zigbee2MQTT**.

> **Estado:** investigação em andamento. Nenhum firmware foi substituído até o momento.

## Objetivo

```text
Home Assistant / Debian
        |
        | LAN / Wi-Fi
        v
     MCA 1002
        |
        | Zigbee
        v
Dispositivos Zigbee
```

O objetivo ideal é operação totalmente local e sem dependência de cloud.

## Funcionamento original

O MCA 1002 atua como hub Zigbee da linha Intelbras/Mibo, conectando dispositivos Zigbee ao ecossistema do fabricante por Wi-Fi 2,4 GHz. Possui também sirene integrada e suporte anunciado para até 32 dispositivos Zigbee.

Por padrão, ele **não é exposto ao Home Assistant como adaptador Zigbee USB**, portanto não pode ser selecionado diretamente por ZHA ou Zigbee2MQTT sem modificação ou integração adicional.

## Descobertas de hardware

> **CORREÇÃO (2026-09-05):** uma identificação anterior deste módulo como **Tuya CB3S / Beken BK7231N** estava **ERRADA** e foi removida. A serigrafia do módulo é **RE761-N4P**. Não presumir CB3S/BK7231N em nenhuma decisão futura. Ver [`docs/hardware.md`](docs/hardware.md) para o registro completo da correção.

### Módulo secundário: RE761-N4P (chipset UNKNOWN)

Foi identificado visualmente, por serigrafia, o módulo:

```text
RE761-N4P
```

Isso é tudo que está **CONFIRMED** sobre esse módulo neste momento. O chipset/SoC interno, sua função exata (Wi-Fi? bridge? outro?) e sua interface com o REX3B21 permanecem **UNKNOWN** e precisam ser investigados do zero — sem reaproveitar hipóteses da identificação anterior (CB3S/BK7231N), que estava incorreta.

### Zigbee: Rexense REX3B21S / REX3B21

Foi identificado visualmente o módulo:

```text
REXENSE
REX3B21S
U-V5B1
```

A família REX3B21 utiliza plataforma **Silicon Labs EFR32**, com indicação de **EFR32MG21**. O SoC suporta Zigbee 3.0 e interfaces como UART/USART, SPI, I²C e GPIO.

## Arquitetura provável

```text
             Intelbras MCA 1002

        Wi-Fi 2,4 GHz
              |
      +-------v--------+
      | RE761-N4P      |
      | chipset UNKNOWN|
      +-------+--------+
              |
              | interface com REX3B21
              | HIPÓTESE — não confirmado
              |
      +-------v--------+
      | Rexense        |
      | REX3B21(S)     |
      | EFR32MG21      |
      +-------+--------+
              |
            Zigbee
              |
        dispositivos
```

O próximo objetivo é confirmar fisicamente a interface entre os dois módulos.

## Test points e headers

A PCB possui diversos pontos de teste e headers não populados. Um conjunto particularmente interessante é **J3**, com 6 pads.

Para esta investigação adotamos:

- **J3-1** = pad mais próximo da inscrição `J3`;
- sequência até **J3-6**.

### Continuidade de J3 para GND

Com a placa **totalmente desligada**, cada pad foi testado contra a blindagem metálica externa do micro-USB.

| Pad | Continuidade com GND |
|---|---|
| J3-1 | Não |
| **J3-2** | **Sim** |
| J3-3 | Não |
| J3-4 | Não |
| J3-5 | Não |
| J3-6 | Não |

Resultado confirmado até agora:

```text
J3-2 = GND
```

## Possibilidades de reaproveitamento

### A. Firmware alternativo no RE761-N4P

> **INVALIDADO/EM REAVALIAÇÃO:** esta opção era descrita anteriormente como "OpenBeken no CB3S", partindo da identificação incorreta do módulo como Tuya CB3S/BK7231N. Essa premissa caiu. Não sabemos ainda se o RE761-N4P é sequer um SoC Wi-Fi, nem se existe algum firmware alternativo compatível com ele. Esta opção fica em aberto até o chipset ser identificado.

```text
Home Assistant
      |
     MQTT (?)
      |
    Wi-Fi (?)
      |
RE761-N4P (chipset UNKNOWN)
      |
 comunicação com REX3B21 (?)
      |
  REX3B21
      |
    Zigbee
```

Ainda não sabemos o que é o RE761-N4P, qual protocolo ele usa com o REX3B21, nem se existe algum firmware alternativo aplicável.

### B. Acesso direto ao EFR32

Outra possibilidade é acessar diretamente o REX3B21. Se o EFR32 estiver executando firmware NCP/EZSP compatível — ou puder receber firmware adequado — existe potencial de utilização com **ZHA** ou **Zigbee2MQTT**.

Ainda não confirmado.

### C. Reflash do EFR32

Pode ser possível gravar firmware diferente diretamente no EFR32 caso sejam identificados os pads de programação/debug e não existam proteções impeditivas. Essa alternativa só será considerada depois do mapeamento e, se possível, backup do firmware original.

## Plano de investigação

1. **Não gravar firmware ainda.**
2. Mapear GND nos headers/test-points.
3. Identificar 3,3 V.
4. Identificar possíveis TX/RX.
5. Capturar comunicação UART em modo **somente leitura**.
6. Observar o boot do MCA 1002.
7. Observar comunicação durante pareamento/uso Zigbee.
8. Determinar protocolo RE761-N4P ↔ REX3B21.
9. Só então escolher entre firmware alternativo no RE761-N4P, bridge serial, acesso direto ao REX3B21 ou reflash.

## Segurança elétrica

**Não aplicar 5 V em GPIO/UART/test-points.**

Ao usar USB-UART futuramente:

- lógica de **3,3 V**;
- compartilhar apenas GND inicialmente;
- para captura passiva, conectar apenas RX do adaptador à linha observada;
- evitar conectar VCC do USB-UART sem necessidade;
- confirmar tensões antes de conectar qualquer adaptador.

## Estado atual

### Confirmado

- módulo Zigbee **Rexense REX3B21S**, plataforma **Silicon Labs EFR32MG21**, CPU **ARM Cortex-M33**;
- módulo secundário identificado visualmente como **RE761-N4P** — chipset/função ainda **UNKNOWN** (substitui a identificação anterior incorreta de Tuya CB3S/BK7231N);
- vários pads de teste/debug na PCB; J3 possui 6 pads;
- pinout completo de **J3** (ver [`docs/measurements.md`](docs/measurements.md)):
  - J3-1 = 3,3 V / VCC / VTref
  - J3-2 = GND
  - J3-3 = RESET
  - J3-4 = SWDIO
  - J3-5 = SWCLK
  - J3-6 = UNKNOWN (~1,2 V energizado, sem continuidade com os demais pinos testados)
- **J3 é o header SWD do REX3B21/EFR32MG21**, confirmado por conexão real via CMSIS-DAP (Raspberry Pi Pico com firmware Debug Probe) + OpenOCD 0.12.0:
  - SWD DPIDR = `0x6BA02477`
  - OpenOCD: `Cortex-M33 r0p3 processor detected`, `Examination succeed`
  - CPUID (leitura via AHB-AP, `mdw 0xE000ED00 1`) = `0x410FD213`, consistente com Cortex-M33
  - leitura de memória via AHB-AP **funcional** (somente leitura testada até agora)
  - AP0 IDR = `0x84770001` → MEM-AP tipo AHB-AP, JEP106 = ARM Limited (`efr32.dap apid 0`)
  - AP1 IDR = `0x54770002` → MEM-AP tipo APB-AP; confirmado como a **DCI (Debug Challenge Interface)** dos EFR32 Series 2 — o mecanismo de debug lock/unlock e **mass erase** (`Erase Device` = `0x430F`, permanentemente proibido neste projeto).
  - **Estado de debug/security lock confirmado** via `Read Lock Status` (`0x4311`, comando oficial de consulta, autorizado explicitamente antes de rodar): debug port em **"Standard Debug Unlock"** — sem lock ativo, secure debug desabilitado, `Erase Device` habilitado (mas não executado).
  - detalhes completos em [`docs/swd.md`](docs/swd.md)

### Ainda não confirmado

- identidade/chipset do módulo **RE761-N4P**;
- interface entre RE761-N4P e REX3B21;
- se essa interface é UART, e se sim baud rate/protocolo;
- função de **J3-6**;
- firmware atual do EFR32 (ainda não lido/analisado);
- mapa completo de memória do EFR32MG21 (flash size, RAM size, DEVINFO, NVM3, bootloader, EUI-64, manufacturing tokens, calibration data);
- compatibilidade do rádio existente com EZSP/NCP;
- uso direto por ZHA/Zigbee2MQTT;
- viabilidade prática de qualquer firmware alternativo para o RE761-N4P (opção A do plano de reaproveitamento está em reavaliação).

## Descoberta principal: firmware identificado

Usando a convenção oficial da Silicon Labs (struct `ApplicationProperties_t`, localizada via ponteiro gravado na palavra 13 do vetor de interrupções — código-fonte oficial conferido em `SiliconLabs/simplicity_sdk`), mapeamos a flash principal do EFR32MG21:

```text
0x00000000 - ~0x00004000   Gecko Bootloader (Silicon Labs), versão 1.8
0x00004000 - ?             Aplicação Zigbee (APPLICATION_TYPE_ZIGBEE confirmado no firmware)
```

A aplicação principal contém strings de identificação legíveis, lidas diretamente do firmware:

```text
REXENSE_HA_COO_Stk6710_MG215_1.7.3
REX_COO
REXENSE
1121
```

**`COO` muito provavelmente significa "Coordinator"** — ou seja, há indício forte (ainda não 100% confirmado) de que **este firmware específico já roda como Coordenador Zigbee**, não apenas como NCP/Router genérico. Isso é diretamente relevante para o objetivo do projeto (usar o MCA 1002 como coordenador Zigbee local). Versão do firmware: `1.7.3`. Detalhes completos, incluindo o registro byte a byte, em [`docs/experiments.md`](docs/experiments.md).

## Backup completo realizado

Backup completo da flash (512 KB, `0x00000000`–`0x0007FFFF`) feito via `dump_image` (leitura pura). Arquivo mantido **fora do Git e fora do OneDrive** (pode conter chaves de rede reais); apenas metadados (SHA-256, tamanho) documentados em [`docs/swd.md`](docs/swd.md).

## Achado estratégico: protocolo do host é proprietário (AT commands), não EZSP padrão

Análise de strings no backup revelou que este firmware **não fala EZSP puro** com o resto do hardware — a Rexense implementou uma **camada de comandos AT proprietária** sobre UART (`AT+FORM`, `AT+PERMITJOIN`, `AT+GETNETINFO`, `AT+SETMAC`, `AT+COOCFG`, entre ~29 comandos identificados) para controlar o coordenador Zigbee. Há indícios (strings `"ASH Frame Error"`/`"ASH Overrun Error"`) de que o framing ASH da Silicon Labs está presente, mas não está confirmado se os comandos AT trafegam dentro de frames ASH ou por uma via separada.

**Consequência prática para o objetivo do projeto (usar como coordenador Zigbee local no Home Assistant):** com o firmware atual, **ZHA e Zigbee2MQTT não conseguem se comunicar diretamente** com este rádio — ambos esperam EZSP binário padrão via `bellows`/`ember`, não este conjunto de comandos AT proprietário. Build do firmware identificado: `REXENSE_HA_COO_Stk6710_MG215_1.7.3`, de 28/07/2023.

Dois caminhos possíveis a partir daqui:

1. **Engenharia reversa do protocolo AT/ASH** e criação de uma ponte/driver customizado — preserva o firmware original, mas exige entender também como eventos/dados recebidos são reportados ao host (ainda não encontrado nas strings).
2. **Reflash do EFR32MG21 com firmware NCP/EZSP oficial** — mais direto para compatibilidade, mas contraria a prioridade atual do projeto de não substituir firmware, e só seria considerado depois de entendimento e backup completos.

Detalhes completos da análise em [`docs/experiments.md`](docs/experiments.md).

## Próximo passo

Decisão em aberto: seguir com engenharia reversa do protocolo AT/UART (próximo passo técnico seria captura passiva de tráfego UART real, usando os pinos PA5/TXD e PA6/RXD do REX3B21S, ainda não testados por continuidade) ou considerar reflash mais adiante. Também em aberto, com prioridade menor: confirmar tamanho real da flash (hoje INFERRED como 512 KB), localizar dados NVM3 reais, e identificar o módulo RE761-N4P.

Nenhuma operação destrutiva (erase, write, unlock, recover) será feita nesta etapa.

## Referências

Ver [`docs/references.md`](docs/references.md) para a lista completa de datasheets, reference manuals e documentação oficial usada nesta investigação.

---

Este documento será atualizado continuamente conforme novas medições, pinouts, logs UART, dumps de firmware e testes forem realizados.