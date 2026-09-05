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

### AP1 — identificado como DCI / Authentication Access Port (AAP) — ⚠️ CUIDADO

Comando:

```tcl
efr32.dap apid 1
```

Classificação: **READ-ONLY / SAFE** (mesmo comando `apid`, sem variante de escrita).

Resultado:

```text
0x54770002
```

Decodificação:

| Campo | Bits | Valor | Significado |
|---|---|---|---|
| Revision | [31:28] | `0x5` | específico da implementação |
| JEP106 continuation | [27:24] | `0x4` | banco 4 |
| JEP106 identity | [23:17] | `0x3B` | ARM Limited |
| Class | [16:13] | `0x8` | MEM-AP |
| Variant | [7:4] | `0x0` | — |
| Type | [3:0] | `0x2` | APB-AP |

**Interpretação e ALERTA:** pesquisa contra a documentação oficial Silicon Labs (**AN1190: Series 2 Secure Debug**, PDF oficial de `silabs.com`) indica que, nos EFR32 Series 2, a funcionalidade de debug lock/unlock e mass erase é implementada por um segundo Access Port chamado **Debug Challenge Interface (DCI)**, também referida como **Authentication Access Port (AAP)** em fontes de comunidade, exposto como um APB-AP separado do AHB-AP principal. O padrão observado (AHB-AP com IDR `0x84770001` + um segundo APB-AP) é consistente com relatos de terceiros para o EFR32MG21 especificamente.

Trecho literal do AN1190 (seção 3.2.2, Debug Challenge Interface):

> "The Debug Challenge Interface (DCI) is made available through commands in Simplicity Studio and Simplicity Commander. [...] For more information about DCI, see AN1303: Programming Series 2 Devices using the Debug Challenge Interface (DCI) and Serial Wire Debug (SWD)."

E, sobre o comando `Erase Device` disponível via DCI (seção 5.2 / Tabela "Debug Unlock Command Reference"):

> "Performs a device mass erase and resets the debug configuration to its initial unlocked state."

**Isso é exatamente o mecanismo de "recover"/mass erase mencionado nas regras de segurança deste projeto.** O AN1190 também documenta um comando `Read Lock Status` ("Returns the current debug lock status and configuration", disponibilidade "Always") que seria a forma correta de checar o estado de debug/security lock sem alterá-lo — mas o **protocolo exato de registradores da DCI (quais offsets de `apreg` correspondem a quê) ainda não foi lido** (está em AN1303, ainda não consultado neste projeto).

**REGRA ADOTADA:** nenhuma operação além de `apid` (leitura do IDR) será feita em AP1 até o protocolo da DCI (AN1303) ser lido e compreendido. Nenhum `apreg`/`dpreg`/`mdw` direcionado a AP1 deve ser executado sem essa pesquisa prévia.

**Status:** CONFIRMED — AP1 existe, é um APB-AP ARM. INFERRED (alta confiança, pendente de confirmação em AN1303) — corresponde à DCI/AAP usada para debug lock/unlock/mass erase. UNKNOWN — protocolo exato de registradores.

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
| AP1 IDR | CONFIRMED — `0x54770002` = MEM-AP (APB-AP), ARM. INFERRED = DCI/AAP (Silicon Labs), usada para debug lock/unlock/mass erase. **Não tocar além de `apid` sem ler AN1303 antes.** |
| Estado de debug/security lock (DCI/AP lock do EFR32 Series 2) | UNKNOWN — ainda não verificado |
| Part number exato / flash size / RAM size do EFR32MG21 | UNKNOWN — ainda não lido de DEVINFO |
| Leitura de flash principal | Não tentada |
| Leitura de SRAM | Não tentada |

## Protocolo completo da DCI (AP1) — pesquisado, hardware NÃO tocado ainda

Pesquisa feita em 2026-09-05 na documentação oficial Silicon Labs atual (`docs.silabs.com`, seção "Programming Series 2 Devices Using DCI and SWD" — sucessora do AN1303, que está formalmente deprecated e hoje só contém uma página de capa redirecionando para `docs.silabs.com`). Nenhum destes registradores foi lido/escrito no dispositivo ainda; isto é só pesquisa.

