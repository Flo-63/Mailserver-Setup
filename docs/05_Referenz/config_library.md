
# Config Library

Diese Datei enthält alle relevanten Konfigurationsdateien und Scripts des Setups. Platzhalter wie `{{DOMAIN}}` sind in `[Variablen und Platzhalter](../00_Einleitung/00_variablen.md)` definiert.

Sensible Werte wie Passwörter sind durch `{{SECRET_...}}` Platzhalter ersetzt.

---

## Heimserver

### `/etc/postfix/main.cf`

```ini
# Basisinfos
myhostname = {{HOME_SMTP}}
myorigin = /etc/mailname
mydestination = $myhostname, localhost.$mydomain, localhost
local_recipient_maps =

# Virtuelle Mailboxen via PostfixAdmin/MySQL
mailbox_transport = dovecot
virtual_alias_maps = mysql:/etc/postfix/mysql_virtual_alias_maps.cf
virtual_mailbox_domains = mysql:/etc/postfix/mysql_virtual_domains_maps.cf
virtual_mailbox_maps = mysql:/etc/postfix/mysql_virtual_mailbox_maps.cf
virtual_mailbox_base = /var/vmail
virtual_mailbox_limit = 0
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

inet_interfaces = all
inet_protocols = ipv4

# Mailweiterleitung über Relay (Port 587 mit SASL)
relayhost = [{{RELAY_HOSTNAME}}]:587
smtp_use_tls = yes
smtp_tls_security_level = encrypt
smtp_tls_CApath = /etc/letsencrypt/live/{{HOME_SMTP}}
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous

# Erlaubte Absender (LAN + Relay-IP fest)
mynetworks = 127.0.0.0/8 192.168.1.0/24 {{RELAY_IP}}

# SASL Authentifizierung über Dovecot (für Mailclients)
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth_dovecot
smtpd_sasl_auth_enable = yes

# TLS für eingehende Verbindungen
smtpd_tls_cert_file = /etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/{{HOME_SMTP}}/privkey.pem
smtpd_tls_CAfile = /etc/letsencrypt/live/{{HOME_SMTP}}/chain.pem
smtpd_tls_security_level = may
smtpd_tls_mandatory_protocols = TLSv1.2 TLSv1.3
smtpd_tls_mandatory_ciphers = high
smtpd_tls_exclude_ciphers = aNULL, MD5
smtpd_tls_dh1024_param_file = /etc/ssl/certs/dh4096.pem
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# Empfangsregeln
smtpd_recipient_restrictions =
    permit_sasl_authenticated,
    permit_mynetworks,
    reject_unauth_destination

# Keine lokalen Filterdienste
content_filter =
receive_override_options =
mailbox_command =
mailbox_size_limit = 0

# OpenDKIM Milter
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:12301
non_smtpd_milters = $smtpd_milters

# Allgemein
message_size_limit = 52428800
recipient_delimiter = +
compatibility_level = 3.6
```

---

### `/etc/postfix/master.cf`

```
smtp       inet  n       -       y       -       -       smtpd
  -o smtpd_tls_security_level=may
  -o smtpd_recipient_restrictions=permit_mynetworks,reject

smtps      inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_cert_file=/etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem
  -o smtpd_tls_key_file=/etc/letsencrypt/live/{{HOME_SMTP}}/privkey.pem
  -o smtpd_tls_CAfile=/etc/letsencrypt/live/{{HOME_SMTP}}/chain.pem
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

2525      inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/2525
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

pickup     fifo  n       -       y       60      1       pickup
  -o content_filter=
  -o receive_override_options=no_header_body_checks
cleanup    unix  n       -       y       -       0       cleanup
qmgr       fifo  n       -       n       300     1       qmgr
tlsmgr     unix  -       -       y       1000?   1       tlsmgr
rewrite    unix  -       -       y       -       -       trivial-rewrite
bounce     unix  -       -       y       -       0       bounce
defer      unix  -       -       y       -       0       bounce
trace      unix  -       -       y       -       0       bounce
verify     unix  -       -       y       -       1       verify
flush      unix  n       -       y       1000?   0       flush
proxymap   unix  -       -       n       -       -       proxymap
proxywrite unix  -       -       n       -       1       proxymap
smtp       unix  -       -       y       -       -       smtp
relay      unix  -       -       y       -       -       smtp
showq      unix  n       -       y       -       -       showq
error      unix  -       -       y       -       -       error
retry      unix  -       -       y       -       -       error
discard    unix  -       -       y       -       -       discard
local      unix  -       n       n       -       -       local
virtual    unix  -       n       n       -       -       virtual
lmtp       unix  -       -       y       -       -       lmtp
anvil      unix  -       -       y       -       1       anvil
scache     unix  -       -       y       -       1       scache

dovecot    unix  -       n       n       -       -       pipe
  flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -d ${recipient}
```

---

### `/etc/postfix/sasl_passwd`

```
[{{RELAY_HOSTNAME}}]:587    {{RELAY_SASL_USER}}:{{SECRET_RELAY_SASL_PASSWORD}}
```

> Nach Änderung: `postmap hash:/etc/postfix/sasl_passwd && chmod 600 /etc/postfix/sasl_passwd`

---

### `/etc/postfix/mysql_virtual_domains_maps.cf`

```ini
hosts = 127.0.0.1
user = postfixadmin
password = {{SECRET_DB_PASSWORD}}
dbname = postfixadmin
query = SELECT domain FROM domain WHERE domain='%s' AND active = '1'
```

---

### `/etc/postfix/mysql_virtual_mailbox_maps.cf`

