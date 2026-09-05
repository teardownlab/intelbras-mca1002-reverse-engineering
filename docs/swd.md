# Acesso SWD ao EFR32MG21 (módulo REX3B21S) — Intelbras MCA 1002

Este documento registra a configuração de debug SWD, as regras de segurança adotadas e os resultados obtidos até agora. Todos os resultados aqui são de operações **read-only**.

## Regra mais importante

Neste estágio do projeto, a prioridade é:

1. identificar completamente o hardware;
2. preservar o firmware original;
3. preservar dados de fabricação/calibração;
4. entender a arquitetura;
5. descobrir o firmware/protocolo atual;
6. **somente depois** avaliar modificações.

**Proibido neste estágio:** erase, mass erase, recover, unlock, flash write, program, load_image, mww/mwh/mwb, write_memory, alteração de option bytes/lock bits/debug config/bootloader/NVM, ou qualquer escrita na memória do dispositivo. Se debug protection estiver habilitada, **parar** — não tentar desbloquear. Alguns mecanismos de unlock/recover do EFR32 Series 2 podem apagar a flash.

Cada comando OpenOCD documentado abaixo é classificado explicitamente como **READ-ONLY / SAFE** ou **POTENTIALLY DESTRUCTIVE**. Somente comandos READ-ONLY / SAFE são executados nesta fase.

## Hardware de debug

- **Probe:** Raspberry Pi Pico (RP2040 original), firmware `debugprobe_on_pico.uf2` (Raspberry Pi Debug Probe). Enumera no Windows como CMSIS-DAPv2.
  - VID:PID = `2e8a:000c`
  - Firmware version = `2.0.0`
  - SWD suportado
- **Software:** xPack OpenOCD `0.12.0+dev-02228-ge5888bda3-dirty`, build 2025-10-04.

## Conexões físicas

```text
Pico GP2  -> J3-5 (SWCLK)
Pico GP3  -> J3-4 (SWDIO)
Pico GND  -> J3-2 (GND)
```

- **RESET (J3-3): NÃO conectado.**
- **J3-1 (3,3 V / VTref): NÃO conectado ao Pico.**
- **J3-6: NÃO conectado** (função ainda desconhecida).
- O MCA 1002 é alimentado normalmente pela própria fonte.
- O Pico é alimentado pelo USB do computador.
- Alimentações não são compartilhadas entre Pico e MCA 1002 além do GND comum.

## Configuração do target no OpenOCD

Comandos usados para estabelecer o target Cortex-M33 via SWD:

```tcl
swd newdap efr32 cpu -expected-id 0x6ba02477
dap create efr32.dap -chain-position efr32.cpu
target create efr32.cpu cortex_m -dap efr32.dap -ap-num 0
```

Classificação: **READ-ONLY / SAFE**. Esses comandos apenas declaram ao OpenOCD como enumerar o DAP e criar o objeto de target; não escrevem no dispositivo. O `-expected-id` é uma verificação (o OpenOCD recusa prosseguir se o DPIDR lido não bater), não uma escrita.

## Resultados confirmados

### Primeiro contato SWD

```text
SWD DPIDR = 0x6BA02477
```

Status: **CONFIRMED**. DPIDR (Debug Port Identification Register) lido com sucesso, confirmando comunicação SWD funcional com o Debug Port do EFR32MG21.

### Exame do core

```text
Cortex-M33 r0p3 processor detected
8 breakpoints
4 watchpoints
Examination succeed
```

Status: **CONFIRMED**. OpenOCD reconheceu o core como Cortex-M33 revisão r0p3, com 8 breakpoints e 4 watchpoints — consistente com a implementação Cortex-M33 usada no EFR32MG21. "Examination succeed" indica que o AP (Access Port) e o core foram enumerados e ficaram acessíveis para operações subsequentes de debug.

### Leitura de CPUID via AHB-AP

Comando:

```tcl
mdw 0xE000ED00 1
```

Classificação: **READ-ONLY / SAFE**. `mdw` (memory display word) lê e exibe uma palavra de 32 bits do espaço de memória do target via o AP configurado; não escreve nada. O endereço `0xE000ED00` é o registrador `CPUID` no espaço de sistema ARMv8-M (PPB — Private Peripheral Bus), mapeado de forma idêntica em toda a família Cortex-M, incluindo Cortex-M33; não é um endereço específico do fabricante do SoC.

Resultado:

```text
0xE000ED00: 410FD213
```

Status: **CONFIRMED**.

```text
CPUID = 0x410FD213
```

Interpretação do CPUID (`0x410FD213`), conforme o formato padrão ARM (Implementer / Variant / Architecture / PartNo / Revision):

