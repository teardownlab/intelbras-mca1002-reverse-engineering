# Protocolo de provisionamento BLE do RE761-N4P (engenharia reversa do app Mibo)

Este documento consolida o protocolo BLE usado pelo app oficial **Mibo** (`br.com.intelbras.mibocam`) para parear e configurar o MCA 1002, obtido via decompilação (jadx) do APK oficial extraído de um aparelho Android do próprio usuário via ADB (`adb pull`). Isso é engenharia reversa de interoperabilidade com hardware próprio, não redistribuição do app — **o APK e o código decompilado NÃO são commitados neste repositório**, ficam apenas em `C:\Users\guilh\mca1002-backups\mibo-apk\` (local, fora do controle de versão).

Classes-fonte relevantes (pacote `com.mm.android.iotdeviceadd.bluetoothble.*`, prefixo `mm.android` = SDK interno da Dahua usado em vários apps white-label do grupo, incluindo o Mibo/Intelbras):

- `BleTransfer.java` — orquestra o handshake e o fluxo de mensagens
- `utils/EccCoder.java` — geração de chave ECC e ECDH
- `AESEncryptUtil.java` — cifra simétrica
- `BleDataResolveHelper.java` — parsing das mensagens recebidas (TLV)
- `BleWriteDataManager.java` — construção das mensagens enviadas (TLV)
- `BleNotifyDataManager.java` — remontagem de pacotes BLE fragmentados
- `com/lc/net/config/bluetooth/helper/BleHelperKt.java` — utilitários (derivação do IV a partir do `pid`)

## GATT

- Serviço: `0000fdd0-0000-1000-8000-00805f9b34fb`
- Característica de escrita (app→dispositivo): `0000fd01-0000-1000-8000-00805f9b34fb`
- Característica de notificação (dispositivo→app): `0000fd02-0000-1000-8000-00805f9b34fb`
- Descriptor CCC padrão (`00002902-...`) habilitado pra receber notify.

## Framing dos pacotes recebidos (notify, `0xfd02`) — fragmentação

Cada pacote BLE notificado tem o formato:

```
byte[0] = 0xD0            (marcador fixo, obrigatório)
byte[1] = tipo de fragmento
byte[2] = (não usado diretamente no reassembly — provavelmente sequência/tamanho)
byte[3..] = payload do fragmento
```

Tipos de fragmento (`byte[1]`):
- `1` = início de mensagem multi-pacote (reseta buffer se já havia uma sessão em andamento)
- `2` = continuação
- `4` = fim de mensagem multi-pacote (retorna o buffer acumulado completo)
- `5` = mensagem completa em um único pacote (equivalente a início+fim; retorna o buffer imediatamente)

Acumule `byte[3:]` de cada fragmento (tipos 1/2/4) num buffer; ao receber tipo `4` ou `5`, o buffer acumulado (incluindo o payload do próprio pacote final) é a mensagem completa, pronta para descriptografia (se aplicável) e parsing TLV.

## Framing das mensagens (nível lógico, após remontagem/descriptografia)

Formato TLV aninhado:

```
byte[0]    = cmdH
byte[1]    = cmdL          (identificador da mensagem = "cmdH_cmdL", ex: "0_16")
byte[2]    = tamanho total dos subcampos que seguem (em bytes, não em subcampos)
depois, repetido para cada subcampo:
  subId_hi (1 byte)
  subId_lo (1 byte)
  subLen   (1 byte)
  subData  (subLen bytes)
