---
hidden: true
---

# craccon CTF 2025

Brief descriptions and short writeups for the challenges I solved in craccon-ctf 2025.

## 1. PowerPlay

As AES encryption is being used and it has the underlying **SBOX** implementation with data leakage being there we can do a sidechannel attack. I also referred to some writeups like [https://github.com/emorchy/PicoCTF2023-PowerAnalysis](https://github.com/emorchy/PicoCTF2023-PowerAnalysis) that also used side channel attacks but in a different scenario. Specifically, I did CPA ([http://wiki.newae.com/Correlation\_Power\_Analysis](http://wiki.newae.com/Correlation_Power_Analysis)) and asked ChatGPT to write a quick script for me that implemented this method.

{% code fullWidth="true" %}
```python
import numpy as np
from pathlib import Path
import re

# ---------- Load data ----------
PT_PATH = Path('plaintexts.npy')      # shape (N, 16), dtype=uint8
TR_PATH = Path('power_traces.npy')    # shape (N, T), dtype=float32/float64
ENC_PATH = Path('enc.py')             # contains ciphertext hex as a comment

plain = np.load(PT_PATH)    # (N,16) uint8
traces = np.load(TR_PATH)   # (N,T)  float
N, T = traces.shape
assert plain.shape == (N, 16)

# ---------- AES S-box + Hamming Weight ----------
SBOX = np.array([
    0x63,0x7c,0x77,0x7b,0xf2,0x6b,0x6f,0xc5,0x30,0x01,0x67,0x2b,0xfe,0xd7,0xab,0x76,
    0xca,0x82,0xc9,0x7d,0xfa,0x59,0x47,0xf0,0xad,0xd4,0xa2,0xaf,0x9c,0xa4,0x72,0xc0,
    0xb7,0xfd,0x93,0x26,0x36,0x3f,0xf7,0xcc,0x34,0xa5,0xe5,0xf1,0x71,0xd8,0x31,0x15,
    0x04,0xc7,0x23,0xc3,0x18,0x96,0x05,0x9a,0x07,0x12,0x80,0xe2,0xeb,0x27,0xb2,0x75,
    0x09,0x83,0x2c,0x1a,0x1b,0x6e,0x5a,0xa0,0x52,0x3b,0xd6,0xb3,0x29,0xe3,0x2f,0x84,
    0x53,0xd1,0x00,0xed,0x20,0xfc,0xb1,0x5b,0x6a,0xcb,0xbe,0x39,0x4a,0x4c,0x58,0xcf,
    0xd0,0xef,0xaa,0xfb,0x43,0x4d,0x33,0x85,0x45,0xf9,0x02,0x7f,0x50,0x3c,0x9f,0xa8,
    0x51,0xa3,0x40,0x8f,0x92,0x9d,0x38,0xf5,0xbc,0xb6,0xda,0x21,0x10,0xff,0xf3,0xd2,
    0xcd,0x0c,0x13,0xec,0x5f,0x97,0x44,0x17,0xc4,0xa7,0x7e,0x3d,0x64,0x5d,0x19,0x73,
    0x60,0x81,0x4f,0xdc,0x22,0x2a,0x90,0x88,0x46,0xee,0xb8,0x14,0xde,0x5e,0x0b,0xdb,
    0xe0,0x32,0x3a,0x0a,0x49,0x06,0x24,0x5c,0xc2,0xd3,0xac,0x62,0x91,0x95,0xe4,0x79,
    0xe7,0xc8,0x37,0x6d,0x8d,0xd5,0x4e,0xa9,0x6c,0x56,0xf4,0xea,0x65,0x7a,0xae,0x08,
    0xba,0x78,0x25,0x2e,0x1c,0xa6,0xb4,0xc6,0xe8,0xdd,0x74,0x1f,0x4b,0xbd,0x8b,0x8a,
    0x70,0x3e,0xb5,0x66,0x48,0x03,0xf6,0x0e,0x61,0x35,0x57,0xb9,0x86,0xc1,0x1d,0x9e,
    0xe1,0xf8,0x98,0x11,0x69,0xd9,0x8e,0x94,0x9b,0x1e,0x87,0xe9,0xce,0x55,0x28,0xdf,
    0x8c,0xa1,0x89,0x0d,0xbf,0xe6,0x42,0x68,0x41,0x99,0x2d,0x0f,0xb0,0x54,0xbb,0x16
], dtype=np.uint8)
HW = np.array([bin(i).count("1") for i in range(256)], dtype=np.uint8)

# ---------- Normalize traces (per time sample) ----------
traces_z = (traces - traces.mean(axis=0)) / (traces.std(axis=0) + 1e-9)

# ---------- CPA core ----------
def cpa_byte(byte_idx: int):
    """Return (best_guess, best_abs_corr, best_time_index) for one key byte."""
    p = plain[:, byte_idx]                                  # (N,)
    guesses = np.arange(256, dtype=np.uint8)                # (256,)
    # Model matrix: rows=guesses, cols=samples -> shape (256, N)
    m = HW[SBOX[np.bitwise_xor(p[:, None], guesses[None, :])]].T.astype(np.float32)
    # Standardize each row
    m -= m.mean(axis=1, keepdims=True)
    m /= (m.std(axis=1, keepdims=True) + 1e-6)
    # Correlate: (256 x N) @ (N x T) -> (256 x T). Since standardized, this is Pearson ρ.
    corr = (m @ traces_z) / (len(p) - 1)
    abs_corr = np.abs(corr)
    g_idx, t_idx = np.unravel_index(np.argmax(abs_corr), abs_corr.shape)
    return int(g_idx), float(abs_corr[g_idx, t_idx]), int(t_idx)

key_bytes = []
byte_corrs = []
byte_times = []
for b in range(16):
    g, c, t = cpa_byte(b)
    key_bytes.append(g)
    byte_corrs.append(c)
    byte_times.append(t)

key = bytes(key_bytes)
key_hex = key.hex()
print("Recovered AES-128 key:", key_hex)
print("Per-byte best |corr|:", [round(c, 3) for c in byte_corrs])
print("Per-byte time indices:", byte_times)

# ---------- Decrypt ciphertext from enc.py (if present) ----------
try:
    enc_text = ENC_PATH.read_text()
    # Look for a long hex string after a '#'
    m = re.search(r"#\s*([0-9a-fA-F]{64,})", enc_text)
    if m:
        ct_hex = m.group(1).strip()
        from Crypto.Cipher import AES
        from Crypto.Util.Padding import unpad
        cipher = AES.new(key, AES.MODE_ECB)
        pt_padded = cipher.decrypt(bytes.fromhex(ct_hex))
        try:
            pt = unpad(pt_padded, 16)
            print("Decrypted plaintext:", pt)
        except ValueError:
            print("Decrypted (raw, unpad failed):", pt_padded)
    else:
        print("Ciphertext not found in enc.py comment. Skipping decryption.")
except FileNotFoundError:
    print("enc.py not found. Skipping decryption.")

```
{% endcode %}

## 2. Fake Estate

The alert in the challenge said that "**The Man In the Middle**" stopped us from accessing the desired endpoint. This reminded me of the **Next.js** middleware bypass [CVE](https://projectdiscovery.io/blog/nextjs-middleware-authorization-bypass). So I just had to attach the following header while making a request:

```json
x-middleware-subrequest: src/middleware:src/middleware:src/middleware:src/middleware:src/middleware
```

## 3. Pay2Win

Looking at the image given to us and the description of the challenge it was clear that pixelation was being used to hide the data, but I had no idea which tool to use or how it works. Checking the metadata and strings of the image revealed **greenshot**. Then I found this [Depixelization\_poc](https://github.com/spipm/Depixelization_poc) which clearly stated how to remove this blurring from greenshot. Cropping the image and running the tool gave the password for the admin panel. After logging in to it, I inspected the source code and inside it found the hidden flag.&#x20;

> Now starts the favourite part of CTFS for me **forensics and cloud.**

## 4. Snapshot

It was a memdump provided to us so I opened it with volatility and explored it. First I ran some basic commands like `windows.info, windows.pslist...`  but they did not reveal anything very interesting, I thought of pursuing the open **StikyNot.exe** but was too lazy. So I then did filescan and got:

```bash
0x1e5e9800	\Windows\System32\zh-CN\httpflag.ctf.defhawk.com8000flagsnapshot
```

I then tried accessing [http://flag.ctf.defhawk.com:8000/flag/](http://flag.ctf.defhawk.com:8000/flag/) but it asked for `x-api-key` so I just ran strings through snapshot and found it: `x-api-key: sn4psh0t_v0l4t1litty`  then I got flag.

## 5. To The Vault

I first proceeded the standard way and visited [https://azurectflab1.blob.core.windows.net/$web?restype=container\&comp=list](https://azurectflab1.blob.core.windows.net/$web?restype=container\&comp=list). This gave a list of files which I went through several times but did not get anything, so I opened **Microsoft Azure Storage Explorer** from where I changed the visibility settings to All Blobs and got to see the hidden `creds.txt` file.&#x20;

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

I then logged in the Azure CLI with these credentials and ran `az keyvault list -o table` which gave name of the vault as **ctflab1.** Then I ran a simple script to fetch all secret values which gave flag.

```sh
for s in $(az keyvault secret list --vault-name ctflab1 --query "[].name" -o tsv); do
  echo "== $s ==" 
  az keyvault secret show --vault-name ctflab1 --name "$s" --query value -o tsv
done
```

## 6. The Hidden Snap

It was interesting as I first got something unintended 🥲 and they patched it. I asked ChatGPT about the most commonly used places to store hidden files on a local filesystem and after few commands I got a hit on the path `cat /root/.hidden_snap_id`  and `snap-0e37d8eae0553406e` . I realised this is a public **snapshot id** for AWS because I had solved a similar challenge before from BSides Mumbai. I ran an automated script for all regions and got a hit in the ap-south-1 region.

```
aws ec2 describe-snapshots --snapshot-ids snap-0e37d8eae0553406e --region ap-south-1
```

I then created a volume from this snapshot, launched an EC2 instance and attached this volume. After this, I **ssh** into the instance, mounted `xvdf1` and explored it till I found the flag.
