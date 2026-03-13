
# Automatisierung

Der laufende Betrieb erfordert mehrere wiederkehrende Aufgaben, die automatisiert werden sollten:

- IP-Adresse bei DNS-Änderung aktualisieren (zwei parallele Systeme)
- TLS-Zertifikate automatisch erneuern
- TLSA-Record nach Zertifikatserneuerung aktualisieren (DANE)
- Mailserver nach Zertifikatserneuerung neu laden
- Backups der Maildaten

---

## DynDNS – zwei parallele Systeme

Der Heimserver nutzt zwei unabhängige DynDNS-Mechanismen, die beide die aktuelle IP aktuell halten:

| System | Hostname | Methode |
|---|---|---|
| No-IP Client | `{{HOME_HOSTNAME}}.ddns.net` | No-IP-Daemon auf dem Server |
| deSEC-API-Skript | `{{HOME_SMTP}}`, `{{DOMAIN}}` | Cronjob alle 5 Minuten |

`{{HOME_SMTP}}` ist ein direkter A-Record bei deSEC – kein CNAME auf `{{HOME_HOSTNAME}}.ddns.net`. Beide Systeme aktualisieren dieselbe IP unabhängig voneinander.

### No-IP Client

Der No-IP-Daemon (`noip2`) läuft als Systemdienst und aktualisiert `{{HOME_HOSTNAME}}.ddns.net` automatisch. Die Binary liegt unter `/usr/local/bin/noip2`.

Status prüfen:

```bash
systemctl status noip2
# oder direkt:
/usr/local/bin/noip2 -S
```