### Conexão da DCI via SWD (conforme documentação oficial)

1. Sequência JTAG-to-SWD.
2. Ler IDCODE (SWD DP reg 0) — `0x6BA02477` para Series 2 com Cortex-M33 (bate com o DPIDR que já lemos).
3. ABORT (SWD DP reg 0 = `0x0000001E`) para limpar erros/sticky flags.
4. STAT (SWD DP reg 1 = `0x50000000`) para requisitar power-up do sistema/debug domain.
5. SELECT (SWD DP reg 2 = `0x01000000`) para apontar a interface SWD do chip para a DCI.
6. CSW (SWD AP reg 0 = `0x22000002`) para configurar transferência de 32 bits.

### Registradores da DCI (expostos via AP1)

| Registrador | Descrição | Endereço | Observações |
|---|---|---:|---|
| `DCI_WDATA` | Escreve dado/comando para a DCI | `0x1000` | Comando |
| `DCI_RDATA` | Lê dado de resposta da DCI | `0x1004` | Resposta |
| `DCI_STATUS` | Status dos acessos à DCI | `0x1008` | Bit 0 = `WPENDING` (write pendente; escritas em WDATA são descartadas enquanto ativo). Bit 8 = `RDATAVALID` (resposta pronta; limpo ao ler RDATA). |

Mecanismo de acesso (por word): selecionar o registrador escrevendo seu endereço em **SWD AP register 1** (ex.: `0x1008` para status, `0x1000` para write, `0x1004` para read), depois ler/escrever o valor via **SWD AP register 3**. Em termos de OpenOCD, isso mapeia para `efr32.dap apreg 1 4 <addr>` (seleciona) seguido de `efr32.dap apreg 1 0xc <valor>` (ou leitura), aproximadamente — a sintaxe exata ainda precisa ser validada com cautela antes de qualquer execução.

### Formato de comando/resposta da DCI

Toda chamada começa escrevendo um word de comprimento (incluindo esse próprio word; mínimo 8) seguido do Command ID de 32 bits em `DCI_WDATA`, e payload adicional se aplicável. A resposta é um word (comprimento [15:0] + código de resposta [31:16]) seguido do payload de resposta.

Códigos de resposta: `0`=OK, `1`=INVALID_COMMAND, `2`=AUTHORIZATION_ERROR, `3`=INVALID_SIGNATURE, `4`=BUS_ERROR, `5`=INTERNAL_ERROR, `6`=CRYPTO_ERROR, `7`=INVALID_PARAMETER, `8`=INTEGRITY_ERROR, `9`=SECUREBOOT_ERROR, `10`=SELFTEST_ERROR, `11`=NOT_INITIALIZED.

### Lista de comandos SE via DCI — classificados por risco

