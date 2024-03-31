---
title: "[Hack The Box] - No-Threshold web challenge."
author: filip
date: 2024-03-31 14:10:00 +0800
categories: [Hack The Box, Challenge, Web Security, Linkedin posts]
tags: [Notes]
render_with_liquid: false
---



Yet another realistic scenario showing a problem of custom realization on MFA function. But suppose you are familiar with possible scenarios at PortSwigger Academy on the “Vulnerabilities in multi-factor authentication" topic and read the given code carefully. In that case, the exploitation won’t take much time. A good exercise for web application pentesters and application security engineers.

>To avoid such an issue in your application as in this task, ensure that your MFA implementation links a sign-in attempt to a session handler, and if the given session handler has a limited amount of attempts to enter correct credentials with MFA, before a temporary account lockout with the given options for waiting for timer or resetting MFA. 
CAPTCHA is a great addition that is welcomed as well.
{: .prompt-tip }

Also, let’s cite the **[PortSwigger](https://portswigger.net/web-security/authentication/securing)**’s recommendation:

>”Ideally, 2FA should be implemented using a dedicated device or app that generates the verification code directly. As they are purpose-built to provide security, these are typically more secure.”

Be cool, l33t, and responsible for your applications. 

{: .nolineno}
Thanks, **[Hack The Box](https://app.hackthebox.com/)** for the materials and hosting.
![Completed](assets/posts/No-Treshold.jpg){: .shadow }

> This page might be updated with a write-up in future. No guarantees ;)


#### References:
{: data-toc-skip='' .mt-3 }
- [PortSwigger MFA vulnerabilities materials](https://portswigger.net/web-security/authentication/multi-factor)
- [PortSwigger, securing authentication](https://portswigger.net/web-security/authentication/securing)
- [OWASP, General cheat sheet about MFA](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)

**#htb** **#hackthebox** **#appsec** **#whitebox** **#codereview** **#hacking** **#mfa** **#webpentesting** **#challenge** **#cybersecurity** **#ethicalhacking**