| Campo | Bits | Valor | Significado |
|---|---|---|---|
| Implementer | [31:24] | `0x41` | ARM |
| Variant | [23:20] | `0x0` | r0 |
| Architecture | [19:16] | `0xF` | definido por CPUID adicional (ARMv7-M/v8-M usam este encoding) |
| PartNo | [15:4] | `0xD21` | Cortex-M33 |
| Revision | [3:0] | `0x3` | p3 |

Isso confirma de forma independente, a partir de um registrador padrão ARM (não específico do fabricante), que o core é **Cortex-M33 r0p3** — consistente com o que o OpenOCD já havia reportado na etapa de exame do core.

### Acesso de leitura via AHB-AP

Status: **CONFIRMED — funcional**. A leitura acima via `mdw` confirma que o AHB-AP (Access Port número 0, conforme `-ap-num 0` na configuração do target) permite leitura de memória mapeada do sistema. Isso é uma evidência inicial (não uma confirmação completa) de que debug/memory access protection não está bloqueando leituras no espaço testado (PPB/system space). Ainda não testado: leitura da região de flash principal, SRAM, nem verificação formal do registro de segurança/debug lock do EFR32 Series 2 (ver "Próximos passos").

### Enumeração de Access Ports — AP0 IDR

Comando:

```tcl
efr32.dap apid 0
```

Classificação: **READ-ONLY / SAFE**. Conforme a documentação oficial do OpenOCD (`doc/openocd.texi`, comando `$dap_name apid`), `apid` "Displays ID register from AP num" — não existe variante de escrita para este comando (ao contrário de `apreg`, que só escreve se um `value` for explicitamente passado). É uma leitura pura do IDR (Identification Register) do Access Port, um registrador de identificação da infraestrutura CoreSight/ADIv5, fora do espaço de memória de aplicação do dispositivo.

Resultado:

```text
0x84770001
```

Decodificação (formato padrão ARM ADIv5 para AP IDR):

| Campo | Bits | Valor | Significado |
|---|---|---|---|
| Revision | [31:28] | `0x8` | específico da implementação |
| JEP106 continuation | [27:24] | `0x4` | banco 4 |
| JEP106 identity | [23:17] | `0x3B` | ARM Limited |
| Class | [16:13] | `0x8` | MEM-AP |
| Variant | [7:4] | `0x0` | — |
| Type | [3:0] | `0x1` | AHB-AP |

**Interpretação:** AP0 é um MEM-AP do tipo AHB-AP, JEP106 = ARM Limited. Confirma formalmente que o AP0 (já usado com sucesso para ler o CPUID) é um Access Port válido do tipo correto para acessar o barramento de sistema do EFR32MG21. Não revela, por si só, o estado de debug/security lock — isso está em registrador(es) separado(s) ainda não lido(s).

**Status:** CONFIRMED — AP0 = MEM-AP (AHB-AP), ARM.

**Ambiente de execução:** OpenOCD rodado em modo batch (não interativo), mesma sequência de bring-up já validada (`swd newdap` / `dap create` / `target create`, sem `reset`), seguido de `init` e do comando acima, encerrando com `shutdown`. Nenhum reset, erase, write ou unlock foi executado.

## Resumo de status

| Item | Status |
|---|---|
| Conexão SWD física (J3-4/J3-5/J3-2) | CONFIRMED |
| DPIDR | CONFIRMED = `0x6BA02477` |
| Core reconhecido pelo OpenOCD | CONFIRMED = Cortex-M33 r0p3 |
| CPUID via AHB-AP | CONFIRMED = `0x410FD213` |
| Leitura de memória via AHB-AP (PPB) | CONFIRMED — funcional |
| AP0 IDR (enumeração de Access Ports) | CONFIRMED — `0x84770001` = MEM-AP (AHB-AP), ARM |
| Estado de debug/security lock (DCI/AP lock do EFR32 Series 2) | UNKNOWN — ainda não verificado |
| Part number exato / flash size / RAM size do EFR32MG21 | UNKNOWN — ainda não lido de DEVINFO |
| Leitura de flash principal | Não tentada |
| Leitura de SRAM | Não tentada |

## Próximos passos (somente READ-ONLY / SAFE)

1. Identificar a variante exata do EFR32MG21 e ler DEVINFO (part number, revisão, flash size, RAM size) — pesquisar antes o endereço correto para EFR32 **Series 2** na documentação oficial Silicon Labs (não extrapolar endereços de Series 1).
2. Enumerar os Access Ports disponíveis no DAP.
3. Verificar o estado de debug/security/protection **sem alterá-lo**.
4. Somente depois, planejar leitura de flash/SRAM/NVM3 para backup.

Nenhum desses passos foi executado ainda além do que está documentado acima.
