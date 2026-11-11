# PrivSMS

Mensagens P2P cifradas de ponta a ponta para Android. Sem servidor de mensagens: o texto e os
ficheiros vão **diretamente de telemóvel para telemóvel** por WebRTC DataChannel.

## Compilar (Linux Mint)

```bash
cd PrivSMS
./build.sh          # instala o que faltar (JDK 17, SDK Android, Gradle) e compila
./build.sh test     # idem + testes unitários
```
O APK fica em `./PrivSMS-debug.apk`. Instalar: `adb install -r PrivSMS-debug.apk`
(ou copiar para o telemóvel). Requer ligação à internet na 1.ª compilação.

## Como usar
1. Em cada telemóvel: **Definições** → defina o seu nome.
2. **+ → Contactos**: um mostra o QR Code, o outro "Ler QR Code" — e vice-versa (ambos têm de se adicionar).
3. Abra a conversa e escreva. A app faz um **pedido de ligação**; se o outro estiver online, abre o canal
   direto e envia. Caso contrário a mensagem fica ⏱ (por enviar) e é entregue quando o contacto ficar online.
   ✓ = enviada, ✓✓ = recebida (confirmada), ✓✓ azul = vista.
4. ⏳ (relógio no topo do chat): mensagens temporárias. O contador começa quando o destinatário vê a mensagem.
5. Perto um do outro, sem internet: **Contactos → Dispositivos próximos (Wi-Fi Direct)**.

## Arquitetura
- `crypto/` X25519 + HKDF-SHA256 + AES-256-GCM; chaves em EncryptedSharedPreferences (Android Keystore).
- `data/local/` Room + SQLCipher (passphrase aleatória, `secure_delete` ligado).
- `data/remote/` `WebRTCManager` / `P2PConnection` (DataChannel), `Signaling` (relay ntfy + TCP Wi-Fi Direct), `WifiDirectManager`.
- `data/repository/` fila de envio com ACK, reenvio, ficheiros em blocos, expiração, limpeza segura.
- `service/` serviço em primeiro plano, `WorkManager` (apagar expiradas mesmo com a app fechada), notificações.
- `ui/` Compose Material 3 (chats, chat, contactos, definições).

## Limitações / decisões
- **Sinalização:** dois telemóveis em redes diferentes (4G vs Wi-Fi) precisam de trocar o "olá" inicial (SDP/ICE)
  sem se conhecerem o IP. Sem NENHUM servidor isso é impossível; por isso usa-se um relay público (ntfy.sh)
  **só para esses pedidos cifrados**, sem cache e sem mensagens nem ficheiros. Pode usar um ntfy próprio
  (Definições). Metadados visíveis ao relay: IP e instante dos pedidos.
- NAT simétrico/CGNAT estrito pode impedir a ligação direta (só STUN, sem TURN): nesse caso fica "offline".
- Sem forward secrecy (chaves estáticas, sem double ratchet).
- O telemóvel deve manter a app/serviço ativo (otimização de bateria agressiva em alguns fabricantes pode matá-lo).
- Para ler as mensagens recebidas, o contacto tem de estar adicionado dos dois lados.
