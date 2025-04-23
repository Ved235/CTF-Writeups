# ASK Me Again (dawgCTF 2025)
You’re given an SDR capture (ask.iq) containing an Amplitude‑Shift Keying (ASK)–modulated signal. Recover the binary stream, convert to ASCII and extract the flag.
> **Hint:**  
> - The file name `ask.iq` → ASK modulation.  
> - “Isn’t binary a great way to represent ASCII characters?” → it’s just on/off keyed bits.

## 1. File Format & I/Q Reconstruction

1. **Determine sample format**  
   Try reading as 16‑bit signed integers—result: even number of samples, so it’s interleaved I/Q:

   ```python
   import numpy as np

   raw = np.fromfile('ask.iq', dtype=np.int16)
   assert len(raw) % 2 == 0, "Expected interleaved I/Q pairs"
   ```

2. **De‑interleave & normalize**  
   ```python
   i = raw[0::2].astype(np.float32)
   q = raw[1::2].astype(np.float32)
   # Normalize 16‑bit signed to ±1.0
   iq = (i + 1j*q) / 32768.0
   ```

At this point, `iq` is a complex baseband signal where amplitude encodes our bits.

---

## 2. Envelope Detection

ASK = on/off carrier. Extract the instantaneous amplitude:

```python
env = np.abs(iq)
```

A quick peek at its min/max confirms two plateaus—carrier‑off (0) vs carrier‑on (≈1.38).

---

## 3. Threshold Slicing

Slice that analog envelope into a raw 0/1 stream:

```python
thresh = (env.max() + env.min()) / 2
bits_raw = env > thresh    # boolean array: True=1, False=0
```

> **Note:** Without smoothing, you’ll see tiny 1–4 sample “blips” around every transition.

---

## 4. Suppress Jitter

To clean up those edge blips, apply a short moving average:

```python
window = np.ones(100) / 100
env_sm = np.convolve(env, window, mode='same')
bits = env_sm > ((env_sm.max() + env_sm.min()) / 2)
```

From here on we work with `bits`, a much cleaner boolean stream.

---

## 5. Calculating Samples‑Per‑Bit = 480

- **SDR capture rate:** 48 000 samples/sec  
- **Suspected baud rate:** ~100 bits/sec (common low‑speed ASK)  
- $\text{samples per bit (SPB)} = \frac{48\,000}{100} = 480$

We **hard‑code**:
```python
SPB = 480
```

---

## 6. Symbol Slicing & Majority Vote

Fold each block of 480 samples into one bit:

```python
nbits = len(bits) // SPB
symbols = bits[:nbits*SPB].reshape(nbits, SPB)
decoded_bits = (symbols.mean(axis=1) > 0.5).astype(int)
```

Now `decoded_bits` is a clean array of 0/1 values—one per transmitted bit.

---

## 7. Pack into ASCII Bytes

1. **Truncate** to a whole number of bytes:
   ```python
   nbytes = len(decoded_bits) // 8
   decoded_bits = decoded_bits[:nbytes*8]
   ```

2. **Group & convert** (MSB first):
   ```python
   chars = []
   for i in range(nbytes):
       byte = decoded_bits[i*8:(i+1)*8]
       val  = int(''.join(str(b) for b in byte), 2)
       chars.append(chr(val))
   message = ''.join(chars)
   print("Flag:", message)
   ```

---

## 9. Full, Self‑Contained Script

Save this as `decode_ask.py`:

```python
import numpy as np

def main():
    # 1) Read raw I/Q as int16
    raw = np.fromfile('ask.iq', dtype=np.int16)
    if len(raw) % 2 != 0:
        raise RuntimeError("Odd number of IQ samples")

    # 2) De‑interleave & normalize
    i = raw[0::2].astype(np.float32)
    q = raw[1::2].astype(np.float32)
    iq = (i + 1j*q) / 32768.0

    # 3) Envelope detect
    env = np.abs(iq)

    # 4) Smooth (optional)
    window = np.ones(100) / 100
    env = np.convolve(env, window, mode='same')

    # 5) Threshold slice
    thresh = (env.max() + env.min()) / 2
    bits = env > thresh

    # 6) Hard‑coded samples per bit
    SPB = 480
    print(f"Using SPB = {SPB} samples/bit")

    # 7) Fold & majority vote
    nbits = len(bits) // SPB
    symbols = bits[:nbits*SPB].reshape(nbits, SPB)
    decoded_bits = (symbols.mean(axis=1) > 0.5).astype(int)

    # 8) Truncate & ASCII pack
    nbytes = len(decoded_bits) // 8
    decoded_bits = decoded_bits[:nbytes*8]
    flag = ''.join(
        chr(int(''.join(str(b) for b in decoded_bits[i*8:(i+1)*8]), 2))
        for i in range(nbytes)
    )

    print("Decoded flag:", flag)

if __name__ == "__main__":
    main()
```

Run:

```bash
python decode_ask.py
```

**Output:**
```
Using SPB = 480 samples/bit
Decoded flag: DawgCTF{D3M0DUL4710N_1S_FUN}
```
