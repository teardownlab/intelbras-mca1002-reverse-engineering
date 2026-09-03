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

### Wi-Fi: Tuya CB3S

Foi identificado visualmente um módulo **Tuya CB3S**, baseado no SoC **Beken BK7231N**.

Isso fornece evidência física forte de tecnologia Tuya no lado Wi-Fi do MCA 1002. O BK7231N possui UART e existem firmwares alternativos para ele, incluindo **OpenBeken**.

Isso **não significa** que instalar OpenBeken automaticamente torne o MCA 1002 compatível com ZHA/Zigbee2MQTT. Ainda precisamos descobrir como o CB3S controla o rádio Zigbee.

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
      | Tuya CB3S      |
      | BK7231N        |
      +-------+--------+
              |
              | UART / outra interface
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

### A. OpenBeken no CB3S

O CB3S/BK7231N possui suporte conhecido pelo OpenBeken.

```text
Home Assistant
      |
     MQTT
      |
    Wi-Fi
      |
OpenBeken / CB3S
      |
 comunicação serial (?)
      |
  REX3B21
      |
    Zigbee
```

Ainda não sabemos o protocolo entre CB3S e REX3B21 nem se o OpenBeken pode controlar esse módulo Zigbee nessa implementação.

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
8. Determinar protocolo CB3S ↔ REX3B21.
9. Só então escolher entre OpenBeken, bridge serial, acesso direto ao REX3B21 ou reflash.

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

- módulo Wi-Fi **Tuya CB3S / BK7231N**;
- módulo Zigbee **Rexense REX3B21S / REX3B21**;
- plataforma Zigbee baseada em **Silicon Labs EFR32**, com indicação de EFR32MG21;
- vários pads de teste/debug na PCB;
- J3 possui 6 pads;
- **J3-2 é GND**, confirmado por continuidade.

### Ainda não confirmado

- interface entre CB3S e REX3B21;
- se a comunicação é UART;
- pinout completo de J3;
- baud rate/protocolo serial;
- firmware atual do EFR32;
- compatibilidade do rádio existente com EZSP/NCP;
- uso direto por ZHA/Zigbee2MQTT;
- viabilidade prática do OpenBeken para o MCA 1002 completo.

## Próximo passo

Com **J3-2 identificado como GND**, medir a tensão DC dos demais pads de J3 em relação a J3-2 para procurar **3,3 V** e possíveis linhas UART em idle alto.

Nenhuma gravação será feita nesta etapa.

## Referências a validar/adicionar

- Intelbras — MCA 1002;
- Tuya — CB3S;
- OpenBeken — BK7231N;
- Rexense — REX3B21;
- Silicon Labs — EFR32MG21.

Links e datasheets serão adicionados conforme forem validados durante a investigação.

---

Este documento será atualizado continuamente conforme novas medições, pinouts, logs UART, dumps de firmware e testes forem realizados.