Konfiguration und Installation sind in der [No-IP-Dokumentation](https://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client/) beschrieben.

### deSEC-API-Skript

`/usr/local/bin/update-dedyn.sh`:

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

### API-Token hinterlegen

Den deSEC-API-Token in `/root/.dedyn-token` ablegen:

```bash
echo "DEIN_TOKEN_HIER" > /root/.dedyn-token
chmod 600 /root/.dedyn-token
```

> Der Token wird über das deSEC Control Panel unter **Token Management** erstellt. Minimale Berechtigung: Schreibzugriff auf die eigene Domain.

### Skript ausführbar machen

```bash
chmod +x /usr/local/bin/update-dedyn.sh
```

### Cron-Job einrichten

Das Skript läuft alle 5 Minuten:

```
*/5 * * * * /usr/local/bin/update-dedyn.sh >> /dev/null 2>&1
```

Eintragen:

```bash
crontab -e
```

### Aktualisierte Records

Das Skript pflegt zwei A-Records:

| Record | Hostname | Zweck |
|---|---|---|
| `smtp` | `{{HOME_SMTP}}` | Submission-Endpunkt für Mailclients |
| `%40` (= `@`) | `{{DOMAIN}}` | Root-Domain |

> `%40` ist die URL-kodierte Form von `@` und wird von der deSEC-API für den Root-Record akzeptiert.

### Log prüfen

```bash
tail -f /var/log/dedyn-update.log
```

---

## TLS-Zertifikate automatisch erneuern

Certbot installiert beim ersten Aufruf automatisch einen Systemd-Timer für die Erneuerung. Die Deploy-Hooks in `/etc/letsencrypt/renewal-hooks/deploy/` werden nach jeder erfolgreichen Erneuerung ausgeführt.

Timer prüfen:

```bash
systemctl list-timers | grep certbot
```

Erneuerung testen:

```bash
certbot renew --dry-run
```

Die Hooks für den automatischen Reload der Maildienste sind in [TLS für IMAP und SMTP](../03_Konfiguration/11_tls_imap_smtp.md) beschrieben.

### Certbot Hooks im Überblick

| Hook-Typ | Datei | Aktion |
|---|---|---|
| Pre | `001-open-ports.sh` | Öffnet Port 80+443 für HTTP-01-Challenge |
| Post | `001-close-ports.sh` | Schließt Port 80+443 wieder |
| Deploy | `001-dovecot-apache-restart.sh` | Startet Dovecot und Apache neu |
| Deploy | `update-tlsa.sh` | Aktualisiert TLSA-Record bei deSEC |

---

## DANE/TLSA – automatische Aktualisierung

DANE (DNS-Based Authentication of Named Entities) ermöglicht es, das TLS-Zertifikat eines Mailservers über DNS zu verifizieren. Der TLSA-Record enthält einen Hash des öffentlichen Schlüssels – empfangende Server können damit prüfen, ob das präsentierte Zertifikat mit dem DNS-Eintrag übereinstimmt.

Da Let's Encrypt-Zertifikate alle 90 Tage erneuert werden und sich dabei der öffentliche Schlüssel ändern kann, muss der TLSA-Record nach jeder Erneuerung aktualisiert werden. Das übernimmt der Deploy-Hook automatisch.

### TLSA-Record Aufbau

```
_25._tcp.{{HOME_SMTP}}.    TLSA    3 1 1 <hash>
```

| Feld | Wert | Bedeutung |
|---|---|---|
| `3` | DANE-EE | End-Entity-Zertifikat (nicht CA) |
| `1` | SPKI | Hash über den Public Key (nicht das gesamte Zertifikat) |
| `1` | SHA-256 | Hash-Algorithmus |

### Deploy-Hook

`/etc/letsencrypt/renewal-hooks/deploy/update-tlsa.sh`:

```bash
#!/bin/bash
TLSA_HASH=$(openssl x509 -in /etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem -noout -pubkey \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | xxd -p -c 64)

curl -X PATCH https://desec.io/api/v1/domains/{{DOMAIN}}/rrsets/_25._tcp.smtp/TLSA/ \
  -H "Authorization: Token $(cat /root/.dedyn-token)" \
  -H "Content-Type: application/json" \
  -d "{\"records\": [\"3 1 1 $TLSA_HASH\"]}"
```

### TLSA-Record initial anlegen

Beim ersten Einrichten muss der Record manuell angelegt werden:

```bash
# Hash berechnen
TLSA_HASH=$(openssl x509 -in /etc/letsencrypt/live/{{HOME_SMTP}}/fullchain.pem -noout -pubkey \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | xxd -p -c 64)

echo "TLSA Hash: $TLSA_HASH"

# Record bei deSEC anlegen (POST statt PATCH für neuen Record)
curl -X POST https://desec.io/api/v1/domains/{{DOMAIN}}/rrsets/ \
  -H "Authorization: Token $(cat /root/.dedyn-token)" \
  -H "Content-Type: application/json" \
  -d "{\"subname\": \"_25._tcp.smtp\", \"type\": \"TLSA\", \"ttl\": 3600, \"records\": [\"3 1 1 $TLSA_HASH\"]}"
```

### TLSA-Record prüfen

```bash
dig TLSA _25._tcp.{{HOME_SMTP}}
```

---

## Backups

Folgende Daten müssen regelmäßig gesichert werden:

| Pfad | Inhalt | Server |
|---|---|---|
| `/var/vmail` | Alle Mailboxen (Maildir) | Heimserver |
| `/etc/postfix` | Postfix-Konfiguration + MySQL-Maps | Heimserver |
| `/etc/dovecot` | Dovecot-Konfiguration inkl. `dovecot-mysql.conf` | Heimserver |
| `/etc/opendkim` | OpenDKIM-Konfiguration + Schlüssel | Heimserver |
| `/etc/rspamd/local.d` | Rspamd-Konfiguration | Relay |
| `/etc/rspamd/dkim` | DKIM-Schlüssel (Rspamd) | Relay |
| `/etc/letsencrypt` | TLS-Zertifikate + Renewal-Config + Hooks | beide |
| `/root/.dedyn-token` | deSEC API-Token | Heimserver |
| `/usr/local/bin/update-dedyn.sh` | DynDNS-Skript | Heimserver |
| MySQL-Dump `postfixadmin` | Domains, Mailboxen, Aliases | Heimserver |

> **Secrets nicht unverschlüsselt sichern:** DKIM-Schlüssel, API-Token und TLS-Private-Keys sollten verschlüsselt oder in einem separaten, gesicherten Backup-Ziel abgelegt werden.

Einfaches Backup-Skript als Ausgangspunkt:

```bash
#!/bin/bash
BACKUP_DIR="/backup/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

rsync -a /var/vmail "$BACKUP_DIR/"
rsync -a /etc/postfix "$BACKUP_DIR/"
rsync -a /etc/dovecot "$BACKUP_DIR/"
rsync -a /etc/opendkim "$BACKUP_DIR/"
rsync -a /etc/letsencrypt "$BACKUP_DIR/"
```

---

## ✅ Ergebnis

Nach diesem Kapitel:

- IP-Änderungen werden automatisch per deSEC-API propagiert
- TLS-Zertifikate werden automatisch erneuert und Dienste danach neu geladen
- Alle kritischen Daten sind in einem Backup-Konzept erfasst

---

## 🔁 Navigation

**← Zurück:** [Spamfilter & Virenschutz](../03_Konfiguration/12_spamfilter.md)  
**→ Weiter:** [Troubleshooting](../04_Betrieb/15_troubleshooting.md)

