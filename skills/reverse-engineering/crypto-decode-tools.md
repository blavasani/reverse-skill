# Encryption, Decryption, Encoding, and Decoding Tools Quick Reference

> Encrypted, encoded, and hashed data are common in reverse engineering and CTF work. This document lists the most useful tools by scenario.

---

## Automatic Identification + Decryption (When the Cipher Is Unknown)

| Tool | Stars | Purpose | Link |
|------|-------|---------|------|
| **Ciphey** | 18k+ | Automatic AI identification and decryption (supports 50+ encoding, encryption, and hash types) | https://github.com/Ciphey/Ciphey |
| **CyberChef** | 29k+ | Online/offline encoding and decoding Swiss Army knife (drag-and-drop operations) | https://github.com/gchq/CyberChef |
| **dcode.fr** | — | Online collection of 900+ cipher, encoding, and mathematics tools | https://www.dcode.fr/ |

### Using Ciphey

```bash
pip install ciphey
# 自动检测并解密
ciphey -t "密文"
# 从文件读取
ciphey -f encrypted.txt
```

Ciphey supports Base64/32/16, Caesar, Vigenere, XOR, AES (weak keys), Morse, Binary, Hex, URL encoding, HTML entities, hash identification, and more.

### Using CyberChef

```text
在线版：https://gchq.github.io/CyberChef/
离线版：下载 GitHub Release 的 HTML 文件直接打开

常用 Recipe：
- From Base64 → 解 Base64
- XOR → 异或解密（可暴力尝试 key）
- AES Decrypt → AES 解密
- Magic → 自动检测编码类型
```

---

## Hash Identification and Cracking

| Tool | Purpose | Link |
|------|---------|------|
| **hashID** | Identify hash types (MD5/SHA/bcrypt, etc.) | https://github.com/psypanda/hashID |
| **hash-identifier** | Same as above, Python version | https://github.com/blackploit/hash-identifier |
| **haiti** | Modern hash identification tool (more accurate) | `gem install haiti` |
| **Hashcat** | GPU hash cracking | https://hashcat.net/ |
| **John the Ripper** | CPU hash cracking | https://www.openwall.com/john/ |
| **hashes.com** | Online hash lookup (rainbow tables) | https://hashes.com/ |

```bash
# 识别哈希类型
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
# 输出: [+] MD5

# haiti（更准确）
haiti '5f4dcc3b5aa765d61d8327deb882cf99'

# Hashcat 破解
hashcat -m 0 hash.txt rockyou.txt  # MD5
hashcat -m 1000 hash.txt rockyou.txt  # NTLM
```

---

## RSA Attacks

| Tool | Purpose | Link |
|------|---------|------|
| **RsaCtfTool** | Automated RSA attacks (20+ attack methods) | https://github.com/Ganapati/RsaCtfTool |
| **SageMath** | Mathematical computation (large integer factorization/elliptic curves) | https://www.sagemath.org/ |
| **factordb.com** | Online large integer factorization lookup | http://factordb.com/ |
| **yafu** | Local large integer factorization | https://github.com/bbuhrow/yafu |

```bash
# RsaCtfTool 自动攻击
python RsaCtfTool.py --publickey pub.pem --private
python RsaCtfTool.py --publickey pub.pem --uncipherfile cipher.txt

# 支持的攻击：
# Wiener、Boneh-Durfee、Fermat、Pollard p-1、Williams p+1
# Common modulus、Small q、Hastads、Noveltyprimes 等
```

---

## XOR Analysis

| Tool | Purpose | Link |
|------|---------|------|
| **xortool** | XOR key length guessing + known-plaintext attacks | https://github.com/hellman/xortool |
| **CyberChef XOR** | Visual XOR operations | Built into CyberChef |

```bash
# 猜测 XOR key 长度
xortool encrypted_file
# 用猜测的 key 长度解密
xortool -l 4 -c 00 encrypted_file

# 已知明文攻击（知道部分明文）
xortool-xor -f encrypted -s "known_plaintext"
```

---

## Classical Ciphers

| Cipher Type | Tool | Description |
|-------------|------|-------------|
| Caesar | CyberChef / dcode.fr | Brute-force 25 shifts |
| Vigenere | dcode.fr / Ciphey | Requires guessing key length |
| Substitution | quipqiup.com | Automatic solving using frequency analysis |
| Enigma | dcode.fr | Online simulator |
| Rail Fence | dcode.fr / CyberChef | Rail fence cipher |
| Playfair | dcode.fr | Requires a key |
| Morse | CyberChef | Convert dots and dashes to text |
| Bacon | dcode.fr | Binary steganography |
| ROT13/47 | CyberChef / `tr` | Simple substitution |

