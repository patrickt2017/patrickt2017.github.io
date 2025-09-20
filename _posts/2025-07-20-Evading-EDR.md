---
title: "Tactics for Evading Static PE and Memory-Based Detection"
excerpt: "A quick sharing of simple ways to evade statfic PE and memory scanning."
categories:
    - EDR-Evasion
tags:
    - EDR-Evasion
toc: true
show_date: true
classes: wide
---

# Introduction
A few weeks ago I developed a .NET loader called [VEHNetLoader](https://github.com/patrickt2017/VEHNetLoader) and later I tried to run it in other EDR solutions. It has been immediately detected and blocked. Hence, it encouraged me to figure out what is going on and look for improvements.

# Problems with the Original Loader
## Scanning on Import Directory Table
To create a CLR instance, `CLRCreateInstance` in `metahost.h` and `mscoree.lib` is used as follows.
```c
hr = CLRCreateInstance(&CLSID_CLRMetaHost, &xIID_ICLRMetaHost, (LPVOID*)&pMetaHost);
```

```c
STDAPI CLRCreateInstance(REFCLSID clsid, REFIID riid, /*iid_is(riid)*/ LPVOID *ppInterface);
```
_The definition of CLRCreateInstance_

When the header and the library are included in the loader, the import directory table of its Portable Executable (PE) file would include the `mscoree.dll` DLL and the entry of `CLRCreateInstance`. EDR could treat this as malicious file by static analysis of the PE file and its import directory table.

![](/assets/images/2025-07-20/2025-07-20-Evading-EDR-PE.png)

## Assembly Output Content Detected by Memory Scanning
The output of the .NET assembly is stored in a buffer in cleartext. If EDR performs in-memory scanning of the loader process, it could identify any malicious string of the .NET assembly output.

![](/assets/images/2025-07-20/2025-07-20-Evading-EDR-Memory-Scanning.png)

# Solutions
## Loading Functions in Runtime

The solution is simple that we do not import the library and function in the PE file. Instead, we dynamically load the library and function before calling `CLRCreateInstance`.

```c
fnCLRCreateInstance CLRCreateInstance = (fnCLRCreateInstance)GetProcAddress(LoadLibraryW(L"mscoree.dll"), "CLRCreateInstance");
```

In the revised PE file, the import directory table no longer contains the malicious library and function.
![](/assets/images/2025-07-20/2025-07-20-Evading-EDR-PE-2.png)

## Obfuscation / Encoding / Encryptions

The idea behind is also easy to understand - obfuscate/encode/encrypt the malicious string into a form that EDR could not understand and determine the process is illegal.

Before obfuscation, the output is stored in plaintext and EDR could identify this is from Rubeus tool by pattern matching.
![](/assets/images/2025-07-20/2025-07-20-Evading-EDR-Memory-Scanning-2.png)

After obfuscation, it is much difficult to know the meaning of the obfuscated content below.
![](/assets/images/2025-07-20/2025-07-20-Evading-EDR-Memory-Scanning-3.png)

# Conclusion

In addition to the issues mentioned in the blog, we should be aware that other powerful EDR platforms may use alternative approaches, such as behavioral analysis, to flag the process as suspicious.

# Resources
1. https://www.r-tec.net/r-tec-blog-net-assembly-obfuscation-for-memory-scanner-evasion.html
