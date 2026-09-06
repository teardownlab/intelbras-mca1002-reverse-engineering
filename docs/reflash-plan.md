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
| 1. Confirmar variante exata do chip | INFERRED como `EFR32MG21A020F512IM32` (512 KB flash, 96 KB RAM) — o link do firmware coube exatamente dentro do orçamento de RAM desse chip (ver abaixo), o que é uma confirmação indireta forte |
| 2. Toolchain headless (SLT) | **Concluído** — GSDK 2026.6.1, slc-cli 6.0.23, gcc-arm-none-eabi 14.2, LLVM embedded 21.1, CMake, Ninja, Commander, zap, tudo instalado via `slt install` |
| 3. Gerar projeto NCP-UART-HW customizado | **Concluído** — `iostream_usart` (instância `vcom`) configurado com USART0, TX=PA5, RX=PA6, sem controle de fluxo (2 fios) |
| 4. Compilar | **Concluído com sucesso** — ver detalhes abaixo |
| 5. Confirmar pinos PA5/PA6 fisicamente | Não iniciado |
| 6. Gravar via SWD | Não iniciado — aguarda confirmação explícita quando chegar a hora |
| 7. Testar com Z2M/ZHA | Não iniciado |

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
