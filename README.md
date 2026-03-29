# hacking-owasp-Juice-shop
This report outlines the security vulnerabilities identified within the OWASP Juice Shop application during a penetration testing session.

## Overview

OWASP Juice Shop is an intentionally insecure web application designed for security training. It has over 100 vulnerabilities that can be exploited and can be run on localhost:3000. I used Docker to run it locally.

<img width="1916" height="1079" alt="image" src="https://github.com/user-attachments/assets/c92c5f4e-79a0-442b-876a-b6b0af1224ba" />


This penetration testing report documents the vulnerabilities I found and the steps I took to exploit them using Burp Suite. This is my first "proper" cybersecurity project where I put into practice the concepts I learned online, such as SQL injections, XSS attacks, and more. This report doesn't cover every single vulnerability in the shop—only the ones I was actually able to exploit.

## 1. Broken Access Control (SQL injection)

**Description:**
Access control enforces policy such that users cannot act outside of their intended permissions. Failures typically lead to unauthorized information disclosure, modification or destruction of all data, or performing a business function outside the user's limits.(https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/
)

This vulnerability allowed for an Authentication Bypass, granting access to the administrator account using only a known email address.

I identified the admin's email through a public review left on a product page:
<img width="500" height="420" alt="image" src="https://github.com/user-attachments/assets/0a910f57-3ae3-438b-a8ff-50a03f0f5c3e"/>

After finding it I navigated to the login page and used Burp Suite’s Intruder tool to intercept the request and automate a customized attack against the web application by inserting multiple payloads. I tested several payloads, such as ' OR 1=1 --, designed to manipulate the backend query logic to return a "true" value while commenting out the password verification.
<img width="975" height="522" alt="image" src="https://github.com/user-attachments/assets/1c639b62-d733-4c8b-9b28-a1fa5ea698fa" />

By injecting the payload admin@juice-sh.op
'-- into the email field, I successfully bypassed the login requirements, this immediately returned an authentication response with a length that was far above the usual error responses which raised a red flag for me, by accessing the response I found out that The server had given me access and returned the token for the administrator account :
<img width="613" height="428" alt="image" src="https://github.com/user-attachments/assets/86a4c591-ef84-4c85-9ac9-c818a759529b" />

I used the browser’s Developer Tools (F12) to access the local Storage and manually injected the stolen token. After refreshing the page, I was granted full administrative privileges:

<img width="740" height="612" alt="image" src="https://github.com/user-attachments/assets/aed6145e-aa95-4d32-8f24-011c1ffa4a52" />

This exploit confirms that the application relies solely on the client-side token for session management.

## 2. Cross-Site Scripting (XSS)

**Description:**
XSS attacks are a type of injection, in which malicious scripts are injected into otherwise benign and trusted websites. XSS attacks occur when an attacker uses a web application to send malicious code, generally in the form of a browser side script, to a different end user. (https://owasp.org/www-community/attacks/xss/
)

After understanding how lacking the structure of the website was, i decided to start exploring all input fields for other sorts of injections, this is the moment where I identified a lack of Input Sanitization in the search bar. The application fails to verify or encode input before rendering it in the UI, therefore executing whatever code is entered. The objective here was to insert a javascript alert into the website.

<img width="213" height="52" alt="image" src="https://github.com/user-attachments/assets/7f959ad7-9786-4850-ba2d-f3a3f8a29110" />

By searching up <blabla> blabla i could see that that the HTML tags were rendered directly in the code rather than being escaped:

<img width="627" height="83" alt="image" src="https://github.com/user-attachments/assets/53301f86-0544-4287-bd0e-629736ea8226" />

knowing this I injected a basic JavaScript alert payload. The script executed immediately with no errors returned.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/ab8c361c-ae4d-484b-9823-9af4566dd712" />

This DOM XSS attack allows an attacker to steal session cookies, redirect users to malicious sites, or run malicious scripts for other users.

## 3. Cryptographic Failures & Reverse Engineering

**Description:**
Cryptographic failure refers to the inadequate protection of sensitive data. Reverse engineering is the process of deconstructing software to understand its internal logic and exploit hidden vulnerabilities.

The objective was to create a new coupon code by understanding how the coupons were generated through reverse engineering.

I discovered that the websites chatbot seemed to have an interesting personality, it could be manipulated into revealing a 10% discount code if you begged it hard enough:
<img width="975" height="209" alt="image" src="https://github.com/user-attachments/assets/f79fcba2-6950-4216-b8eb-ca2078b29e80" />

After finding this first coupon i inserted the coupon into many decoders online. Recalling earlier source code where the 'z85' library was referenced, I successfully identified the string as being encoded in Z85(ASCII85).

<img width="351" height="271" alt="image" src="https://github.com/user-attachments/assets/633d83e5-8661-4f8a-a29b-d3c1e7782a1e" />


