# Referências — Intelbras MCA 1002

Links e documentos usados nesta investigação, por ordem de prioridade (oficial do fabricante do chip > ARM > OpenOCD > módulo/FCC > comunidade). Endereços de memória e qualquer operação potencialmente destrutiva devem ser validados contra a documentação oficial (Silicon Labs / ARM) antes de qualquer leitura ou escrita — não usar apenas fontes de comunidade para isso.

## Silicon Labs — EFR32MG21 (Series 2)

- [EFR32xG21 Wireless Gecko Reference Manual (PDF, oficial)](https://www.silabs.com/documents/public/reference-manuals/efr32xg21-rm.pdf) — fonte primária para memory map, DEVINFO, NVM3, registradores.
- [EFR32MG21 Multiprotocol Wireless SoC Family Data Sheet (PDF, oficial)](https://www.silabs.com/documents/public/data-sheets/efr32mg21-datasheet.pdf)
- [EFR32MG21 Series 2 Multiprotocol Wireless SoC — página do produto](https://www.silabs.com/wireless/zigbee/efr32mg21-series-2-socs)
- [EFR32MG21 Based Series 2 Wireless Modules — página de módulos](https://www.silabs.com/wireless/zigbee/efr32mg21-series-2-modules)
- [EFR32MG21 Gecko MCU and Peripheral Software Documentation (docs.silabs.com)](https://docs.silabs.com/mcu/5.8/efr32mg21/)

**Importante:** o EFR32MG21 é **Series 2**. Não usar endereços/registradores de documentação de Series 1 (ex.: EFR32MG1x) por analogia — o memory map muda entre séries.

## ARM — Cortex-M33

- [Arm Cortex-M33 Processor Technical Reference Manual (múltiplas revisões, developer.arm.com)](https://developer.arm.com/documentation/100230/latest/)
- [Arm Cortex-M33 Devices Generic User Guide](https://developer.arm.com/documentation/100235/latest/)

O core identificado neste projeto é **r0p3** (via `Examination succeed` do OpenOCD e via CPUID `0x410FD213` lido em `0xE000ED00`). Ao consultar o TRM, preferir a revisão mais próxima de r0p3 disponível; os registradores relevantes para debug/identificação (CPUID, PPB) são estáveis entre revisões menores.

## OpenOCD

- [OpenOCD — Documentation (portal oficial)](https://openocd.org/pages/documentation.html)
- [OpenOCD User's Guide (HTML)](https://openocd.org/doc/html/index.html)
- [OpenOCD User's Guide (PDF)](https://openocd.org/doc/pdf/openocd.pdf)
- [Repositório oficial (mirror read-only)](https://github.com/openocd-org/openocd/)

Versão em uso neste projeto: xPack OpenOCD `0.12.0+dev-02228-ge5888bda3-dirty` (build 2025-10-04).

## Rexense — REX3B21 / REX3B21S

- [REXENSE REX3B21 Low-Power Zigbee Module — User Manual](https://manuals.plus/rexense/rex3b21-low-power-zigbee-module-manual)
- [Data Sheet of ZigBee Module REX3B V5.0 (via FCC filing 2AOE2-REX3B)](https://fcc.report/FCC-ID/2AOE2-REX3B/5569691.pdf)
- [Página do produto REX3B no site da Rexense (瑞瀛物联)](https://www.rexense.com/h-pd-89.html)

Observação: os documentos encontrados são para a família **REX3B**/REX3B21; o módulo físico identificado no MCA 1002 traz a marcação **REX3B21S**. Tratar como o mesmo SoC/plataforma (EFR32MG21) até que uma diferença específica seja encontrada, mas não presumir que o pinout do datasheet do REX3B genérico é idêntico sem conferir contra as medições de continuidade já feitas (ver [`measurements.md`](measurements.md)).

## Módulo secundário RE761-N4P

**Nenhuma fonte oficial encontrada até agora.** Buscas iniciais por "RE761-N4P" não retornaram datasheet, FCC filing nem página de fabricante. Este item permanece **UNKNOWN** — ver [`hardware.md`](hardware.md) para os próximos passos de identificação sugeridos (busca por FCC ID na própria PCB, fotos em alta resolução do módulo).

## Notas de uso destas referências

- Antes de qualquer comando OpenOCD que leia um endereço de memória específico do EFR32MG21 (DEVINFO, NVM3, flash info), o endereço deve ser conferido no **EFR32xG21 Wireless Gecko Reference Manual** oficial acima — não apenas em headers de projetos de comunidade (ex.: `efm32-base`) ou em respostas de busca, que podem estar desatualizados ou ser de outra variante.
- Endereços de Series 1 (EFR32MG1x e afins) **não devem ser extrapolados** para o EFR32MG21 (Series 2).