```

Os subcampos ficam acessíveis como chaves `"{subId_hi}_{subId_lo}"` → valor (string, geralmente hex ou ASCII dependendo do campo).

## Handshake completo

1. **App gera par de chaves ECC** — curva **`secp256k1`** (mesma curva do Bitcoin — bibliotecas padrão como `cryptography`/`ecdsa` em Python suportam nativamente).

2. **App envia sua chave pública, SEM criptografia** (comando `sendECPublicKey`):
   ```
   cmd = 0_1
   subcampo (0,3) = hex(X) + hex(Y)  — coordenadas do ponto público, concatenadas, convertidas de volta pra bytes
   ```

3. **Dispositivo responde com sua própria chave pública** (mensagem `cmd=0_1`, mesmo formato — X e Y concatenados em hex, 128 caracteres hex = 64 bytes = 32+32).

4. **App calcula segredo compartilhado:**
   ```
   shared_secret_raw = ECDH(private_key_app, public_key_device)
   aes_key = SHA256(shared_secret_raw)   # 32 bytes = AES-256
   ```

5. **IV da AES** — derivado do **`pid`** (product id do dispositivo, ex.: `sqNzDUSq`, visto no log de boot):
   ```python
   iv = pid.encode('utf-8')
   if len(iv) < 16:
       iv = iv + b'\x00' * (16 - len(iv))
   # se pid já tiver >=16 bytes, usa direto sem padding
   ```

6. **A partir daqui, toda mensagem (nos dois sentidos) é: `Base64(AES-256-CBC-PKCS7(payload_tlv, key=aes_key, iv=iv))`.**

7. **App pede informações do dispositivo:**
   ```
   payload = [0x00, 0x10, 0x00]     # cmd 0_16, sem subcampos
   ```
   Dispositivo responde (decriptografado) com `cmd=0_16` e subcampos:
   - `0_1` = PID (hex → string)
   - `0_2` = SN (hex → string)
   - `0_3` = **TOKEN** (hex → string) — provavelmente necessário para registrar o dispositivo na nuvem; papel exato em fluxos locais não confirmado.

8. **App envia config de WiFi** (comando `setWifiConfig`):
   ```
   payload = [0x00, 0x11, 0x02,
              0x00, 0x01, len(ssid), ...ssid (UTF-8)...,
              0x00, 0x02, len(pass), ...password (UTF-8)...]
   # cmd 0_17
   ```
   Dispositivo responde com `cmd=0_17`; sucesso se **não** houver subcampo `0_255` (`KEY_COMMON_ERROR`) com valor `"01"` (falha) — ausência de erro ou valor `"00"` = sucesso.

9. **Comando adicional descoberto — `setEntryUrl(addr, port)` (cmd 0_18):**
   ```
   payload = [0x00, 0x12, 0x02,
              0x00, 0x01, len(addr), ...addr (UTF-8)...,
              0x00, 0x02, 0x04, <port como 4 bytes big-endian>]
   ```
   **Isso reconfigura o endereço do servidor ("DRS"/entry point) que o dispositivo usa** — por padrão `iotaccess.easy4ipcloud.com` (visto no log de boot). **Esse é o candidato mais forte pra tornar o MCA1002 standalone/local**, sem depender da nuvem Dahua/Intelbras — se apontarmos esse endereço para um servidor próprio, o dispositivo passaria a "telefonar" para nós, não para a Dahua. **Não testado ainda.** Não sabemos: (a) se esse comando funciona sem autenticação/token adicional, (b) qual protocolo o dispositivo fala com esse "entry point" depois de configurado (precisaria ser descoberto/implementado do zero — provavelmente HTTP ou similar, baseado no padrão de nome "DRS" = provavelmente "Device Registration Service").

## Bloqueio atual para executar isso na prática

Mesmo com o protocolo completo mapeado, **ainda não conseguimos conectar via BLE ao MCA1002 de forma programática** (nem via `bleak`/Python no PC, nem via configurações nativas do Bluetooth do Windows, nem via um app scanner BLE genérico no celular Android do usuário) — apesar do firmware confirmar que o advertising inicia (`SSV_GAP_BLE_ADV_START_COMPLETE_EVT`) e do **app oficial Mibo conseguir parear normalmente** no mesmo celular. Isso sugere que o app usa algum mecanismo de descoberta mais específico que uma varredura BLE genérica não replica (ex.: filtro por manufacturer data específico, ou até Bluetooth clássico/SPP em paralelo ao BLE) — **não determinado ainda**.

**Próximo passo sugerido:** capturar o tráfego BLE real entre o celular Android e o MCA1002 durante um pareamento pelo app oficial, usando o recurso nativo do Android **"HCI snoop log"** (Opções de desenvolvedor → "Ativar log de snoop Bluetooth HCI"), depois analisar o arquivo `.cfa`/`.pklg` gerado no Wireshark. Isso revelaria exatamente como o app descobre e conecta ao dispositivo, e permitiria confirmar/corrigir os detalhes do protocolo acima com tráfego real capturado, sem depender só da leitura do código-fonte decompilado.