```ini
hosts = 127.0.0.1
user = postfixadmin
password = {{SECRET_DB_PASSWORD}}
dbname = postfixadmin
query = SELECT maildir FROM mailbox WHERE username='%s' AND active = '1'
```

---

### `/etc/postfix/mysql_virtual_alias_maps.cf`

```ini
hosts = 127.0.0.1
user = postfixadmin
password = {{SECRET_DB_PASSWORD}}
dbname = postfixadmin
query = SELECT goto FROM alias WHERE address='%s' AND active = '1'
```

---

### `/etc/postfix/mysql_sender_login_maps.cf`

```ini
hosts = 127.0.0.1
user = postfixadmin
password = {{SECRET_DB_PASSWORD}}
dbname = postfixadmin
query = SELECT username AS allowedUser FROM mailbox WHERE username='%s' AND active = 1
        UNION SELECT goto FROM alias WHERE address='%s' AND active = 1
```

---

### `/etc/postfix/mysql_virtual_relayhosts_maps.cf`

```ini
hosts = 127.0.0.1
user = postfixadmin
password = {{SECRET_DB_PASSWORD}}
dbname = postfixadmin
query = SELECT description FROM domain WHERE domain='%d' AND active = '1'
```

---

### `/etc/opendkim.conf`

```
Syslog                  yes
UMask                   002
Canonicalization        relaxed/simple
Mode                    sv
AutoRestart             yes
AutoRestartRate         10/1h
Background              yes
DNSTimeout              5
SignatureAlgorithm      rsa-sha256

KeyTable                refile:/etc/opendkim/key.table
SigningTable            refile:/etc/opendkim/signing.table
ExternalIgnoreList      refile:/etc/opendkim/trusted.hosts
InternalHosts           refile:/etc/opendkim/trusted.hosts

Socket                  inet:12301@localhost
UserID                  opendkim:opendkim
PidFile                 /run/opendkim/opendkim.pid
```

---

### `/etc/opendkim/key.table`

```
{{DKIM_SELECTOR}}._domainkey.{{DOMAIN}}    {{DOMAIN}}:{{DKIM_SELECTOR}}:/etc/opendkim/keys/{{DOMAIN}}/{{DKIM_SELECTOR}}.private
```

---

### `/etc/opendkim/signing.table`

```
*@{{DOMAIN}}    {{DKIM_SELECTOR}}._domainkey.{{DOMAIN}}
```

---

### `/etc/opendkim/trusted.hosts`

```
127.0.0.1
localhost
::1
*.{{DOMAIN}}
{{HOME_HOSTNAME}}.ddns.net
```

> `{{HOME_HOSTNAME}}.ddns.net` ist der No-IP-DynDNS-Name des Heimservers. Er wird parallel zu `{{HOME_SMTP}}` (A-Record via deSEC) gepflegt.

---

### Certbot Hooks

**Pre-Hook** `/etc/letsencrypt/renewal-hooks/pre/001-open-ports.sh`:

```bash
#!/bin/bash
echo "[Certbot pre-hook] Öffne Ports 80 und 443 für Let's Encrypt Validierung..."
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT
```

**Post-Hook** `/etc/letsencrypt/renewal-hooks/post/001-close-ports.sh`:

```bash
#!/bin/bash
echo "[Certbot post-hook] Entferne temporäre Portfreigaben für 80 und 443..."
iptables -D INPUT -p tcp --dport 80 -j ACCEPT
iptables -D INPUT -p tcp --dport 443 -j ACCEPT
```

**Deploy-Hook** `/etc/letsencrypt/renewal-hooks/deploy/001-dovecot-apache-restart.sh`:

```bash
#!/bin/bash
echo "ssl certs updated in dovecot" && service dovecot restart
echo "ssl certs updated in apache" && service apache2 restart

# Zertifikate in Postfix-Chroot aktualisieren
cp /etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem /var/spool/postfix/etc/letsencrypt/live/{{HOME_SMTP}}/
cp /etc/letsencrypt/live/{{HOME_SMTP}}/chain.pem /var/spool/postfix/etc/letsencrypt/live/{{HOME_SMTP}}/
cp /etc/letsencrypt/live/{{HOME_SMTP}}/cert.pem /var/spool/postfix/etc/letsencrypt/live/{{HOME_SMTP}}/
cp /etc/letsencrypt/live/{{HOME_SMTP}}/privkey.pem /var/spool/postfix/etc/letsencrypt/live/{{HOME_SMTP}}/
cp /etc/hosts /var/spool/postfix/etc/hosts
echo "ssl certs updated in postfix chroot" && postfix reload
```

**Deploy-Hook** `/etc/letsencrypt/renewal-hooks/deploy/update-tlsa.sh`:

```bash
#!/bin/bash
TLSA_HASH=$(openssl x509 -in /etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem -noout -pubkey \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | xxd -p -c 64)

curl -X PATCH https://desec.io/api/v1/domains/{{DOMAIN}}/rrsets/_465._tcp.smtp/TLSA/ \
  -H "Authorization: Token $(cat /root/.dedyn-token)" \
  -H "Content-Type: application/json" \
  -d "{\"records\": [\"3 1 1 $TLSA_HASH\"]}"
```

> Dieser Hook aktualisiert den TLSA-Record für DANE nach jeder Zertifikatserneuerung. Siehe [TLS für IMAP und SMTP](../03_Konfiguration/11_tls_imap_smtp.md).

---

### `/usr/local/bin/update-dedyn.sh`

