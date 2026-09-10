# Firmware original do EFR32MG21 (REX3B21S) — dump sanitizado

Este diretório contém o dump completo da flash original do EFR32MG21 (módulo Zigbee Rexense REX3B21S) do Intelbras MCA 1002, lido via SWD **antes** de qualquer gravação de firmware novo (ver [`../docs/reflash-plan.md`](../docs/reflash-plan.md) e [`../docs/swd.md`](../docs/swd.md) para o histórico completo).

## Arquivo

| Campo | Valor |
|---|---|
| Nome | `mca1002_efr32mg21_original_firmware_sanitized.bin` |
| Tamanho | 524.288 bytes (512 KiB — flash completa do EFR32MG21A020) |
| Endereço base | `0x00000000` (início da flash principal do EFR32MG21) |
| Firmware | Rexense `REXENSE_HA_COO_Stk6710_MG215_1.7.3`, build 28/07/2023 (Gecko Bootloader v1.8 em `0x0000`–`0x4000` + aplicação Zigbee Coordinator em `0x4000`+) |
| Lido em | 2026-09-05, via OpenOCD (`dump_image`), read-only, antes de qualquer gravação |
| SHA-256 do dump **original, não sanitizado** (não publicado) | `34b5db7e121351d8139b7d0956bd952314a87d58d9b452728e7dac4f511cf170` |
| SHA-256 deste arquivo (sanitizado, publicado aqui) | `358921bc93ed213085f2cfeb1d2473ec5c224865e97127ce10cf419076128548` |

## O que foi removido (sanitização, 2026-09-10)

Antes de publicar, o dump completo foi analisado byte a byte em busca de segredos (ver metodologia abaixo). Foram encontradas **3 páginas de 4 KiB da região NVM3** (final da flash, endereços `0x078000`, `0x07A000` e `0x07C000`) contendo dados não-apagados, das quais **uma continha material criptográfico real**:

- Um bloco de ~48–64 bytes de alta entropia (estatisticamente indistinguível de dados aleatórios) imediatamente adjacente a uma chave conhecida — consistente com uma **chave de rede Zigbee (Network Key) real gerada pelo coordenador**, e/ou material de segurança relacionado (ex.: chave de link do Trust Center específica do dispositivo).
- Logo em seguida, a string ASCII **`ZigBeeAlliance09`** — esta *não* é segredo: é a chave de link padrão pública definida pela especificação Zigbee 3.0 para comissionamento sem install code (usada por qualquer coordenador Zigbee no mundo como fallback).
- O restante dessas 3 páginas contém apenas estruturas internas do NVM3 (contadores de página, tabela de objetos) — não são segredos, mas fazem parte do mesmo mecanismo de armazenamento e foram removidas junto por simplicidade e segurança.

**O que foi feito:** essas 3 páginas de 4 KiB (12.288 bytes no total, ~2,3% do arquivo) foram preenchidas com `0xFF` (o padrão de "flash apagada"), tornando-as indistinguíveis de uma região NVM3 nunca provisionada. **Nada mais foi alterado** — bootloader e aplicação (código executável, ~224 KB) permanecem bit-a-bit idênticos ao dump original, verificado por diff.

**O que NÃO foi removido/alterado:** todo o restante do arquivo — bootloader completo, aplicação Zigbee completa, vetor de interrupções, `ApplicationProperties_t`, strings de identificação (`REXENSE_HA_COO_...`), comandos AT, etc. — está intacto e idêntico ao hardware real. Isso preserva 100% do valor para quem quiser estudar o firmware original (engenharia reversa, comparação, etc.); o que foi removido é exclusivamente estado de runtime específico deste dispositivo (chaves), não código.

## Metodologia da sanitização

1. Mapeamento da flash em blocos de 4 KiB para identificar regiões não-apagadas (a maior parte da flash está em `0xFF`, exceto bootloader+app no início e algumas páginas NVM3 no final).
2. Varredura de entropia byte a byte, restrita à região NVM3 candidata (`0x070000`–`0x080000`), para localizar blocos estatisticamente aleatórios (candidatos a chaves) distintos de dados estruturados/repetitivos (que são normais no NVM3 e não são segredos).
3. Varredura de strings sensíveis (`password`, `secret`, `private`, `wifi`, `ssid`, `token`, etc.) em toda a região de código (bootloader + aplicação) — nada encontrado além de nomes de arquivo-fonte usados em macros de assert (`mfg-token.c`, `token.c`), que não são segredos.
4. Redação das 3 páginas NVM3 identificadas, preenchendo com `0xFF`.
5. Verificação por diff byte a byte: confirmado que **apenas** essas 3 páginas (12.288 bytes) diferem do dump original; todo o restante é bit-a-bit idêntico.

## Dump original (não sanitizado)

O dump original, completo e não sanitizado, **não é publicado neste repositório** — permanece apenas local, fora do Git, pelo motivo acima (contém material criptográfico real específico deste dispositivo). Seu hash SHA-256 está registrado acima e em [`../docs/swd.md`](../docs/swd.md) para fins de verificação/proveniência, caso seja necessário confirmar a integridade de uma cópia obtida por outro meio.