| Command ID | Nome | Classificação | Observação |
|---|---|---|---|
| `0xFE00` | Read Serial Number | **SAFE (leitura)** | Lê número de série provisionado de fábrica (16 bytes). |
| `0xFE01` | Get Status | **SAFE (leitura)** | Inclui boot status, versões de firmware SE/MCU, **debug lock status**, secure boot config. |
| `0xFE04` | Read User Configuration | **SAFE (leitura)** | Config não-reconfigurável do OTP. |
| `0xFF08` | Read Public Key | **SAFE (leitura)** | Lê chave pública armazenada. |
| `0x4311` | **Read Lock Status** | **SAFE (leitura)** | "This command is used to read the lock status of the debug port." Retorna bits: debug lock, device erase enabled, secure debug lock, debug lock hardware status, invasive/non-invasive locks. |
| `0x430C` | Apply Lock | ⚠️ **DESTRUTIVO/ALTERA CONFIG** | Habilita o debug lock. |
| `0x430D` | Enable Secure Debug | ⚠️ **ALTERA CONFIG** | Habilita secure debug. |
| `0x430E` | Disable Secure Debug | ⚠️ **ALTERA CONFIG** | Desabilita secure debug. |
| `0x430F` | **Erase Device** | 🛑 **MASS ERASE** | "Performs a device mass erase... clears and verifies the main flash and RAM." Apaga o firmware original. **NUNCA EXECUTAR.** |
| `0x4310` | Disable Device Erase | 🛑 **IRREVERSÍVEL** | Desabilita permanentemente o comando Erase Device. One-time, permanente. **NUNCA EXECUTAR.** |
| `0x4312` | Set Debug Restrictions | ⚠️ **ALTERA CONFIG** | Define bits de restrição de debug (DBGLOCK/NIDLOCK/SPIDLOCK/SPNIDLOCK). |
| `0xFF00` | Initialize OTP | 🛑 **IRREVERSÍVEL, one-time** | Provisionamento de fábrica. **NUNCA EXECUTAR.** |
| `0xFF07` | Initialize Public Key | 🛑 **IRREVERSÍVEL, one-time** | **NUNCA EXECUTAR.** |
| `0xFF0B` | Initialize AES Key | 🛑 **IRREVERSÍVEL, one-time** (dispositivos HSE) | **NUNCA EXECUTAR.** |
| `0x4302` / `0x4303` | SE Image Check / Apply | 🛑 **ALTERA FIRMWARE DO SE** | Upgrade de firmware do SE. **NUNCA EXECUTAR.** |

### Ponto de decisão importante

`Read Lock Status` (`0x4311`) e `Get Status` (`0xFE01`) são, **por especificação da Silicon Labs, comandos de consulta somente leitura, sempre disponíveis, sem efeito colateral** — é exatamente o que o item 7 do plano de investigação pede ("verificar estado de debug/security/protection SEM alterá-lo").

Porém, **mecanicamente**, invocar qualquer comando da DCI — mesmo estes de leitura — exige **escrever** o word de comando em `DCI_WDATA` (via uma sequência de registros AP1 `apreg`). Isso não escreve em flash, NVM, nem altera nenhuma configuração persistente do dispositivo (são comandos de consulta ao Secure Engine, não ao controlador de memória), mas é, no nível de comando OpenOCD, uma operação de escrita em registrador (`apreg ... <valor>`), o que toca a letra da regra "nenhuma escrita" definida para esta fase do projeto, mesmo não tocando o espírito dela (nenhuma alteração persistente).

**Por isso este projeto NÃO executará `Read Lock Status`/`Get Status` sem confirmação explícita do responsável pelo projeto**, mesmo estando confiantes de que são operações seguras segundo a documentação oficial. Ver decisão registrada em [`experiments.md`](experiments.md) / conversa do projeto.

## Próximos passos (somente READ-ONLY / SAFE)

1. Identificar a variante exata do EFR32MG21 e ler DEVINFO (part number, revisão, flash size, RAM size) — pesquisar antes o endereço correto para EFR32 **Series 2** na documentação oficial Silicon Labs (não extrapolar endereços de Series 1).
2. ~~Enumerar os Access Ports disponíveis no DAP.~~ Feito para AP0 e AP1 (ver acima). Pode haver mais (AP2, AP3...) — verificar depois, com a mesma cautela dada a AP1.
3. **Ler o AN1303 (Silicon Labs, oficial) antes de qualquer outro comando em AP1.** AN1303 descreve o protocolo de registradores da DCI necessário para usar com segurança o comando `Read Lock Status` (que, segundo o AN1190, é somente leitura e sempre disponível) sem risco de acionar `Erase Device` ou outro comando por engano.
4. Verificar o estado de debug/security/protection **sem alterá-lo** — via AP1/DCI somente depois do passo 3, ou eventualmente via outro mecanismo mais seguro (ex.: registrador de status exposto no espaço de memória via AP0, se existir e for documentado).
5. Somente depois, planejar leitura de flash/SRAM/NVM3 para backup.

Nenhum desses passos foi executado ainda além do que está documentado acima.
