# Muddy Water (SwampCTF 2025)

The main objective was to analyze a packet capture of a brute‑force attack against our Domain Controller, determine which account was successfully authenticated and then extract and crack the NTLMv2 hash to recover the password.

**Step‑1: Load and Filter the PCAP**

* Open the provided PCAP file in Wireshark.
*   Apply the display filter:

    ```
    smb2.cmd == 1 && smb2.nt_status == 0
    ```

    This isolates all NTLM authentication exchanges without and error response. We only got 1 such packet.

<figure><img src="https://github.com/user-attachments/assets/cee1e255-718b-4a36-bb6f-a618a7a9fe8a" alt=""><figcaption></figcaption></figure>

***

**Step‑2: Identify the Authentication Attempts**

<figure><img src="https://github.com/user-attachments/assets/78925b66-bd4c-42fc-8c44-5cbff96ee051" alt=""><figcaption></figcaption></figure>

Looking at packets near to 72074. Two NTLMSSP\_AUTH requests were observed:

* **Packet 72068:**\
  Authentication for **DESKTOP-0TNOE4V\stacktracerghost**
* **Packet 72069:**\
  Authentication for **DESKTOP-0TNOE4V\hackbackzip** \\

But packet **Packet 72070** shows a _STATUS\_LOGON\_FAILURE_ response. It can be seen that the failed attempt is for **stacktracerghost** so **hackbackzip** must have had the successful login

![image](https://github.com/user-attachments/assets/b97faee1-8f56-4b94-b471-f8e771d9d229)

***

**Step‑3: Extract the NTLMv2 Challenge/Response**

* **Extract the Server Challenge:**
  * Locate the NTLMSSP\_CHALLENGE packet (**packet 72065**).
  * In its details, copy the server challenge (`d102444d56e078f4`).
*   **Extract the NTLMv2 Response:**

    * Open the successful NTLMSSP\_AUTH packet (for **hackbackzip**).
    *   In the NTLMSSP section to find the long hexadecimal blob, which is the NTLMv2 response.

        {% code overflow="wrap" %}
        ```
        eb1b0afc1eef819c1dccd514c962320101010000000000006f233d3d9f9edb01755959535466696d0000000002001e004400450053004b0054004f0050002d00300054004e004f0045003400560001001e004400450053004b0054004f0050002d00300054004e004f0045003400560004001e004400450053004b0054004f0050002d00300054004e004f0045003400560003001e004400450053004b0054004f0050002d00300054004e004f00450034005600070008006f233d3d9f9edb010900280063006900660073002f004400450053004b0054004f0050002d00300054004e004f004500340056000000000000000000
        ```
        {% endcode %}

    The first 32 characters are the NTProofSTR

***

**Step‑4: Construct the Hash File for Hashcat**

Hashcat expects NTLMv2 hashes in the following format (mode 5600):

```
USERNAME::DOMAIN:SERVERCHALLENGE:NTProofStr:NTLMv2_BLOB
```

Using the extracted values:

* **Username:** `hackbackzip`
* **Domain:** `DESKTOP-0TNOE4V`
* **Server Challenge:** `d102444d56e078f4`
* **NTProofStr:** `eb1b0afc1eef819c1dccd514c9623201`
*   **NTLMv2 Blob:**

    {% code overflow="wrap" %}
    ```
    01010000000000006f233d3d9f9edb01755959535466696d0000000002001e004400450053004b0054004f0050002d00300054004e004f0045003400560001001e004400450053004b0054004f0050002d00300054004e004f0045003400560004001e004400450053004b0054004f0050002d00300054004e004f0045003400560003001e004400450053004b0054004f0050002d00300054004e004f00450034005600070008006f233d3d9f9edb010900280063006900660073002f004400450053004b0054004f0050002d00300054004e004f004500340056000000000000000000
    ```
    {% endcode %}

The final hash line becomes:

{% code overflow="wrap" %}
```
hackbackzip::DESKTOP-0TNOE4V:d102444d56e078f4:eb1b0afc1eef819c1dccd514c9623201:01010000000000006f233d3d9f9edb01755959535466696d0000000002001e004400450053004b0054004f0050002d00300054004e004f0045003400560001001e004400450053004b0054004f0050002d00300054004e004f0045003400560004001e004400450053004b0054004f0050002d00300054004e004f0045003400560003001e004400450053004b0054004f0050002d00300054004e004f00450034005600070008006f233d3d9f9edb010900280063006900660073002f004400450053004b0054004f0050002d00300054004e004f004500340056000000000000000000
```
{% endcode %}

Save this to a file named, for example `ntlm.hash`.

***

**Step‑5: Crack the Hash with Hashcat**

Run Hashcat using mode 5600:

```bash
hashcat -m 5600 ntlm.hash /path/to/wordlist.txt
```

Hashcat processes the hash and using the provided wordlist recovers the plaintext password. ( I used **rockyou.txt** )

***

**Step‑6: Forming the flag**

Upon cracking the password was recovered as **pikeplace**

**Flag:**

`swampCTF{hackbackzip:pikeplace}`
