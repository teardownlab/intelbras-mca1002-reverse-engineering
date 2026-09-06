# Plano de reflash — EFR32MG21 como coordenador Zigbee para Home Assistant

Este documento cobre a fase **pós-investigação** do projeto: a partir de 2026-09-05, com o firmware original identificado e um backup completo já feito (ver [`swd.md`](swd.md) e [`experiments.md`](experiments.md)), o responsável pelo projeto autorizou modificar o firmware do EFR32MG21, com o objetivo de:

> Ter um coordenador Zigbee funcional, acessível pelo servidor Home Assistant via USB (UART) ou Wi-Fi, permitindo sincronizar/parear os dispositivos Zigbee existentes.

## Por que não dá pra usar o firmware atual como está

O firmware original (Rexense, `REXENSE_HA_COO_Stk6710_MG215_1.7.3`) implementa uma camada de comandos **AT proprietária** sobre UART, não o protocolo **EZSP** padrão que `bellows` (ZHA) e o driver `ember` (Zigbee2MQTT) esperam. Ver [`experiments.md`](experiments.md) para a análise completa de strings que levou a essa conclusão.

## Plano

1. **Confirmar a variante exata do EFR32MG21** (flash size, package) — necessário para o build linkar corretamente. Estado atual: INFERRED como ~512 KB via sondagem empírica de endereço (ver `experiments.md`); ainda não 100% CONFIRMED.
2. **Montar o toolchain de build headless** (sem a IDE completa Simplicity Studio):
   - `slt-cli` (Silicon Labs Tool) — gerenciador de pacotes oficial para desenvolvimento via linha de comando.
   - Via `slt install`: `simplicity-sdk` (GSDK), `slc-cli`, `cmake`, `ninja`, `gcc-arm-none-eabi`, `commander`, `zap`.
   - Instalados também via winget como fallback/complemento: GNU Arm Embedded Toolchain, CMake, Ninja, Microsoft OpenJDK 17 (SLT também instala sua própria versão de Java internamente).
3. **Gerar um projeto NCP-UART-HW customizado** para o EFR32MG21, via `slc generate`, configurando explicitamente:
   - UART roteado para **PA5 (TX) / PA6 (RX)** — os mesmos pinos documentados para o REX3B21S (`pino 3 = PA5/TXD`, `pino 4 = PA6/RXD`), para que a fiação física não precise mudar.
   - Tamanho de flash/RAM compatível com a variante confirmada no passo 1.
4. **Compilar** o projeto (`ninja`/`make` via CMake gerado pelo `slc generate`).
5. **Confirmar fisicamente o acesso aos pinos PA5/PA6** na placa do MCA 1002 (continuidade, como fizemos com J3) — para saber onde conectar um adaptador USB-serial.
6. **Gravar via SWD/OpenOCD** — classificado como **POTENCIALMENTE DESTRUTIVO**. Requer confirmação explícita do responsável do projeto antes da execução, mesmo com a autorização geral já dada para modificar firmware, dado que é a ação mais consequente do projeto até agora. Backup completo (SHA-256 documentado em `swd.md`) já existe antes desta etapa.
7. **Testar** com Zigbee2MQTT ou ZHA via o adaptador USB-serial, confirmando que o coordenador aparece e permite parear dispositivos.

## Status de cada etapa

| Etapa | Status |
|---|---|
| 1. Confirmar variante exata do chip | **CONFIRMED definitivamente** — `EFR32MG21A020, rev 47`, flash 512 KiB, page size 8192 B, via `flash probe` do OpenOCD (driver oficial Silicon Labs) |
| 2. Toolchain headless (SLT) | **Concluído** — GSDK 2026.6.1, slc-cli 6.0.23, gcc-arm-none-eabi 14.2, LLVM embedded 21.1, CMake, Ninja, Commander, zap, tudo instalado via `slt install` |
| 3. Gerar projeto NCP-UART-HW customizado | **Concluído** — `iostream_usart` (instância `vcom`) configurado com USART0, TX=PA5, RX=PA6, sem controle de fluxo (2 fios) |
| 4. Compilar | **Concluído com sucesso** |
| 5. Confirmar pinos PA5/PA6 fisicamente | Ainda não feito — próximo passo antes de testar via USB |
| 6. Gravar via SWD | **CONCLUÍDO com sucesso em 2026-09-05** — ver detalhes abaixo |
| 7. Testar com Z2M/ZHA | Não iniciado — depende da etapa 5 |

## Gravação realizada (2026-09-05)

### Problema encontrado: driver de flash do OpenOCD 0.12.0 não suportava Series 2

A versão do OpenOCD instalada anteriormente (xpack 0.12.0) tem um driver `efm32` sem a tabela de famílias do Series 2 (`Unknown MCU family 128`). Encontrado um **script oficial e atual da própria Silicon Labs** no repositório oficial do OpenOCD (`tcl/target/silabs/series2.cfg` e `xg21.cfg`, copyright 2026 Silicon Laboratories Inc.) — esse script já implementa exatamente o protocolo DCI que reconstruímos manualmente antes (validação cruzada com nosso trabalho anterior). Só que ele exige uma versão mais nova do binário do OpenOCD (com suporte a `family 128` no driver `efm32`).

