
# TLS für SMTP und IMAP

TLS verschlüsselt die Mailkommunikation auf allen Ebenen. In diesem Setup werden Zertifikate von **Let's Encrypt** (via Certbot) verwendet.

---

## Übersicht: Welches Zertifikat wo

| Server | Hostname | Dienst | Port |
|---|---|---|---|
| Relay-Server | `{{RELAY_HOSTNAME}}` | Postfix SMTP (eingehend/ausgehend) | 25 |
| Relay-Server | `{{RELAY_HOSTNAME}}` | Postfix Submission (Mailclients) | 587 |
| Heimserver | `{{HOME_SMTP}}` | Postfix Submission (Mailclients) | 587 |
| Heimserver | `{{HOME_IMAP}}` | Dovecot IMAP | 993 |

---

## Relay-Server

### 1. Certbot installieren

```bash
apt install certbot
```

### 2. Zertifikat ausstellen

```bash
certbot certonly --standalone -d {{RELAY_HOSTNAME}}
```

> `--standalone` startet einen temporären Webserver für die ACME-Challenge. Postfix muss dafür nicht gestoppt werden, da Port 80 genutzt wird.

Zertifikatspfade:

```
/etc/letsencrypt/live/{{RELAY_HOSTNAME}}/fullchain.pem
/etc/letsencrypt/live/{{RELAY_HOSTNAME}}/privkey.pem
```

### 3. TLS in Postfix konfigurieren

`/etc/postfix/main.cf`:

```ini
# TLS eingehend (smtpd)
smtpd_tls_cert_file = /etc/letsencrypt/live/{{RELAY_HOSTNAME}}/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/{{RELAY_HOSTNAME}}/privkey.pem
smtpd_tls_security_level = may

# TLS ausgehend (smtp)
smtp_tls_security_level = may
smtp_tls_loglevel = 1
```

### 4. Submission Port aktivieren

`/etc/postfix/master.cf`:

```
submission inet n - y - - smtpd
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_cert_file=/etc/letsencrypt/live/{{RELAY_HOSTNAME}}/fullchain.pem
  -o smtpd_tls_key_file=/etc/letsencrypt/live/{{RELAY_HOSTNAME}}/privkey.pem
```

### 5. Certbot-Hook für automatische Erneuerung

Nach der Zertifikatserneuerung muss Postfix neu geladen werden.

`/etc/letsencrypt/renewal-hooks/deploy/reload-postfix.sh`:

```bash
#!/bin/bash
systemctl reload postfix
```

```bash
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-postfix.sh
```

### 6. Postfix neu starten

```bash
systemctl restart postfix
```

---

## Heimserver

### 1. Certbot installieren

```bash
apt install certbot
```

### 2. Zertifikate ausstellen

Zwei separate Zertifikate – eines für Submission, eines für IMAP:

```bash
certbot certonly --standalone -d {{HOME_SMTP}}
certbot certonly --standalone -d {{HOME_IMAP}}
```

Zertifikatspfade:

```
/etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem   # für Postfix
/etc/letsencrypt/live/{{HOME_SMTP}}/privkey.pem
/etc/letsencrypt/live/{{HOME_IMAP}}/fullchain.pem   # für Dovecot
/etc/letsencrypt/live/{{HOME_IMAP}}/privkey.pem
```

### 3. TLS in Postfix konfigurieren

`/etc/postfix/main.cf`:

```ini
smtpd_tls_cert_file = /etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/{{HOME_SMTP}}/privkey.pem
smtpd_tls_security_level = may
smtp_tls_security_level = may
```

### 4. TLS in Dovecot konfigurieren

`/etc/dovecot/conf.d/10-ssl.conf`:

```
ssl = required
ssl_cert = </etc/letsencrypt/live/{{HOME_IMAP}}/fullchain.pem
ssl_key  = </etc/letsencrypt/live/{{HOME_IMAP}}/privkey.pem
```

### 5. Certbot-Hooks für automatische Erneuerung

Da zwei Zertifikate genutzt werden, braucht jedes seinen eigenen Hook – oder ein gemeinsamer Hook deckt beide ab:

`/etc/letsencrypt/renewal-hooks/deploy/reload-mail.sh`:

```bash
#!/bin/bash
systemctl reload postfix
systemctl reload dovecot
```

```bash
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-mail.sh
```

> Dieser Hook wird nach **jeder** Zertifikatserneuerung ausgeführt – also sowohl nach `{{HOME_SMTP}}` als auch nach `{{HOME_IMAP}}`. Das ist korrekt, da beide Dienste nach einer Erneuerung neu geladen werden müssen.

### 6. Dienste neu starten

```bash
systemctl restart postfix
systemctl restart dovecot
```

---

## Überprüfung

**Relay – SMTP TLS:**

```bash
openssl s_client -connect {{RELAY_HOSTNAME}}:25 -starttls smtp
# Zertifikat für {{RELAY_HOSTNAME}} erwartet
```

**Relay – Submission:**

```bash
openssl s_client -connect {{RELAY_HOSTNAME}}:587 -starttls smtp
```

**Heimserver – IMAP:**

```bash
openssl s_client -connect {{HOME_IMAP}}:993
# Zertifikat für {{HOME_IMAP}} erwartet
```

**Automatische Erneuerung testen:**

```bash
certbot renew --dry-run
```

---

## ✅ Ergebnis

Nach diesem Kapitel:

- SMTP auf dem Relay ist TLS-verschlüsselt (eingehend und ausgehend)
- Submission auf beiden Servern erzwingt TLS
- IMAP über Dovecot ist nur verschlüsselt erreichbar (Port 993)
- Zertifikate werden automatisch erneuert und Dienste danach neu geladen

---

## 🔁 Navigation

**← Zurück:** [DMARC konfigurieren](../03_Konfiguration/10_dmarc.md)  
**→ Weiter:** [Spamfilter & Greylisting](../03_Konfiguration/12_spamfilter.md)

