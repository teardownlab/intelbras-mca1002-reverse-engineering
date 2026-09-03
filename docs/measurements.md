# Medições elétricas — Intelbras MCA 1002

Este arquivo registra as medições realizadas durante a investigação da PCB do MCA 1002.

## Header J3

Convenção de numeração:

- J3-1 = pad mais próximo da inscrição `J3`
- sequência até J3-6

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

## Interpretação provisória

J3-1, J3-3, J3-4 e J3-5 estão próximos da tensão lógica de 3,3 V. Isso pode corresponder a uma combinação de:

- VCC/VTREF;
- linhas digitais em estado lógico alto;
- SWDIO/SWCLK;
- RESET em idle alto;
- UART TX/RX em idle alto.

**Não é possível distinguir essas funções apenas pela leitura DC.**

J3-6 em aproximadamente 1,2 V é atípico para uma linha digital estática de 3,3 V. Pode ser uma linha com atividade rápida cuja média aparece como ~1,2 V no multímetro, um sinal com pull-up/pull-down, ou outra função. Ainda não há identificação.

## Relação com o REX3B21

A documentação do REX3B21 indica os seguintes sinais relevantes no módulo:

- pino 3: PA5 / TXD
- pino 4: PA6 / RXD
- pino 5: 3,3 V / VCC
- pino 7 e 18: GND
- pino 15: PA1 / SWCLK
- pino 17: PA2 / SWDIO
- pino 19: RESET

Como J3 possui 6 pads e está próximo ao módulo Zigbee, uma hipótese importante é que seja um header de produção/debug contendo VCC, GND, SWDIO, SWCLK, RESET e possivelmente UART/SWO ou outro sinal. **Essa hipótese ainda não foi confirmada por continuidade de trilhas.**

## Próximos testes recomendados

1. Não aplicar tensão externa em nenhum pad.
2. Com a placa desligada, testar continuidade entre J3 e os pinos relevantes do REX3B21 (VCC, TXD, RXD, SWCLK, SWDIO e RESET).
3. Se a identificação por continuidade não for possível, usar analisador lógico ou USB-UART apenas em modo de escuta para verificar atividade nos pads.
4. Somente após identificar o pinout considerar conexão de debugger ou reflashing.
