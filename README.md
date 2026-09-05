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
  - detalhes completos em [`docs/swd.md`](docs/swd.md)

### Ainda não confirmado

- identidade/chipset do módulo **RE761-N4P**;
- interface entre RE761-N4P e REX3B21;
- se essa interface é UART, e se sim baud rate/protocolo;
- função de **J3-6**;
- firmware atual do EFR32 (ainda não lido/analisado);
- estado de debug/security/read protection do EFR32 (a verificar sem alterar);
- mapa completo de memória do EFR32MG21 (flash size, RAM size, DEVINFO, NVM3, bootloader, EUI-64, manufacturing tokens, calibration data);
- compatibilidade do rádio existente com EZSP/NCP;
- uso direto por ZHA/Zigbee2MQTT;
- viabilidade prática de qualquer firmware alternativo para o RE761-N4P (opção A do plano de reaproveitamento está em reavaliação).

## Próximo passo

Com acesso SWD confirmado e leitura de memória funcional via AHB-AP, o próximo objetivo é identificar completamente a variante do EFR32MG21 (part number, revisão, flash size, RAM size) e verificar o estado de debug/security **sem alterá-lo**, como base para um plano de backup completo do firmware original antes de qualquer modificação.

Nenhuma operação destrutiva (erase, write, unlock, recover) será feita nesta etapa.

## Referências

Ver [`docs/references.md`](docs/references.md) para a lista completa de datasheets, reference manuals e documentação oficial usada nesta investigação.

---

Este documento será atualizado continuamente conforme novas medições, pinouts, logs UART, dumps de firmware e testes forem realizados.