---

## Encoding Identification and Conversion

| Encoding | Identifying Characteristics | Decoding Method |
|----------|-----------------------------|-----------------|
| Base64 | Ends in `=` or `==`; character set A-Za-z0-9+/ | `base64 -d` / CyberChef |
| Base32 | Uppercase letters + 2-7; ends in `=` | CyberChef |
| Base58 | No 0/O/I/l; common in short identifier encodings | CyberChef |
| Hex | Only 0-9a-f; even length | `xxd -r -p` / CyberChef |
| URL encoding | `%XX` format | `urldecode` / CyberChef |
| HTML entities | `&#XX;` or `&amp;` format | CyberChef |
| Unicode escape | `\uXXXX` format | Python `decode('unicode_escape')` |
| JWT | `xxxxx.yyyyy.zzzzz` (three Base64URL segments) | jwt.io / CyberChef |
| Brainfuck | Only the eight characters `><+-.,[]` | Online interpreter |
| Ook! | Only `Ook.` `Ook!` `Ook?` | Online interpreter |

---

## Identifying Encryption in Reverse Engineering

### Identifying Algorithms by Constants

| Constants/Features | Algorithm |
|--------------------|-----------|
| `0x67452301, 0xEFCDAB89, 0x98BADCFE, 0x10325476` | MD5 |
| `0x6A09E667, 0xBB67AE85, 0x3C6EF372` | SHA-256 |
| `0x63, 0x7C, 0x77, 0x7B` (start of S-Box) | AES |
| `0x243F6A88` (π in hexadecimal) | Blowfish |
| `0xB7E15163, 0x9E3779B9` | RC5/RC6/TEA |
| `0x61707865` ("expa") | ChaCha20/Salsa20 |
| `0xC6EF3720` | XTEA |

### Identifying Algorithms by Behavior

| Behavioral Characteristics | Possible Algorithms |
|----------------------------|---------------------|
| 256-byte lookup table + swap operations | RC4 |
| 16-byte blocks + multiple permutation rounds | AES |
| Feistel structure (left/right swaps) | DES/Blowfish/TEA |
| Large integer multiplication/modular exponentiation | RSA |
| Elliptic curve point operations | ECDSA/ECDH |
| Fixed 64-round loop | TEA/XTEA |
| 32 rounds + delta constant | XTEA |

---

## Automated Cryptanalysis

| Tool | Purpose | Link |
|------|---------|------|
| **FeatherDuster** | Automated cryptanalysis framework | https://github.com/nccgroup/featherduster |
| **PkCrack** | ZIP known-plaintext attack | https://www.unix-ag.uni-kl.de/~conrad/krypto/pkcrack.html |
| **bkcrack** | ZIP known-plaintext attack (modern version) | https://github.com/kimci86/bkcrack |
| **z3** | SMT solver (constraint solving) | https://github.com/Z3Prover/z3 |
| **angr** | Symbolic execution (automatic input solving) | https://angr.io/ |

---

## Quick Decision Tree

```text
拿到一段未知数据：

1. 看长度和字符集
   - 只有 hex 字符 → 可能是 hex 编码或哈希
   - 末尾有 = → Base64
   - 三段点分 → JWT
   - 32/40/64 字符 hex → 哈希（MD5/SHA1/SHA256）

2. 用 Ciphey 自动尝试
   ciphey -t "数据"

3. 如果 Ciphey 失败 → 用 CyberChef Magic 模式

4. 如果是哈希 → hashID 识别类型 → Hashcat/John 破解

5. 如果是 RSA → RsaCtfTool 自动攻击

6. 如果是 XOR → xortool 分析 key

7. 如果是传统 ZIP 加密 → 优先使用 `bkcrack` 已知明文攻击，不要先做无证据的密码暴力

8. 如果是自定义加密 → IDA/Ghidra 逆向算法 → 手写解密脚本
```

---

## Online Resources

| Resource | Link | Purpose |
|----------|------|---------|
| CyberChef | https://gchq.github.io/CyberChef/ | General-purpose encoding and decoding |
| dcode.fr | https://www.dcode.fr/ | 900+ cipher tools |
| quipqiup | https://quipqiup.com/ | Automatic substitution cipher solving |
| factordb | http://factordb.com/ | RSA large integer factorization |
| jwt.io | https://jwt.io/ | JWT decoding/validation |
| hashes.com | https://hashes.com/ | Reverse hash lookup |
| crackstation | https://crackstation.net/ | Online hash cracking |