```bash
#!/bin/bash
TOKEN_FILE="/root/.dedyn-token"
LOG_FILE="/var/log/dedyn-update.log"
IP_FILE="/var/lib/dedyn/last_ip"
DOMAIN="{{DOMAIN}}"
TOKEN=$(cat "$TOKEN_FILE")

mkdir -p /var/lib/dedyn

CURRENT_IP=$(curl -s --max-time 10 https://ifconfig.me)
if [ -z "$CURRENT_IP" ]( -z "$CURRENT_IP" .md); then
  echo "$(date) - ❌ Fehler: Konnte IP nicht ermitteln." >> "$LOG_FILE"
  exit 1
fi

LAST_IP=$(cat "$IP_FILE" 2>/dev/null || echo "")

if [ "$CURRENT_IP" != "$LAST_IP" ]( "$CURRENT_IP" != "$LAST_IP" .md); then
  update_record() {
    local RR_NAME="$1"
    local RRSET_URL="https://desec.io/api/v1/domains/${DOMAIN}/rrsets/${RR_NAME}/A"
    local RESPONSE=$(curl -s --max-time 15 --retry 3 --retry-delay 5 \
      -X PATCH "$RRSET_URL/" \
      -H "Authorization: Token $TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"records\": [\"$CURRENT_IP\"]}")
    if echo "$RESPONSE" | grep -q "$CURRENT_IP"; then
      echo "$(date) - ✅ Updated ${RR_NAME}.${DOMAIN} to $CURRENT_IP" >> "$LOG_FILE"
    else
      echo "$(date) - ❌ Fehler beim Update ${RR_NAME}.${DOMAIN} – Antwort: $RESPONSE" >> "$LOG_FILE"
    fi
  }

  update_record "smtp"   # {{HOME_SMTP}}
  update_record "%40"    # @ = Root-Domain ({{DOMAIN}})
else
  echo "$(date) - ℹ️ IP unverändert: $CURRENT_IP" >> "$LOG_FILE"
fi

echo "$CURRENT_IP" > "$IP_FILE"
```

---

### `/etc/dovecot/dovecot.conf`

```
auth_mechanisms = plain login
log_timestamp = "%Y-%m-%d %H:%M:%S "

passdb {
  args = /etc/dovecot/dovecot-mysql.conf
  driver = sql
}

userdb {
  args = /etc/dovecot/dovecot-mysql.conf
  driver = sql
}

plugin {
  sieve_before = /var/vmail/sieve/spam-global.sieve
  sieve_dir = /var/vmail/%d/%n/sieve/scripts/
  sieve = /var/vmail/%d/%n/sieve/active-script.sieve
}

protocols = imap pop3 sieve

service auth {
  unix_listener /var/spool/postfix/private/auth_dovecot {
    group = postfix
    mode = 0660
    user = postfix
  }
  unix_listener auth-master {
    mode = 0600
    user = vmail
  }
  user = root
}

service managesieve-login {
  executable = /usr/lib/dovecot/managesieve-login
}
service managesieve {
  executable = /usr/lib/dovecot/managesieve
}

ssl = required
ssl_cert = </etc/letsencrypt/live/{{HOME_IMAP}}/fullchain.pem
ssl_key = </etc/letsencrypt/live/{{HOME_IMAP}}/privkey.pem
ssl_dh = </etc/dovecot/dh.pem

protocol pop3 {
  pop3_uidl_format = %08Xu%08Xv
}

protocol lda {
  auth_socket_path = /var/run/dovecot/auth-master
  mail_plugin_dir = /usr/lib/dovecot/modules
  mail_plugins = sieve
  postmaster_address = {{ADMIN_MAIL}}
}

protocol sieve {
  managesieve_implementation_string = dovecot
  managesieve_logout_format = bytes=%i/%o
  managesieve_max_line_length = 65536
}

mail_home = /var/vmail/%d/%n
ssl_dh = </etc/dovecot/dh.pem

!include /etc/dovecot/conf.d/*.conf
```

> **Hinweis:** `ssl_dh` ist in dieser Datei doppelt definiert (Zeile ~15 und ~25). Dovecot verwendet den letzten Wert – das ist identisch, hat also keine Auswirkung. Beim nächsten Bearbeiten der Datei kann das Duplikat entfernt werden.

---

### `/etc/dovecot/dovecot-mysql.conf`

```ini
driver = mysql
connect = host=localhost dbname=postfixadmin user=postfixadmin password={{SECRET_DB_PASSWORD}}
default_pass_scheme = PLAIN-MD5
password_query = SELECT password FROM mailbox WHERE username = '%u'
user_query = SELECT CONCAT('maildir:/var/vmail/',maildir) AS mail, 5000 AS uid, 5000 AS gid FROM mailbox WHERE username = '%u'
```

> Die `user_query` liefert den vollständigen Maildir-Pfad direkt aus der PostfixAdmin-Datenbank. `mail_location` in `10-mail.conf` wird dadurch überschrieben.

---

### `/etc/dovecot/conf.d/10-master.conf` (relevante Abschnitte)

```
default_internal_user = vmail

service imap-login {
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service auth {
  unix_listener /var/spool/postfix/private/auth_dovecot {
    mode = 0660
    user = postfix
    group = postfix
  }
  unix_listener auth-userdb {
    mode = 0600
    user = vmail
    group = vmail
  }
  user = root
}
```

---

### `/etc/dovecot/conf.d/10-auth.conf` (relevante Abschnitte)

```
auth_mechanisms = plain

# Passwdfile für einzelne Admin-User (Fallback)
!include auth-passwdfile.conf.ext

# SQL-Backend wird in dovecot.conf direkt definiert und hat Vorrang
#!include auth-sql.conf.ext
```

---

### `/etc/dovecot/conf.d/10-ssl.conf`

