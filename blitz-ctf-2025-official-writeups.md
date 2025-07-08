# Blitz CTF 2025 (Official Writeups)

These are the official writeups for the challenges I created for the Blitz CTF... forensics and misc :>

## Not Crypto (Misc)

> It can't be in the crypto category... everything is there for you to grab. Also don't try brute forcing&#x20;
>
> File link: [https://drive.google.com/file/d/1NH\_rCjNcHHgcA5EJhJoA5X3dsepBLW5s/view?usp=sharing](https://drive.google.com/file/d/1NH_rCjNcHHgcA5EJhJoA5X3dsepBLW5s/view?usp=sharing)

Sadly, this challenge remained unsolved till the end as I believe most people were spending time on trying to find an exploit in the cryptography part. That was the reason I kept the challenge **Not Crypto**.&#x20;

To solve this challenge the first step could be figured out from the fact that you are provided with a `.pyc` file instead of a normal `.py` file. Usually in this case you should think of something related to steganography, the password used to encrypt data was unknown so probably that was hidden. By doing some digging you come across:

{% embed url="https://github.com/AngelKitty/stegosaurus" %}

Simply running the extraction command reveals the hidden password:

```bash
./stegosaurus encrypt.cpython-36.pyc -x

Output:
Extracted payload: whoevenputsthepasswordhereandwhyisitsooooooLONG?
```

To figure out the encryption used you can use [https://github.com/rocky/python-uncompyle6](https://github.com/rocky/python-uncompyle6) to uncompile the `.pyc` file and get the orginal code. Then with basic crypto knowledge or even ChatGPT write the decryption script. On decryption you get the flag:

```
Blitz{h0ly_whY_w0uLd_u_puT_tH3_p4ssw0rd_th3re?}
```

## Diff n' Rae

> I love to use GitHub especially because of the compare feature. psst...do star my hide-secrets GitHub repo if you like it 🙂
>
> File link: [https://drive.google.com/file/d/1Fn6CVu0k1JXbI4qo4eDuIM7ThimopX8I/view?usp=sharing](https://drive.google.com/file/d/1Fn6CVu0k1JXbI4qo4eDuIM7ThimopX8I/view?usp=sharing)

The name and GitHub compare hints towards the **diff** tool in Linux. Reading other writeups some even used XOR to solve it which basically works in the same way only.

To solve we have to take diff of the strings of both the images. Running the following commands:

```bash
strings -n 1 img1.jpg > 1.txt
strings -n 1 img2.jpg > 2.txt
diff img1.jpg img2.jpg

Output:
3d2
< Qmx
8c7
< )
---
> pdHp7)
10c9
< ZDFm
---
> Rl8x
77d75
< U1
949a948
>       91N
4707d4705
< TNm
4770a4769
> dUx9
```

By reconstructing the base64 payload from the output we get `QmxpdHp7ZDFmRl8xU191NTNmdUx9` which decodes and gives us the flag as:

```
Blitz{d1fF_1S_u53fuL}
```