Baixado um build "nightly" oficial do próprio repositório `openocd-org/openocd` (release tag `latest`, commit `ddb476b`, compilado 2026-09-04) que já inclui esse suporte.

### Confirmação definitiva do chip (READ-ONLY, antes de gravar)

Comando: `flash probe 0` (via `target/silabs/xg21.cfg`, que já define `FLASHBASE=0x00000000` corretamente para xG21/xG22).

```text
Info : detected part: EFR32MG21 A020, rev 47
Info : flash size = 512 KiB
Info : flash page size = 8192 B
```

**Isso resolve definitivamente a ambiguidade da DEVINFO** que ficou em aberto durante a investigação: o chip é `EFR32MG21A020`, revisão de silício 47, com 512 KiB de flash — exatamente a variante usada para compilar nosso firmware. Também explica por que os campos legados de `DEVINFO_PART`/`MSIZE` liam valores estranhos: esse driver lê a família por um byte em endereço diferente (`0x0FE081FE`, compatível com todas as séries) do que os campos "modernos" de `DEVINFO_PART` que líamos antes (`0x0FE0E004`) — a Rexense aparentemente não popula esses campos modernos, mas o campo legado está correto.

### Gravação

Comando (**POTENCIALMENTE DESTRUTIVO** — autorizado explicitamente pelo responsável do projeto, com backup completo do firmware original já salvo e verificado por hash antes desta etapa):

```tcl
program zigbee_ncp_uart_hw.hex verify reset exit
```

Resultado:

```text
** Programming Started **
Info : detected part: EFR32MG21 A020, rev 47
Info : flash size = 512 KiB
Info : flash page size = 8192 B
Info : Padding image section 0 at 0x00004234 with 4 bytes
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
```

**A imagem compilada começa em `0x00004000`** (mesma região onde ficava a aplicação Zigbee original) — ou seja, a gravação **substituiu apenas a região de aplicação**, preservando intacto o Gecko Bootloader original da Rexense em `0x0000`–`0x4000`. Nenhuma outra região da flash foi tocada.

### Verificação pós-gravação (READ-ONLY)

```tcl
mdw 0x00004000 16
```

```text
0x00004000: 20001008 0000423b 0002956d 00004239 00004239 00029575 0002957d 00004239
0x00004020: 00004239 00004239 00004239 00004239 00029581 00033978 00004239 00004239
```

Vetor de interrupções completamente diferente do original (SP e handlers mudaram) — **confirma que o novo firmware NCP está gravado e o chip reiniciou nele com sucesso**.

**Status:** o EFR32MG21 agora roda firmware NCP EZSP oficial da Silicon Labs (`zigbee_ncp_uart_hw`), com UART configurado em PA5(TX)/PA6(RX), pronto para ser testado como coordenador Zigbee via ZHA/Zigbee2MQTT assim que a conexão física USB-serial for estabelecida (etapa 5, ainda pendente).

## Etapa 5 em andamento — achados físicos por foto (2026-09-05)

O responsável do projeto enviou fotos da placa aberta. Achados:

- **J3 tem serigrafia "ZIGBEE_DEBUG"** — confirma nome oficial do header SWD já em uso.
- **Novo header descoberto: "WIFI_DEBUG"**, perto do parafuso H3, com 4 pads pequenos oxidados/corroídos, não populados. Candidato a debug/UART do lado Wi-Fi/MCU principal (chip com marcação parcial "...0310"). Ainda não testado — pode ou não ser o mesmo barramento UART que vai para o módulo Zigbee (se for o mesmo, gravar direto lá exigiria isolar o chip Wi-Fi para evitar contenção de barramento).
- **Pergunta em aberto:** há uma peça com etiqueta idêntica à do REX3B21S (mesmo `Date:20240805`, `Code:1.2.21.99.10232`) perto dos pontos de teste **H2/ID1**, com seu próprio cabo coaxial de antena. Ainda não confirmado se é um **segundo módulo Zigbee separado** ou o mesmo módulo fotografado de outro ângulo. **Perguntado ao responsável do projeto, resposta pendente.**
- O responsável já soldou fios permanentes em **J3-1 (VCC/pino 5)** e **J3-6** (sinal ainda desconhecido) para facilitar acesso.

### Plano para localizar TXD(pino 3)/RXD(pino 4) fisicamente

Usando VCC(pino 5)=J3-1 e GND(pino 7)=J3-2 (já confirmados por continuidade) como âncoras na mesma fileira de 7 pads castelados do módulo:

- Pad imediatamente ao lado do VCC, na direção **oposta** ao GND → **RXD (pino 4)**.
- Pad duas posições nessa mesma direção → **TXD (pino 3)**.