TLS wird in `dovecot.conf` direkt konfiguriert – diese Datei enthält nur den Standardwert:

```
ssl = required
```

---

### DH-Parameter erzeugen

Dovecot und Postfix nutzen separate DH-Dateien:

```bash
# Für Dovecot
openssl dhparam -out /etc/dovecot/dh.pem 4096

# Für Postfix
openssl dhparam -out /etc/ssl/certs/dh4096.pem 4096
```

> Beide Befehle dauern mehrere Minuten. Sie können parallel auf zwei Terminals ausgeführt werden.

---

## Relay-Server

### `/etc/postfix/main.cf`

```ini
smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
message_size_limit = 52428800

# TLS
smtpd_tls_cert_file = /etc/letsencrypt/live/{{RELAY_HOSTNAME}}/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/{{RELAY_HOSTNAME}}/privkey.pem
smtpd_tls_security_level = may
smtp_tls_security_level = may
smtp_tls_CApath = /etc/ssl/certs
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_tls_auth_only = yes
smtpd_use_tls = yes

# SASL (Dovecot) – für Mailclients und Heimserver
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous

# Empfangsregeln
smtpd_recipient_restrictions =
    permit_sasl_authenticated,
    permit_mynetworks,
    reject_unauth_destination

# Netzwerke – Relay-IP ist vertrauenswürdig für Port 2525
mynetworks = 127.0.0.0/8 [::1]/128 {{RELAY_IP}}

# Hostname
myhostname = {{RELAY_HOSTNAME}}
myorigin = /etc/mailname
mydestination = localhost
relay_domains = {{DOMAIN}}
transport_maps = hash:/etc/postfix/transport

# kein relayhost – ausgehende Mails direkt ins Internet
relayhost =

# Sonstiges
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = ipv4
compatibility_level = 3.6
readme_directory = no
disable_vrfy_command = yes

# HELO-Prüfungen
smtpd_helo_required = yes
smtpd_helo_restrictions =
    permit_mynetworks,
    reject_non_fqdn_helo_hostname,
    reject_invalid_helo_hostname,
    reject_unknown_helo_hostname,
    permit

# Daten-Prüfungen
smtpd_data_restrictions =
    reject_unauth_pipelining,
    reject_multi_recipient_bounce,
    permit

# RBL-Prüfungen
smtpd_client_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    check_client_access hash:/etc/postfix/rbl_override,
    reject_rbl_client dnsbl.sorbs.net,
    reject_rbl_client bl.spamcop.net,
    reject_rbl_client b.barracudacentral.org,
    reject_rbl_client psbl.surriel.com,
    permit

# Rspamd Milter (Port 11332 – nicht 12301, der ist von OpenDKIM auf dem Heimserver belegt)
milter_protocol = 6
milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
smtpd_milters = inet:localhost:11332
non_smtpd_milters = inet:localhost:11332
milter_default_action = accept
```

---

### `/etc/postfix/transport`

```
{{DOMAIN}}    smtp:[{{HOME_SMTP}}]:2525
```

---

### `/etc/postfix/sasl_passwd`

Leer – der Relay sendet ausgehende Mails direkt ins Internet ohne Smarthost.

---

### `/etc/rspamd/local.d/worker-proxy.inc`

```
# Milter-Dienst für Postfix
milter = true;

# Port 11332 – nicht 12301 (von OpenDKIM auf dem Heimserver belegt)
bind_socket = "127.0.0.1:11332";

milter_mail_macros = "i, mail_addr, rcpt_addr, from_addr, rcpt_user, rcpt_domain";
```

---

### `/etc/rspamd/local.d/worker-controller.inc`

```
password = "{{SECRET_RSPAMD_CONTROLLER_PASSWORD}}";
bind_socket = "127.0.0.1:11334";
```

---

### `/etc/rspamd/local.d/dkim_signing.conf`

```
domain {
  {{DOMAIN}} {
    selector = "{{DKIM_SELECTOR}}";
    path = "/etc/rspamd/dkim/{{DKIM_SELECTOR}}.{{DOMAIN}}.key";
  }
}
sign_local = true;
sign_authenticated = true;
use_domain = "header";
```

---

### `/etc/rspamd/local.d/antivirus.conf`

```
clamav {
  type = "clamav";
  symbol = "CLAM_VIRUS";
  servers = "/var/run/clamav/clamd.ctl";
  action = "quarantine";
}
```

---

### `/etc/rspamd/local.d/spf.conf`

```
enabled = true;
auth_and_local = true;
```

---

### `/etc/rspamd/local.d/actions.conf`

```
reject = 12;
add_header = 6;
greylist = 4;
```

---

### `/etc/rspamd/local.d/history_redis.conf`

```
servers = "127.0.0.1";
```

---

### `/etc/rspamd/local.d/options.inc`

```
quarantine {
  quarantine_dir = "/var/lib/rspamd/quarantine";
}
```

---

### `/etc/rspamd/local.d/neural.conf`

```
enabled = false;
```

---

### `/etc/systemd/system/rspamd.service.d/override.conf`

```ini
[Service]
User=_rspamd
Group=_rspamd
```

> **Wichtig:** Datei muss mit Newline enden – fehlendes Newline verhindert den Start.

---

### `/etc/tmpfiles.d/rspamd.conf`

```
d /run/rspamd 0755 _rspamd _rspamd -
```

> Stellt sicher dass `/run/rspamd/` nach Reboots automatisch angelegt wird.

---

---

### `/etc/iptables/drop_tor.sh`

