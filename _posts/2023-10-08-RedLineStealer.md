---
layout: post
title: "RedLineStealer Analysis"
summary: "Analysis of information stealer malware that inserts malware into system processes"
author: bradypus404
date: '2023-10-08 00:50:01 +0900'
category: ['malware']
tags: RedLineStealer, Malware
thumbnail: /assets/img/posts/RedLineStealer/Cover.png
keywords: malware, reversing
usemathjax: false
permalink: /blog/Malware/6be57566a72c81a9336d39b56627c14aa6a04e604954b71a84e83125171a742c
---

# Title: Malware Analysis RedLineStealer
 Writter: foliv0ra, eveheeero

- SHA256 hash: 6be57566a72c81a9336d39b56627c14aa6a04e604954b71a84e83125171a742c

- Family: RedLineStealer

- First seen: 2023-09-23 22:00:12 UTC

- Environment: Windows11 Sandbox


- Used Tool
    1. DIE
    2. x64dbg
    3. Process Explorer

## {Analysis OverView}

Based on VirusTotal, It was uploaded at 2023-09-23 20:58:42 UTC. The Malware turned out to be RedLineStealer, which steals personal information such as user Emails, Web browsers, and Cyptocurrency.

File Type: EXE

![Alt text](/assets/img/posts/RedLineStealer/image-2.png){: width="100%" height="100%"}

## {Static Analysis}
### DIE(Detect It Easy)

![Alt text](/assets/img/posts/RedLineStealer/image.png){: width="100%" height="100%"}

- Compiler: Microsoft Visaul C/C++(2017 v.15.5-6)[EXE32], Microsoft Visaul C/C++(2022 v.17.4)[-]
- Not Packing
But, When I opened the binary with debbugger, my thoughts changed.

![Alt text](/assets/img/posts/RedLineStealer/image-1.png){: width="100%" height="100%"}

- Entropy

## {Summary}

The code inside the Malware is decrypted and injected into Microsoft.NET's AppLaunch.exe, and AppLaunch.exe is created and executed. There is no code to steal personal information in this malware, ant it is assumed that the code injected into AppLaunch.exe contains code to steal personal information.

## {Dynamic Analysis}
### x64dbg

1. The front loop is not related to the main function, so it is skipped, and the place where BP is placed is the main function.

![Alt text](/assets/img/posts/RedLineStealer/image-3.png){: width="100%" height="100%"}

2. In this part, obfuscation techniques are used. If you look closely at the Operand of the jump command, it points to an address that is different from the address value processed in the debugger.

![Alt text](/assets/img/posts/RedLineStealer/image-4.png){: width="100%" height="100%"}


3. When you jump to that address, it is converted to the code below and a new jump is created.

![Alt text](/assets/img/posts/RedLineStealer/image-5.png){: width="100%" height="100%"}


4. Decrypt the values in the encrypted memory dump using xor and add.

![Alt text](/assets/img/posts/RedLineStealer/image-11.png){: width="100%" height="100%"}


5. If you run it dynamically and perform decryption, you can see that an EXE (header: 4D 5A) file like the memory has been created.

![Alt text](/assets/img/posts/RedLineStealer/image-12.png){: width="100%" height="100%"}


6. In the createprocessW API, the 6th Argument is saved as 4, which is the argument value that tells AppLaunch.exe to run and pause. The binary value created here is later injected into AppLaunch.exe.

![dbg CreateprocessW](/assets/img/posts/RedLineStealer/image-8.png){: width="100%" height="100%"}
![Alt text](/assets/img/posts/RedLineStealer/image-10.png){: width="100%" height="100%"}

% Reference link: https://learn.microsoft.com/ko-kr/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw

![CreateprocessW](/assets/img/posts/RedLineStealer/image-7.png){: width="100%" height="100%"}
![Alt text](/assets/img/posts/RedLineStealer/image-9.png){: width="100%" height="100%"}


7. Execute the NTWriteVirtualMemory function and insert the code in the memory into AppLaunch.exe as mentioned above.

![Alt text](/assets/img/posts/RedLineStealer/image-13.png){: width="100%" height="100%"}


8. As a result of monitoring using Process Explorer, it was confirmed that AppLaunch.exe, which had been inserted with code presumed to be malicious code, was executed and immediately paused as mentioned above.

![Alt text](/assets/img/posts/RedLineStealer/image-14.png){: width="100%" height="100%"}


In the next task, we analyze AppLaunch.exe, which has malicious actions inserted into it. :)