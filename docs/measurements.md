# Medições elétricas — Intelbras MCA 1002

Este arquivo registra as medições realizadas durante a investigação da PCB do MCA 1002.

## Header J3

Convenção de numeração confirmada pela serigrafia da PCB:

- **J3-1** = pad quadrado, na extremidade direita;
- sequência até **J3-6**, na extremidade esquerda.

### Continuidade com GND — placa desligada

Referência usada: blindagem metálica externa do conector micro-USB.

| Pad | Continuidade com GND |
|---|---|
| J3-1 | Não |
| **J3-2** | **Sim** |
| J3-3 | Não |
| J3-4 | Não |
| J3-5 | Não |
| J3-6 | Não |

Conclusão:

```text
J3-2 = GND [confirmado]
```

### Tensão DC — placa energizada

Referência: ponta preta em J3-2 (GND), ponta vermelha nos demais pads.

| Pad | Tensão medida |
|---|---:|
| J3-1 | 3,355 V |
| J3-2 | 0 V / GND |
| J3-3 | 3,295 V |
| J3-4 | 3,337 V |
| J3-5 | 3,340 V |
| J3-6 | 1,2 V |

## Continuidade direta entre J3 e o módulo REX3B21

Com a placa totalmente desligada, foram testadas as conexões diretas entre J3 e os pinos relevantes do módulo Zigbee REX3B21.

Resultados confirmados:

| J3 | Pino REX3B21 | Função |
|---|---:|---|
| **J3-1** | **5** | **3,3 V / VCC / VTref** |
| **J3-2** | **7** | **GND** |
| **J3-3** | **19** | **RESET** |
| **J3-4** | **17** | **PA2 / SWDIO** |
| **J3-5** | **15** | **PA1 / SWCLK** |
| J3-6 | — | Não identificado |

Pinout funcional confirmado até agora:

```text
J3-1 = 3V3 / VTref
J3-2 = GND
J3-3 = RESET
J3-4 = SWDIO
J3-5 = SWCLK
J3-6 = desconhecido (~1,2 V quando energizado)
```

## Conclusão importante

O header **J3 é efetivamente um header de debug/programação SWD do módulo Zigbee REX3B21/EFR32**.

Isso é uma descoberta importante porque fornece acesso direto ao microcontrolador Zigbee sem depender do módulo Wi-Fi CB3S.

Em princípio, esse acesso permite usar um debugger compatível com SWD (por exemplo J-Link ou CMSIS-DAP) para:

- identificar o MCU;
- consultar o estado de proteção/debug;
- eventualmente ler a memória flash se a proteção permitir;
- fazer backup do firmware original;
- analisar o firmware existente;
- e, em etapa posterior e somente com backup/entendimento suficiente, gravar outro firmware compatível.

## Sobre J3-6

J3-6 não apresentou continuidade com os pinos 5, 7, 15, 17 ou 19 testados. Quando a placa está energizada, mede aproximadamente **1,2 V** em relação ao GND.

Ainda pode ser:

- SWO;
- UART TX/RX;
- outro GPIO de produção/teste;
- sinal intermediário com resistor/condicionamento;
- linha com atividade digital cuja tensão média aparece como ~1,2 V no multímetro.

Ainda não identificado.

## Relação com o REX3B21

A documentação do REX3B21 indica os seguintes sinais relevantes no módulo:

- pino 3: PA5 / TXD
- pino 4: PA6 / RXD
- pino 5: 3,3 V / VCC
- pino 7 e 18: GND
- pino 15: PA1 / SWCLK
- pino 17: PA2 / SWDIO
- pino 19: RESET

As medições de continuidade confirmam que J3 expõe diretamente VCC, GND, RESET, SWDIO e SWCLK.

## Próximos testes recomendados

1. **Não gravar firmware ainda.**
2. Usar um debugger SWD compatível com lógica de 3,3 V.
3. Conectar apenas:
   - VTref -> J3-1
   - GND -> J3-2
   - RESET -> J3-3
   - SWDIO -> J3-4
   - SWCLK -> J3-5
4. Não alimentar a placa pelo debugger; alimentar o MCA pela própria entrada USB e usar J3-1 apenas como referência de tensão, salvo se houver razão técnica específica e verificada para fazer diferente.
5. Tentar apenas uma operação de identificação/conexão ao MCU inicialmente, sem erase, write ou unlock.
6. Se a conexão SWD funcionar, registrar:
   - identificação exata do chip;
   - estado de lock/debug protection;
   - tamanho da flash;
   - possibilidade de leitura sem apagar.
7. Somente depois considerar dump/backup e qualquer alteração de firmware.