A confirmar por continuidade (placa desligada) antes de soldar fios definitivos — ainda não confirmado, é a próxima ação pendente.

### Depois de identificar TXD/RXD (checklist combinado com o usuário)

1. Testar continuidade e confirmar TXD(3)/RXD(4) fisicamente (pendente).
2. Soldar 3 fios: TXD, RXD, GND.
3. Obter adaptador USB-serial de **3,3 V** (não 5V).
4. Conectar cruzado: TXD(módulo)→RX(adaptador), RXD(módulo)→TX(adaptador), GND→GND. **Não conectar VCC do adaptador** — a placa já é alimentada pela própria fonte.
5. Plugar no servidor, identificar a porta serial.
6. Configurar Zigbee2MQTT (driver `ember`) ou ZHA (`bellows`) apontando para essa porta, baud rate **115200** (definido no firmware).
7. Testar pareamento de dispositivos Zigbee reais.

**Sessão pausada em 2026-09-05 nesta etapa — retomar por aqui.**

### Build bem-sucedido (2026-09-05)

```
cmake --workflow --preset project   (em cmake_gcc/, toolchain arm-none-eabi-gcc 14.2 via SLT)
```

Resultado: `zigbee_ncp_uart_hw.out` / `.hex` / `.bin` / `.s37` gerados com sucesso (275/275 objetos, link ok).

```
   text	   data	    bss	    dec	    hex	filename
 212380	   1572	  96484	 310436	  4bca4	zigbee_ncp_uart_hw.out
```

- Flash usada: ~212 KB de ~496 KB disponíveis (512 KB − 16 KB do bootloader) — folga confortável.
- RAM usada (data+bss): ~95,8 KB de 96 KB — **margem apertada** (linker aceitou, mas quase no limite). Se formos adicionar funcionalidade extra no futuro (ex.: Green Power, mais binding table), pode ser necessário desabilitar algum componente opcional para liberar RAM.
- Aviso não-crítico: `RAIL PTI peripheral not configured` (interface de debug de RF, não usada — pode ser ignorado ou desabilitado depois).

**O fato do link ter sido bem-sucedido dentro do limite de RAM de 96 KB é uma confirmação indireta adicional de que `EFR32MG21A020F512IM32` (ou uma variante com a mesma RAM/flash) é a escolha certa** — se o chip real tivesse menos RAM, o linker teria recusado por estouro de memória.

Arquivos de firmware ainda **não gravados** em lugar nenhum: `C:\Users\guilh\silabs-tools\workspace\zigbee_ncp_uart_hw_mca1002\cmake_gcc\build\base\zigbee_ncp_uart_hw.{out,hex,bin,s37}`.

### Nota técnica: como o ASH/EZSP escolhe o UART físico

Investigando o código-fonte do componente `legacy_ncp_ash` (`ash-ncp.c`), descobri que o canal serial real usado pelo ASH/EZSP no NCP é determinado pela macro `ASH_PORT`, que por sua vez deriva do **iostream "VCOM"** configurado no projeto (`SL_CATALOG_IOSTREAM_USART_PRESENT`/`_EUSART_PRESENT` + `sl_iostream_usart_vcom_config.h`). Como geramos o projeto sem uma placa (`board`), o `slc generate` escolheu por padrão o componente `iostream_vuart` (UART virtual sobre o canal de debug/SWD, **não é um UART físico real**) — isso não serve pro nosso caso, que precisa de um UART físico de verdade (PA5/PA6) para conectar um adaptador USB-serial.

**Próxima ação:** trocar o componente `iostream_vuart` por `iostream_usart` (instância "vcom"), e configurar `SL_IOSTREAM_USART_VCOM_TX_PORT/PIN` e `_RX_PORT/PIN` para `PA5`/`PA6`, além de desabilitar controle de fluxo por hardware (CTS/RTS) já que o módulo documentado só expõe TXD/RXD (2 fios).

## Ferramentas instaladas nesta fase

| Ferramenta | Via | Local |
|---|---|---|
| GNU Arm Embedded Toolchain 14.2 | winget (`Arm.GnuArmEmbeddedToolchain`) | padrão do instalador |
| CMake | winget (`Kitware.CMake`) | padrão do instalador |
| Ninja | winget (`Ninja-build.Ninja`) | padrão do instalador |
| Microsoft OpenJDK 17 | winget (`Microsoft.OpenJDK.17`) | padrão do instalador |
| SLT-CLI 1.2.1 | download direto (`silabs.com/documents/public/software/slt-cli-1.2.1-windows-x64.zip`) | `C:\Users\guilh\silabs-tools\slt-cli\slt.exe` |
| GSDK, slc-cli, commander, zap, etc. | `slt install` (recipe em `C:\Users\guilh\silabs-tools\workspace\pkg.slt`) | `%USERPROFILE%\.silabs\slt\installs\` |
