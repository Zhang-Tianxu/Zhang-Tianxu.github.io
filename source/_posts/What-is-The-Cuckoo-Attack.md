---
title: What is The Cuckoo Attack
date: 2020-06-24 09:43:32
tags:
    - Attacks
    - TEE
categories:
    - Security
    - TEE
    - Attacks
---

# Introduction

The cuckoo bird replaces other birds’ eggs with its own. The victim birds are tricked into feeding the cuckoo chick as if it were their own.Similarly, in a cuckoo attack, the attacker “replaces” the victim’s [TEE](https://en.wikipedia.org/wiki/Trusted_execution_environment) with his own TEE, leading the victim to treat the attacker’s TEE as his own.

Specifically, **the attacker try to convinces the victim that a TEE physically controled by the adversary resides in the victim's own local computer**.

How it can be achieved?

In most TEE's protocal, attestation for TEE is achieved by special TEE, the special TEE only certifies that the code is running on *some* genuine TEE processor, but it does not guarantee that where is the processor actually located. Adversary can take advantage of this shortcoming to achieve cuckoo attack.

In the following picture, there is a malware on the victim's computer, it proxies all messages between user and TEE in his machine. When user want to attestation a TEE on his own computer, the malware intercept this message and forward to TEE on attacker's machine, and then a special TEE on attacker's computer generate a proof to prove the TEE on attacker's machine is an authentic TEE.

As a result, the victim are convinced that the TEE on attacker's machine resides in his own local computer, and trust it. In this case, Any secrets victim enters can be forwarded to attacker's TEE, attacker can steal the secrets by some means (such as phycial means).

![image-20200624101517944](https://tva1.sinaimg.cn/large/007S8ZIlly1gg35tanyf3j31wk0u0dk5.jpg)


# References
1. Parno, Bryan. "Bootstrapping Trust in a" Trusted" Platform." HotSec. 2008.
