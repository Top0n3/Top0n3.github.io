---
title: Kerberos protocol deep explaination
date: 2025-11-06 10:00:00 +0000
categories:
- Penetration Testing
- Guides
tags:
- PenetrationTesting
- Guides
published: false
---

### Kerberos Protocol deep explaination <br> 
Kerberos protocol is widely used to implement authentication especially on Active Directory (AD) environment.
To succeed as internal penetration tester, it is crucial to understand how kerberos protocol works and how attackers can abuse it to compromise the internal AD environment.<br>
The purpose of this article is to explain how Kerberos protocol works. So in the next article, we can focus attacking kerberos authentication in AD environment.
Not that i'm not an expert in this field so this article is just an introduction to kerberos protocol and is written for beginners. feel free to reach me if you found mistake


[Introduction to Kerberos protocol](#introduction)

[Kerberos Auth Workflow](#auth-workflow)<br><br>


[Kerberos AS-REQ Request](#as-req)<br><br>
[Kerberos AS-RES Response](#as-res)<br><br>
[Kerberos TGS-REQ Request](#tgs-req)<br><br>
[Kerberos TGS-RES Request](#tgs-res)<br><br>
[Kerberos AP-REQ Request](#ap-req)<br><br>
[Kerberos AP-RES Request](#ap-res)<br><br>
[Conclusion](#conclusion)<br><br>



# Introduction to Kerberos protocol {#introduction}

In Active directory environment,  NTLM and kerberos are two protocol used to implement authentication system.
Nowdays, kerberos is the most used  because it is more secure than NTLM protocol. 
Kerberos is a protocol that allows users to authenticate on the network and access services
once authenticated. Kerberos uses port 88 by default and has been the default
authentication protocol for domain accounts since Windows 2000. When a user logs into
their PC, Kerberos is used to authenticate them. It is used whenever a user wants to access
a service on the network. Thanks to Kerberos, a user doesn't need to type their password in
constantly, and the server won't need to know every user's password. This is an example of
centralized authentication.
Kerberos is a stateless authentication protocol based on tickets. It effectively decouples a
user's credentials from their requests to consumable resources, ensuring their password is
not transmitted over the network. It is a Zero-knowledge proof protocol. The Kerberos Key
Distribution Center (KDC) does not record previous transactions; instead, the Kerberos
Ticket Granting Service ( TGS ) relies on a valid Ticket Granting Ticket ( TGT ). It
assumes that if a user has a valid TGT, they must have proven their identity.