Blockt Tor-Exit-Nodes per iptables. Wird manuell oder per Cronjob ausgeführt – die Regeln überleben keinen Reboot (nicht in `rules.v4` gespeichert).

```bash
#!/bin/bash
# Block Tor Exit nodes
IPTABLES_TARGET="DROP"
IPTABLES_CHAINNAME="TOR"

if ! iptables -L TOR -n >/dev/null 2>&1; then
  iptables -N TOR >/dev/null 2>&1
  iptables -A INPUT -p tcp -j TOR 2>&1
fi

cd /tmp/
wget -q -O - "https://www.dan.me.uk/torlist/" -U SXTorBlocker/1.0 > /tmp/full.tor
sed -i 's|^#.*$||g' /tmp/full.tor
iptables -F TOR

CMD=$(cat /tmp/full.tor | uniq | sort)
for IP in $CMD; do
  iptables -A TOR -s $IP -j DROP
done

iptables -A TOR -j RETURN
rm /tmp/full.tor
```

Empfohlener Cronjob (`/etc/cron.d/drop-tor`):

```
0 3 * * * root /etc/iptables/drop_tor.sh >> /var/log/drop_tor.log 2>&1
```

---

## Relay-Server – Certbot Hooks

**Pre-Hook** `/etc/letsencrypt/renewal-hooks/pre/fw-open.sh`:

```bash
# /etc/letsencrypt/renewal-hooks/pre/fw-open.sh
#!/bin/bash
/usr/local/bin/fw-open-http.sh
```

**Post-Hook** `/etc/letsencrypt/renewal-hooks/post/fw-close.sh`:

```bash
# /etc/letsencrypt/renewal-hooks/post/fw-close.sh
#!/bin/bash
/usr/local/bin/fw-close-http.sh
```

**Post-Hook** `/etc/letsencrypt/renewal-hooks/post/reload-services.sh`:

```bash
# /etc/letsencrypt/renewal-hooks/post/reload-services.sh
#!/bin/bash
echo "[+] Reloading Postfix and Dovecot after cert renewal..."
systemctl reload postfix
systemctl reload dovecot
```

**`/usr/local/bin/fw-open-http.sh`:**

```bash
# /usr/local/bin/fw-open-http.sh
#!/bin/bash
iptables  -I INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -I INPUT -p tcp --dport 80 -j ACCEPT
```

**`/usr/local/bin/fw-close-http.sh`:**

```bash
# /usr/local/bin/fw-close-http.sh
#!/bin/bash
iptables  -D INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -D INPUT -p tcp --dport 80 -j ACCEPT
```

> Port 443 muss nicht temporär geöffnet werden – er ist in `update_relay_ip.sh` dauerhaft freigegeben.

---

## Relay-Server – Firewall-Skript

### `/usr/local/bin/update_relay_ip.sh`

Setzt die gesamte iptables/ip6tables-Policy des Relay dynamisch. Ermittelt die aktuelle IP des Heimservers per DNS, setzt alle Regeln neu, speichert sie persistent und aktualisiert die Unbound-`access-control`.

```bash
#!/bin/bash
set -euo pipefail
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

DYNDNS_NAME="{{HOME_HOSTNAME}}.ddns.net"
UNBOUND_CONF="/etc/unbound/unbound.conf.d/local.conf"

# WAN-Interface dynamisch bestimmen
WAN_IF=$(ip route | awk '/default/ {print $5; exit}')
: "${WAN_IF:?Konnte WAN-Interface nicht bestimmen}"

HOMESERVER_IPV4=$(dig +short A "$DYNDNS_NAME" | head -n 1 || true)
HOMESERVER_IPV6=$(dig +short AAAA "$DYNDNS_NAME" | head -n 1 || true)

if [ -z "${HOMESERVER_IPV4}" && -z "${HOMESERVER_IPV6}" ]( -z "${HOMESERVER_IPV4}" && -z "${HOMESERVER_IPV6}" .md); then
  echo "❌ Fehler: Konnte IP-Adresse für $DYNDNS_NAME nicht auflösen."
  exit 1
fi

# IPv4
iptables -F INPUT
iptables -F OUTPUT
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT

iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT

if ip link show docker0 >/dev/null 2>&1; then
  iptables -A FORWARD -i docker0 -o "$WAN_IF" -j ACCEPT
  iptables -A FORWARD -i "$WAN_IF" -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
  iptables -A FORWARD -i docker0 -o docker0 -j ACCEPT
fi

if [ -n "${HOMESERVER_IPV4}" ]( -n "${HOMESERVER_IPV4}" .md); then
  iptables -A INPUT -p tcp -s "$HOMESERVER_IPV4" --dport 22   -j ACCEPT
  iptables -A INPUT -p tcp -s "$HOMESERVER_IPV4" --dport 2525 -j ACCEPT
  iptables -A INPUT -p tcp -s "$HOMESERVER_IPV4" --dport 587  -j ACCEPT
  iptables -A INPUT -p udp -s "$HOMESERVER_IPV4" --dport 53   -j ACCEPT
  iptables -A INPUT -p tcp -s "$HOMESERVER_IPV4" --dport 53   -j ACCEPT
fi

iptables -A INPUT -p tcp --dport 25  -j ACCEPT
iptables -A INPUT -p tcp --dport 80  -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT

# IPv6
ip6tables -F INPUT
ip6tables -F OUTPUT
ip6tables -P INPUT DROP
ip6tables -P OUTPUT ACCEPT

ip6tables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
ip6tables -A INPUT -i lo -j ACCEPT

if [ -n "${HOMESERVER_IPV6}" ]( -n "${HOMESERVER_IPV6}" .md); then
  ip6tables -A INPUT -p tcp -s "$HOMESERVER_IPV6" --dport 22   -j ACCEPT
  ip6tables -A INPUT -p tcp -s "$HOMESERVER_IPV6" --dport 2525 -j ACCEPT
  ip6tables -A INPUT -p tcp -s "$HOMESERVER_IPV6" --dport 587  -j ACCEPT
  ip6tables -A INPUT -p tcp -s "$HOMESERVER_IPV6" --dport 53   -j ACCEPT
  ip6tables -A INPUT -p udp -s "$HOMESERVER_IPV6" --dport 53   -j ACCEPT
fi

ip6tables -A INPUT -p tcp   --dport 25  -j ACCEPT
ip6tables -A INPUT -p tcp   --dport 80  -j ACCEPT
ip6tables -A INPUT -p tcp   --dport 443 -j ACCEPT
ip6tables -A INPUT -p icmpv6 -j ACCEPT

# Regeln speichern
iptables-save  > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

# Unbound access-control aktualisieren
cp "$UNBOUND_CONF" "${UNBOUND_CONF}.bak"
sed -i '/# BEGIN HOMESERVER/,/# END HOMESERVER/d' "$UNBOUND_CONF"
{
  echo "    # BEGIN HOMESERVER (automatisch generiert)"
  [ -n "${HOMESERVER_IPV4}" ]( -n "${HOMESERVER_IPV4}" .md) && echo "    access-control: ${HOMESERVER_IPV4}/32 allow"
  [ -n "${HOMESERVER_IPV6}" ]( -n "${HOMESERVER_IPV6}" .md) && echo "    access-control: ${HOMESERVER_IPV6}/128 allow"
  echo "    # END HOMESERVER"
} >> "$UNBOUND_CONF"

systemctl is-active --quiet unbound && systemctl reload unbound || systemctl restart unbound
```

