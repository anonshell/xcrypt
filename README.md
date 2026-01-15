# xcrypt

**End-to-end encryption plugin for WeeChat IRC**

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![WeeChat](https://img.shields.io/badge/WeeChat-plugin-green.svg)](https://weechat.org/)

xcrypt provides seamless end-to-end encryption for WeeChat IRC messages using modern, industry-standard cryptography. Encrypt your private messages and channel conversations with a simple shared password - messages are automatically encrypted before sending and decrypted upon receiving.

📚 **[Full Documentation](https://anonshell.com/xcrypt/)**

---

## Table of Contents

- [Features](#features)
- [Security](#security)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage](#usage)
  - [Setting Up Master Passphrase](#setting-up-master-passphrase)
  - [Encrypting Conversations](#encrypting-conversations)
  - [Managing Encryption](#managing-encryption)
- [Commands Reference](#commands-reference)
- [How It Works](#how-it-works)
- [Message Format](#message-format)
- [Visual Indicators](#visual-indicators)
- [Storage & Persistence](#storage--persistence)
- [Compatibility](#compatibility)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

---

## Features

| Feature | Description |
|---------|-------------|
| 🔐 **Strong Encryption** | AES-256-GCM authenticated encryption (same as used by TLS 1.3) |
| 🔑 **Secure Key Derivation** | PBKDF2-SHA256 with 600,000 iterations (OWASP 2024 recommended minimum) |
| ⚡ **Automatic Operation** | Messages are transparently encrypted/decrypted without manual intervention |
| 👁️ **Visual Indicators** | Clear green `E>` prefix shows when messages are encrypted |
| 📁 **Per-Target Passwords** | Set different passwords for each nick or channel |
| 🔒 **Secure Storage** | Passwords encrypted at rest using a master passphrase |
| 💬 **Full Message Support** | Works with regular messages and `/me` actions |
| 🌐 **Cross-Platform** | Works anywhere WeeChat runs (Linux, macOS, BSD, Windows WSL) |

---

## Security

xcrypt uses modern, well-audited cryptographic primitives:

| Component | Implementation | Details |
|-----------|---------------|---------|
| **Cipher** | AES-256-GCM | 256-bit key, authenticated encryption with associated data (AEAD) |
| **Key Derivation** | PBKDF2-SHA256 | 600,000 iterations for message encryption |
| **Salt** | Random | 16 bytes (128 bits) per message |
| **Nonce/IV** | Random | 12 bytes (96 bits) per message |
| **Authentication** | GCM Tag | 16 bytes (128 bits) integrity verification |

### Why These Choices?

- **AES-256-GCM**: Provides both confidentiality and integrity in a single operation. Used by TLS 1.3, SSH, and IPsec. Hardware-accelerated on modern CPUs (AES-NI).

- **PBKDF2 with 600,000 iterations**: Meets [OWASP 2024 guidelines](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) for password-based key derivation. Slows down brute-force attacks significantly.

- **Random salt per message**: Even if the same password is used, each message produces completely different ciphertext. Prevents pattern analysis.

- **Random nonce per message**: Ensures semantic security - identical plaintexts encrypt to different ciphertexts.

---

## Requirements

- **WeeChat** - Any recent version (tested on 3.0+)
- **Python** - 3.8 or newer
- **cryptography** - Python cryptography library

---

## Installation

### 1. Install the cryptography library

```bash
pip install cryptography
```

Or on Debian/Ubuntu:
```bash
sudo apt install python3-cryptography
```

### 2. Download xcrypt.py

```bash
# Download to WeeChat's Python autoload directory
curl -o ~/.local/share/weechat/python/autoload/xcrypt.py \
  https://raw.githubusercontent.com/anonshell/xcrypt/main/xcrypt.py
```

Or manually download and place in:
- **Linux/macOS**: `~/.local/share/weechat/python/autoload/`
- **Legacy path**: `~/.weechat/python/autoload/`

### 3. Load the script

If WeeChat is already running:
```
/script load xcrypt.py
```

Or restart WeeChat - scripts in `autoload/` load automatically.

### 4. Verify installation

```
/xcrypt
```

You should see the help message with available commands.

---

## Quick Start

```
# 1. Set your master passphrase (protects stored passwords)
/xcrypt passphrase MySecretMasterPass123

# 2. Enable encryption for a nick or channel
/xcrypt set friend SharedSecretPassword
/xcrypt set #private-channel ChannelPassword123

# 3. Start chatting - messages are now encrypted!
```

**Important**: Both parties must use the same password to communicate. Share the password through a secure out-of-band channel (in person, Signal, etc.).

---

## Usage

### Setting Up Master Passphrase

The master passphrase encrypts your stored passwords at rest. **This is required for secure operation.**

```
/xcrypt passphrase <your-master-passphrase>
```

- Minimum 8 characters
- You'll need to enter this each time you start WeeChat
- Without it, passwords are stored in plain text (a warning will be shown)

**Example:**
```
/xcrypt passphrase Tr0ub4dor&3correct
```

### Encrypting Conversations

To enable encryption for a specific nick or channel:

```
/xcrypt set <target> <password>
```

**Examples:**
```
# Encrypt private messages with "alice"
/xcrypt set alice OurSharedSecret123

# Encrypt a channel
/xcrypt set #secret-ops ChannelPassword!@#
```

**Notes:**
- Run this command from any buffer on the same IRC server
- The password must be at least 8 characters
- Both parties must use the exact same password

### Managing Encryption

**Remove encryption for a target:**
```
/xcrypt del <target>
```

**List all encrypted targets:**
```
/xcrypt list
```

**Check encryption status of current buffer:**
```
/xcrypt status
```

---

## Commands Reference

| Command | Description |
|---------|-------------|
| `/xcrypt` | Show help and current status |
| `/xcrypt passphrase <pass>` | Set master passphrase for secure password storage |
| `/xcrypt set <target> <password>` | Enable encryption for a nick or channel |
| `/xcrypt del <target>` | Disable encryption for a nick or channel |
| `/xcrypt list` | List all targets with encryption enabled |
| `/xcrypt status` | Show encryption status for the current buffer |

---

## How It Works

### Encryption Flow (Outgoing Messages)

```
┌─────────────────────────────────────────────────────────────┐
│                     Your Message                            │
│                    "Hello, World!"                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Generate Random Salt (16 bytes)                │
│              Generate Random Nonce (12 bytes)               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│         PBKDF2-SHA256 (password + salt, 600k iter)          │
│                    → 256-bit AES key                        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              AES-256-GCM Encrypt (key, nonce)               │
│                    → ciphertext + tag                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│           Combine: salt + nonce + ciphertext + tag          │
│                    Base64 encode                            │
│                    Add "+ENC:" prefix                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  +ENC:B4J7kP2m... (sent over IRC)                           │
└─────────────────────────────────────────────────────────────┘
```

### Decryption Flow (Incoming Messages)

The process is reversed:
1. Detect `+ENC:` prefix
2. Base64 decode the payload
3. Extract salt, nonce, and ciphertext
4. Derive the same key using PBKDF2
5. Decrypt and verify with AES-256-GCM
6. Display with `E>` prefix indicating successful decryption

---

## Message Format

Encrypted messages are sent in this format:

```
+ENC:<base64-encoded-payload>
```

The payload structure (before base64 encoding):

| Offset | Length | Content |
|--------|--------|---------|
| 0 | 16 bytes | Random salt |
| 16 | 12 bytes | Random nonce |
| 28 | variable | Ciphertext |
| -16 | 16 bytes | GCM authentication tag |

**Example encrypted message:**
```
+ENC:xK9mN2pL4qR7sT1vW3yB5zA8cE0fG2hI4jK6lM8nO0pQ2rS4tU6vW8xY0zA2bC4dE6fG8hI0jK2lM4nO6pQ8rS0tU2vW4xY6zA8bC0dE2fG4hI6jK8lM0nO2pQ4rS6tU8vW0xY2zA4bC6dE8fG0h
```

---

## Visual Indicators

xcrypt uses color-coded prefixes to show message status:

| Indicator | Meaning |
|-----------|---------|
| `E>` (green) | Message was successfully decrypted |
| `[DECRYPT FAILED]` | Decryption failed (wrong password or corrupted) |
| No prefix | Message was not encrypted |

**Example conversation:**
```
<alice> +ENC:xK9mN2pL4qR7s...        ← What others see
E> Hello, this is encrypted!         ← What you see (decrypted)
```

---

## Storage & Persistence

### Password Storage

Encryption passwords are stored in WeeChat's configuration:
- **Location**: `~/.local/share/weechat/plugins.conf` (or `~/.weechat/plugins.conf`)
- **Format**: Encrypted with your master passphrase using AES-256-GCM
- **Key derivation**: PBKDF2 with 100,000 iterations (faster for local storage)

### Configuration Options

All xcrypt settings are stored under `plugins.var.python.xcrypt.*`:

| Option | Description |
|--------|-------------|
| `passphrase_hash` | Verification hash for master passphrase |
| `password_keys` | Comma-separated list of encrypted targets |
| `password.<server>.<target>` | Encrypted password for each target |

### Persistence Across Sessions

- Passwords persist across WeeChat restarts
- You must re-enter your master passphrase each session to unlock them
- Without the master passphrase, encrypted passwords cannot be decrypted

---

## Compatibility

### IRC Clients

xcrypt uses a custom encryption format. Both parties must use xcrypt to communicate securely. The encrypted messages will appear as `+ENC:...` to users without xcrypt.

### Message Length

IRC has message length limits (typically 512 bytes total). Encrypted messages are longer than plaintext due to:
- Base64 encoding (~33% overhead)
- Salt, nonce, and authentication tag (~44 bytes)
- `+ENC:` prefix (5 bytes)

**Recommendation**: Keep messages under ~300 characters for best compatibility.

### CTCP & Actions

- `/me` actions are fully supported and encrypted
- Other CTCP messages (VERSION, PING, etc.) are **not** encrypted to maintain protocol compatibility

### Unicode & Special Characters

Full UTF-8 support. Emojis, non-Latin scripts, and special characters work correctly.

---

## Troubleshooting

### "cryptography library not found"

Install the cryptography package:
```bash
pip install cryptography
# or
pip3 install cryptography
# or (Debian/Ubuntu)
sudo apt install python3-cryptography
```

### "Incorrect passphrase"

You entered a different passphrase than the one used to encrypt your passwords. If you've forgotten it, you'll need to:
1. Delete the stored passwords: `/unset plugins.var.python.xcrypt.*`
2. Set a new passphrase and re-add your encryption passwords

### Messages showing as "+ENC:..."

The other party either:
- Doesn't have xcrypt installed
- Has a different password configured
- Hasn't set up encryption for your nick

### "[DECRYPT FAILED]" messages

- Wrong password configured
- Message was corrupted in transit
- Message wasn't actually encrypted with xcrypt

### Script won't load

Check WeeChat's core buffer for error messages:
```
/buffer core.weechat
```

Common issues:
- Python version too old (need 3.8+)
- Missing cryptography library
- File permissions issue

---

## Security Considerations

### What xcrypt DOES protect

✅ Message content from network observers (ISPs, network admins)  
✅ Message content from IRC server operators  
✅ Message content if logs are obtained without your password  
✅ Stored passwords (when master passphrase is set)  

### What xcrypt does NOT protect

❌ **Metadata** - Who you're talking to, when, and how often  
❌ **Traffic analysis** - Message timing and frequency patterns  
❌ **Compromised endpoints** - If your or the recipient's machine is compromised  
❌ **Weak passwords** - Use strong, unique passwords  
❌ **Key exchange** - You must share passwords securely out-of-band  

### Best Practices

1. **Use strong passwords** - 16+ characters, mix of letters/numbers/symbols
2. **Unique passwords per target** - Don't reuse passwords across channels/nicks
3. **Secure password exchange** - Share passwords in person or via encrypted messaging (Signal, etc.)
4. **Strong master passphrase** - Protects all your stored passwords
5. **Verify recipients** - Confirm you're talking to who you think you are

---

## Contributing

Contributions are welcome! Please feel free to submit issues and pull requests.

### Development Setup

```bash
git clone https://github.com/anonshell/xcrypt.git
cd xcrypt
pip install cryptography weechat-script-lint
```

### Code Quality

Before submitting:
```bash
weechat-script-lint xcrypt.py
# Should show: Score: 100 / 100
```

---

## License

This project is licensed under the GNU General Public License v3.0 - see the [LICENSE](LICENSE) file for details.

```
SPDX-License-Identifier: GPL-3.0-or-later
```

---

## Author

**xcrypt AnonShell**  
📧 contact@anonshell.com  
🌐 https://anonshell.com

---

<p align="center">
  <i>Privacy is not about having something to hide.<br>Privacy is about having something to protect.</i>
</p>
