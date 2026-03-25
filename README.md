# hacking-owasp-Juice-shop
a not so detailed report showcasing the vulnerabilities found in Owasp's Juice shop by ziad.


This penetration testing report documents the vulnerabilites and the steps to finding these said vulnerabilities on Owasp's Juice shop using Burp Suite. This would be my first "proper" cybersecurity project, where i put into practice the notions i learned online such as SQL injections, XSS attacks and etc. 
This report does not cover all vulnerabilities in Owasp's Juice shop, only the ones i was able to exploit.


1. Broken Access Control
   description: Access control enforces policy such that users cannot act outside of their intended permissions. Failures typically lead to unauthorized information disclosure, modification or destruction of all data, or performing a business function outside the       user's limits.(https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/)
   This first vulnerability allowed me to access the administrators account by only knowing his email.
   I found the email because of a review left by the admin under one of the items in the store <img width="500" height="420" alt="image" src="https://github.com/user-attachments/assets/0a910f57-3ae3-438b-a8ff-50a03f0f5c3e"/>