> Das Skript wird per Cronjob alle 15 Minuten ausgeführt (`/etc/cron.d/update-relay-ip`), damit Firewall und Unbound bei IP-Wechsel des Heimservers automatisch aktualisiert werden.

---

## Relay-Server – Backup

### `/usr/local/bin/backup_relay.sh`

Sichert die Relay-Konfiguration (Postfix, Dovecot, Let's Encrypt, Scripts) per `scp` auf den Heimserver. Wird auf dem **Relay** ausgeführt.

```bash
#!/bin/bash
set -euo pipefail

DATE=$(date +"%Y-%m-%d_%H-%M")
TARGET="relaybackup@{{HOME_HOSTNAME}}.ddns.net:/mnt/lvm/cloud/relay"
ARCHIVENAME="relay-backup-$DATE.tar.gz"
ARCHIVPFAD="/tmp/$ARCHIVENAME"
TMPDIR=$(mktemp -d)

mkdir -p "$TMPDIR/etc" "$TMPDIR/usr-local-bin"

cp -r /etc/postfix        "$TMPDIR/etc/"          || echo "⚠️  /etc/postfix fehlt"
cp -r /etc/dovecot        "$TMPDIR/etc/"          || echo "⚠️  /etc/dovecot fehlt"
cp -r /etc/letsencrypt    "$TMPDIR/etc/"          2>/dev/null || true
cp -r /usr/local/bin      "$TMPDIR/usr-local-bin/" 2>/dev/null || true
cp    /etc/aliases        "$TMPDIR/etc/"          2>/dev/null || true
cp    /etc/hostname       "$TMPDIR/etc/"          || true
cp    /etc/hosts          "$TMPDIR/etc/"          || true

tar czf "$ARCHIVPFAD" -C "$TMPDIR" .

scp -O -i ~/.ssh/id_relaybackup "$ARCHIVPFAD" "$TARGET" || {
  echo "❌ Fehler beim Übertragen!"
  exit 2
}

rm -rf "$TMPDIR" "$ARCHIVPFAD"
echo "✅ Backup erfolgreich nach $TARGET übertragen!"
```

> Voraussetzung: SSH-Key `~/.ssh/id_relaybackup` ist auf dem Heimserver für den User `relaybackup` autorisiert. Backup-Ziel: `/mnt/lvm/cloud/relay/` auf dem Heimserver.

---

---

## Heimserver – Scripts

### `/usr/local/bin/update-dedyn.sh`

Bereits dokumentiert weiter oben – aktualisiert `smtp` und `@` A-Records bei deSEC per API.

---

### `/usr/local/bin/update-tlsa.sh`

Aktualisiert den TLSA-Record für DANE (Port 465, SMTPS) nach manuellen Zertifikatsoperationen. Für automatische Erneuerung wird der Deploy-Hook genutzt.

```bash
#!/bin/bash
CERT_PATH="/etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem"
TOKEN=$(cat /root/.dedyn-token)

HASH=$(openssl x509 -in "$CERT_PATH" -noout -pubkey \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 | awk '{print $2}')

curl -X PATCH https://desec.io/api/v1/domains/{{DOMAIN}}/rrsets/_465._tcp.smtp/TLSA/ \
  -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"records\": [\"3 1 1 $HASH\"]}"
```

---

### `/usr/local/bin/mailcheck.sh`

Diagnoseskript – prüft SPF, DKIM, DMARC, PTR, MTA-STS und Spamhaus-Listing für die Domain. Nützlich nach Konfigurationsänderungen.

```bash
#!/bin/bash
DOMAIN="{{DOMAIN}}"
IP="{{RELAY_IP}}"
DKIM_SELECTOR="{{DKIM_SELECTOR}}"

echo "📧 Mailserver-Sicherheitscheck für: $DOMAIN"
echo "-----------------------------------------"

echo -e "\n🔍 SPF:"
dig +short TXT $DOMAIN | grep spf || echo "❌ Kein SPF-Eintrag."

echo -e "\n🔍 DKIM:"
dig +short TXT $DKIM_SELECTOR._domainkey.$DOMAIN || echo "❌ Kein DKIM-Eintrag."

echo -e "\n🔍 DMARC:"
dig +short TXT _dmarc.$DOMAIN || echo "❌ Kein DMARC-Eintrag."

echo -e "\n🔍 Reverse DNS:"
PTR=$(dig +short -x $IP)
[ $PTR == *$DOMAIN* ]( $PTR == *$DOMAIN* .md) && echo "✅ PTR: $PTR" || echo "❌ PTR: $PTR – passt nicht zur Domain!"

echo -e "\n🔍 MTA-STS DNS:"
dig +short TXT _mta-sts.$DOMAIN || echo "❌ Kein _mta-sts Eintrag."

echo -e "\n🔍 MTA-STS Policy:"
curl -s --max-time 5 https://mta-sts.$DOMAIN/.well-known/mta-sts.txt || echo "❌ Keine Policy abrufbar!"

echo -e "\n🔍 TLS-RPT:"
dig +short TXT _smtp._tls.$DOMAIN || echo "❌ Kein TLS-RPT-Eintrag."

echo -e "\n🔍 Spamhaus ZEN:"
REVERSE_IP=$(echo $IP | awk -F. '{print $4"."$3"."$2"."$1}')
BL=$(dig +short ${REVERSE_IP}.zen.spamhaus.org)
[ $BL == 127.* ]( $BL == 127.* .md) && echo "❌ GELISTET: $BL" || echo "✅ Nicht gelistet."
```

---

### `/usr/local/bin/mail-status.sh`

Schnellcheck ob Postfix, Dovecot und MariaDB laufen, plus Queue-Übersicht.

```bash
#!/bin/bash
services=("postfix" "dovecot" "mariadb")
GREEN="\033[0;32m"; RED="\033[0;31m"; NC="\033[0m"

echo "📡 Mailserver-Statusprüfung:"
for svc in "${services[@]}"; do
    systemctl is-active --quiet "$svc" \
      && echo -e "$svc: ${GREEN}läuft${NC}" \
      || echo -e "$svc: ${RED}gestoppt${NC}"
done

echo -e "\n📬 Postfix-Queue:"
postqueue -p | head -n 10
```

---

### `/usr/local/bin/init_blacklist.sh`

Einmaliges Setup-Skript – initialisiert die ipset-Blacklist, lädt externe Blacklists, setzt GeoIP-Blocking und erstellt die iptables-Chain `GEOIP_REJECT`. Nur beim ersten Einrichten oder nach einem vollständigen Reset ausführen.

```bash
#!/bin/bash
IP_BLACKLIST=/etc/ip-blacklist.conf
IP_BLACKLIST_TMP=/tmp/ip-blacklist.tmp
IP_BLACKLIST_LOCAL=/etc/ip-blacklist-local.conf
GEOIPLIST="/usr/local/bin/geoip.lst"

BLACKLISTS=(
  "http://check.torproject.org/cgi-bin/TorBulkExitList.py?ip=1.1.1.1"
  "http://rules.emergingthreats.net/blockrules/rbn-ips.txt"
  "http://www.spamhaus.org/drop/drop.lasso"
  "http://cinsscore.com/list/ci-badguys.txt"
  "http://lists.blocklist.de/lists/all.txt"
)

for url in "${BLACKLISTS[@]}"; do
    curl -s "$url" | grep -Po '(?:\d{1,3}\.){3}\d{1,3}(?:/\d{1,2})?' >> $IP_BLACKLIST_TMP
done

iptables -N GEOIP_REJECT
while read -r line; do
    [ "$line" =~ ^#.*$ ]( "$line" =~ ^#.*$ .md) && continue
    blockcode=${line%#*}
    iptables -I GEOIP_REJECT -m geoip --src-cc $blockcode -j DROP
done < "$GEOIPLIST"
iptables -A INPUT -j GEOIP_REJECT

/usr/local/bin/cpfail2ban2bl.sh
cat $IP_BLACKLIST_LOCAL >> $IP_BLACKLIST_TMP
sort -n -t . -k1,1 -k2,2 -k3,3 -k4,4 $IP_BLACKLIST_TMP | uniq > $IP_BLACKLIST
rm $IP_BLACKLIST_TMP

ipset create blacklist hash:net -exist
ipset flush blacklist
iptables -A INPUT -m set --match-set blacklist src -j DROP

egrep -v "^#|^$" $IP_BLACKLIST | while IFS= read -r ip; do
    ipset add blacklist $ip
done
```

> Voraussetzung: `iptables-mod-geoip` und `xt_geoip`-Daten müssen installiert sein. GeoIP-Daten mit `update_xt_geoip.sh` aktualisieren.

---

### `/usr/local/bin/update-blacklist.sh`

Aktualisiert die laufende Blacklist ohne Reset (Cronjob, täglich empfohlen).

```bash
#!/bin/bash
IP_BLACKLIST=/etc/ip-blacklist.conf
IP_BLACKLIST_TMP=/tmp/ip-blacklist.tmp
IP_BLACKLIST_LOCAL=/etc/ip-blacklist-local.conf
GEOIPLIST="/usr/local/bin/geoip.lst"

BLACKLISTS=(
  "http://check.torproject.org/cgi-bin/TorBulkExitList.py?ip=1.1.1.1"
  "http://rules.emergingthreats.net/blockrules/rbn-ips.txt"
  "http://www.spamhaus.org/drop/drop.lasso"
  "http://cinsscore.com/list/ci-badguys.txt"
  "http://lists.blocklist.de/lists/all.txt"
)

for url in "${BLACKLISTS[@]}"; do
    curl -s "$url" | grep -Po '(?:\d{1,3}\.){3}\d{1,3}(?:/\d{1,2})?' >> $IP_BLACKLIST_TMP
done

iptables -F GEOIP_REJECT
while read -r line; do
    [ "$line" =~ ^#.*$ ]( "$line" =~ ^#.*$ .md) && continue
    blockcode=${line%#*}
    iptables -I GEOIP_REJECT -m geoip --src-cc $blockcode -j DROP
done < "$GEOIPLIST"

/usr/local/bin/cpfail2ban2bl.sh
cat $IP_BLACKLIST_LOCAL >> $IP_BLACKLIST_TMP
sort -n -t . -k1,1 -k2,2 -k3,3 -k4,4 $IP_BLACKLIST_TMP | uniq > $IP_BLACKLIST
rm $IP_BLACKLIST_TMP

ipset flush blacklist
egrep -v "^#|^$" $IP_BLACKLIST | while IFS= read -r ip; do
    ipset add blacklist $ip
done
```

Empfohlener Cronjob (`/etc/cron.d/update-blacklist`):

```
0 2 * * * root /usr/local/bin/update-blacklist.sh >> /var/log/update-blacklist.log 2>&1
```

---

### `/usr/local/bin/cpfail2ban2bl.sh`

Abhängigkeit von `init_blacklist.sh` und `update-blacklist.sh`. Überträgt gebannte IPs aus dem Fail2Ban-Log in die lokale Blacklist.

```bash
#!/bin/bash
grep "Ban" /var/log/fail2ban.log \
  | grep -Po '(?:\d{1,3}\.){3}\d{1,3}(?:/\d{1,2})?' \
  >> /etc/ip-blacklist-local.conf

sort -n -t . -k1,1 -k2,2 -k3,3 -k4,4 /etc/ip-blacklist-local.conf \
  | uniq > /etc/ip-blacklist-local.tmp \
  && mv /etc/ip-blacklist-local.tmp /etc/ip-blacklist-local.conf
```

---

### `/usr/local/bin/update_xt_geoip.sh`

Aktualisiert die MaxMind GeoIP-Datenbank für `xt_geoip`. Benötigt einen gültigen MaxMind-Lizenzkey.

```bash
#!/bin/bash
cd /usr/share/xt_geoip
URL="https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country-CSV&license_key={{SECRET_MAXMIND_KEY}}&suffix=zip"
wget "$URL" -O geoip.zip
unzip -o geoip.zip
dir=$(ls -d */)
cd "$dir"
/usr/libexec/xtables-addons/xt_geoip_build_maxmind -D /usr/share/xt_geoip *.csv
cd ..
modprobe xt_geoip
```

> MaxMind-Lizenzkey in `/etc/environment` oder als Variable in `00_variablen.md` unter `{{SECRET_MAXMIND_KEY}}` dokumentieren. Kostenloser Account auf [maxmind.com](https://www.maxmind.com).

---

### `/usr/local/bin/geoip.lst`

Länderliste für GeoIP-Blocking. Länder ohne `#` am Anfang werden geblockt. Erlaubte Länder (DE, AT, CH, US etc.) sind auskommentiert.

```
# Auskommentiert = erlaubt:
#	DE	#	Germany
#	AT	#	Austria
#	CH	#	Switzerland
#	US	#	United States
# ...

# Aktiv = geblockt (Beispiele):
	CN	#	China
	RU	#	Russia
	KP	#	North Korea
```

> Die vollständige Liste liegt auf dem Heimserver unter `/usr/local/bin/geoip.lst`.

---

### Historisch: `dovecot-lda-wrapper.sh`

```bash
#!/bin/bash
exec sudo -u vmail /usr/lib/dovecot/dovecot-lda "$@"
```

> Dieses Skript wurde früher als Wrapper für die LDA-Zustellung verwendet. Es ist durch die direkte Pipe-Konfiguration in Postfix `master.cf` (`dovecot unix - n n - - pipe flags=DRhu user=vmail:vmail ...`) ersetzt worden und wird nicht mehr benötigt.

---

## Platzhalter-Referenz

| Platzhalter | Bedeutung |
|---|---|
| `{{DOMAIN}}` | Primäre Domain |
| `{{RELAY_HOSTNAME}}` | FQDN Relay-Server |
| `{{RELAY_IP}}` | Statische IP des Relay-Servers |
| `{{HOME_SMTP}}` | FQDN Heimserver Submission |
| `{{HOME_IMAP}}` | FQDN Heimserver IMAP |
| `{{HOME_HOSTNAME}}` | Interner Hostname des Heimservers |
| `{{DKIM_SELECTOR}}` | DKIM-Selektor |
| `{{ADMIN_MAIL}}` | Administrative Mailadresse |
| `{{SECRET_DB_PASSWORD}}` | Passwort des PostfixAdmin DB-Users |
| `{{SECRET_RELAY_SASL_PASSWORD}}` | SASL-Passwort für Heimserver → Relay |
| `{{SECRET_RSPAMD_CONTROLLER_PASSWORD}}` | Passwort für Rspamd Controller |
| `{{SECRET_MAXMIND_KEY}}` | MaxMind GeoLite2 Lizenzkey |

> Alle Platzhalter sind in [Variablen und Platzhalter](../00_Einleitung/00_variablen.md) definiert.

