---
title: "Evading Signature-based Detection in VBA Macros"
excerpt: "Techniques to evade WinAPI signatures and bypass EDR detection in VBA macros."
categories:
    - EDR-Evasion
tags:
    - EDR-Evasion
toc: true
show_date: true
classes: wide
---

# Introduction

Phishing with macro attachments remains a popular technique for establishing command and control (C2) channels on victim machines. However, modern Endpoint Detection and Response (EDR) solutions have become adept at detecting common VBA macro templates. While revisiting my knowledge from the OSEP course, I found that the templates taught in the course are now flagged by modern EDRs. 

In this blog, I will document some techniques I use to evade signature-based detection in VBA macros, particularly focusing on shellcode runners.

---

# Figuring Out the Common Issues

To understand the challenges, I used the VBA template from [OffensiveVBA](https://github.com/S3cur3Th1sSh1t/OffensiveVBA/blob/main/src/Shellcode_CreateThread.vba) as an example. During testing, I identified two primary issues that lead to detection by EDRs.

### 1. Raw Shellcode

Embedding raw shellcode directly in the script is a red flag for EDRs. For example, a simple array containing shellcode bytes will likely trigger detection:

```vbs
Kqrfipip = Array(232, 130, 0, 0, 0, 96, 137, 229, 49, 192, ..., 46, 101, 120, 101, 0)
```

### 2. WinAPI Usage

A typical shellcode execution workflow involves the following WinAPI functions:
1. **VirtualAlloc**: Allocates memory with specific protection.
2. **RtlMoveMemory / WriteProcessMemory**: Copies or writes data to the allocated memory.
3. **CreateThread**: Creates a thread to execute the shellcode.

This pattern is a well-known indicator of compromise (IOC). EDRs often scan for these APIs in VBA macros. For instance, the following declaration and usage of `RtlMoveMemory` can be flagged:

```vbs
Private Declare PtrSafe Function RtlMoveMemory Lib "kernel32" (ByVal Krldhufs As LongPtr, ByRef Gsvspq As Any, ByVal Djjdc As Long) As LongPtr
...
For Rxsqoxe = LBound(Kqrfipip) To UBound(Kqrfipip)
    Tpjln = Kqrfipip(Rxsqoxe)
    Gczn = RtlMoveMemory(Clsghvido + Rxsqoxe, Tpjln, 1)
Next Rxsqoxe
```

---

# Useful Techniques for Evasion

To bypass these detections, I applied the following techniques:

## 1. Multiple Encoding

Encoding the shellcode is a common technique to prevent EDRs from identifying malicious payloads. In the OSEP course, XOR encoding was introduced as a simple yet effective method. Here's an example of XOR encoding in VBA:

```vbs
buf = Array(232, 130, 0, 0, 0, 96, ..., 255, 213)

For i = LBound(buf) To UBound(buf)
    buf(i) = buf(i) Xor 23
Next i
```

However, during testing, I found that even XOR-encoded shellcode was detected by modern EDRs. To address this, I applied an additional layer of encoding on top of the XOR-encoded shellcode. I chose **Caesar Cipher**, which shifts each byte by a fixed number of positions. The encryption and decryption formulas are as follows:

```
Encryption: C = P + K
Decryption: P = C - K
*K = Addition key
*C = Ciphertext
*P = Plaintext
```

By combining XOR and Caesar Cipher, I was able to evade detection on a testing EDR. While I won't share the exact implementation, I recommend experimenting with other encoding techniques, such as **rot13** or **base64**, to further obfuscate the shellcode.

---

## 2. WinAPI Declaration with Alias

Another effective technique is to rename WinAPI functions using the `Alias` keyword in the declaration. This helps avoid detection based on known API names.

For example, instead of directly declaring `RtlMoveMemory`, you can alias it as `CopyMemory`:

```vbs
Private Declare PtrSafe Sub CopyMemory Lib "KERNEL32" Alias "RtlMoveMemory"
...
For Rxsqoxe = LBound(Kqrfipip) To UBound(Kqrfipip)
    Tpjln = Kqrfipip(Rxsqoxe)
    Indjas = Clsghvido + Rxsqoxe
    Gczn = CopyMemory(Indjas, Tpjln, 1)
Next Rxsqoxe
```

This simple change can help bypass signature-based detection that looks for specific API names.

---

# Conclusion

Evading signature-based detection in VBA macros requires a combination of techniques, including encoding the shellcode and obfuscating WinAPI declarations. While the examples in this blog focus on XOR encoding, Caesar Cipher, and API aliasing, there are many other methods you can explore to achieve the same goal.