Decoding the string revealed a specific hexadecimal pattern:
4d 41 52 32 36 2d 31 30
Converting this to ASCII formulated:

```
4d = M
41 = A
52 = R
32 = 2
36 = 6
2d = -
31 = 1
30 = 0
```

Which shows a [MONTH][YEAR]-[DISCOUNT] format, as the previous code was a 10% off and ended with -10 i was able to speculate that the last numbers are the % discount. Using this template, I drafted a custom string for an 80% discount (MAR26-80) and re-encoded it to produce the new coupon: o*IVjhz3)x
<img width="1799" height="351" alt="image" src="https://github.com/user-attachments/assets/37be9fa5-3c3e-416d-a8b8-8057e9eaa9d9" />

The forged coupon was successfully validated by the application, reducing the total order cost by 80%:
<img width="609" height="72" alt="image" src="https://github.com/user-attachments/assets/e18aa312-2bd9-400a-99ad-a55aa628b87c" />

This exploit allows unauthorized discounts, leading to significant financial loss for the shop.

## 4. Parameter Tampering

**Description:**
Parameter tampering involves manipulating the data exchanged between a client and a server to modify application behavior or data.

The objective here was to manipulate the quantity of items in the store, by tricking the server.
Using Burp Suite's proxy interceptor I analyzed the communication between the client and server during the checkout process. For example when the client wanted to request more "apple juices" it would send a GET request and the server would respond with a POST by incrementing or decrementing the value of apple juices, but I observed that the application relies heavily on client-side logic for order quantities, allowing me to intercept and modify the "Quantity" parameter before it reached the backend.
<img width="975" height="550" alt="image" src="https://github.com/user-attachments/assets/9781077e-7ce2-46bc-8edd-ef8cab33e0a7" />

By tampering with the request and changing the quantity to a negative value I successfuly made the company charge me a negative balance:
<img width="975" height="549" alt="image" src="https://github.com/user-attachments/assets/e4d8a2b0-fed9-48d0-958d-71a2848e06c5" />

This vulnerability allows an attacker to obtain products for free or manipulate the store's financial records and inventory levels.


## 5.Security through Obscurity

**Description:**
Obfuscation means to make something difficult to understand. Programming code is often obfuscated to protect intellectual property or trade secrets, and to prevent an attacker from reverse engineering a proprietary software program.(https://www.techtarget.com/searchsecurity/definition/obfuscation)

This exploit involved discovering a hidden page that was not intended for user access. I first noticed the possibility of a hidden URL while reading through all the paths in the main javascript file, where I noticed two specific matchers labeled t$ and e$. The t$ matcher, in particular, stood out.

<img width="658" height="529" alt="image" src="https://github.com/user-attachments/assets/3178c799-955c-4940-80ac-8d39ad01510e" />

Following this discovery, I scanned the main.js file for other instances of t$ and found its definition:
```
function t$(t) {
    return t.length === 0 ? null : t[0].toString().match(i$(25, 184, 174, 179, 182, 186) + "sal".toLowerCase() + a$(13, 144, 87, 152, 139, 144, 83, 138) + "a".toLowerCase()) ? {
        consumed: t
    } : null
}
```

This function seemed to return something into the path, this something seemed to be a string that was made using the output of an i$ function, the word "sal", the a$ function and the letter "a" all in lowercase. After attempting to decode the functions myself i found it to be pretty hard and time inducing to attempt to brute force this code, as the logic relies on an algorithm where an array of integers is reversed and transformed into ASCII characters using the "key" provided as the first argument. Therefore I opted to redefine the i$, a$, and t$ functions directly within the browser console to observe their output:
```
function i$(...t) {
    let n = Array.prototype.slice.call(t), e = n.shift();
    return n.reverse().map(function(i, a) {
        return String.fromCharCode(i - e - 45 - a)
    }).join("")
}

function a$(...t) {
    let n = Array.prototype.slice.call(t), e = n.shift();
    return n.reverse().map(function(i, a) {
        return String.fromCharCode(i - e - 45 - a)
    }).join("")
}
```

After a few attemps in the console I successfully bypassed the obfuscation and obtained the hidden URL: tokensale-ico-ea.
<img width="975" height="780" alt="image" src="https://github.com/user-attachments/assets/6eef4623-d343-4d22-b2a8-189f2b3fa854" />
<img width="975" height="549" alt="image" src="https://github.com/user-attachments/assets/568b2870-f66c-4811-915a-fb63975a772c" />

The reliance on client-side obfuscation provides a false sense of security, this weakly protected and poorly obfuscated code presents a security risk, as it allows users to discover and access internal paths that should remain restricted or hidden.

## 6.Metadata Derived Security Question Bypass











