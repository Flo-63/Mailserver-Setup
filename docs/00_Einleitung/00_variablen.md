
# Variablen und Platzhalter

Diese Datei definiert alle Platzhalter, die in der gesamten Dokumentation verwendet werden. Trage hier deine eigenen Werte ein – die Dokumentation verwendet durchgehend die Platzhalter in der Form `{{PLATZHALTER}}`.

---

## Domain und DNS

| Platzhalter | Dein Wert | Bedeutung |
|---|---|---|
| `{{DOMAIN}}` | | Primäre Domain, z. B. `example.com` |
| `{{DKIM_SELECTOR}}` | | DKIM-Selektor, z. B. `default` |
| `{{DMARC_MAIL}}` | | Empfängeradresse für DMARC-Reports, z. B. `dmarc@example.com` |
| `{{ADMIN_MAIL}}` | | Administrative Kontaktadresse, z. B. `admin@example.com` |

---

## Relay-Server

| Platzhalter | Dein Wert | Bedeutung |
|---|---|---|
| `{{RELAY_HOSTNAME}}` | | FQDN des Relay-Servers, z. B. `mail.example.com` |
| `{{RELAY_IP}}` | | Statische IP des Relay-Servers, z. B. `1.2.3.4` |

---

## Heimserver

| Platzhalter | Dein Wert | Bedeutung |
|---|---|---|
| `{{HOME_HOSTNAME}}` | | Interner Hostname des Heimservers, z. B. `mailhome` |
| `{{HOME_SMTP}}` | | FQDN Submission-Endpunkt, z. B. `smtp.example.com` |
| `{{HOME_IMAP}}` | | FQDN IMAP-Endpunkt, z. B. `imap.example.com` |
| `{{HOME_IP}}` | | Aktuelle IP des Heimservers (dynamisch) |
| `{{HOME_RDNS}}` | | Reverse-DNS des Heimanschlusses (vom ISP vergeben) |

---

## Secrets

| Platzhalter | Dein Wert | Bedeutung |
|---|---|---|
| `{{SECRET_DB_PASSWORD}}` | | Passwort des PostfixAdmin MySQL-Users |
| `{{RELAY_SASL_USER}}` | | SASL-Benutzername für Heimserver → Relay, z. B. `smtpuser` |
| `{{SECRET_RELAY_SASL_PASSWORD}}` | | SASL-Passwort Heimserver → Relay (in `sasl_passwd`) |

> **Nie im Klartext in der Dokumentation speichern.** Diese Werte nur lokal in den Konfigurationsdateien eintragen.

---

## Hinweise

- `{{HOME_IP}}` und `{{HOME_RDNS}}` ändern sich bei dynamischen Heimanschlüssen regelmäßig – sie sind in der Dokumentation nur als Beispielwerte aus dem Mailheader zu verstehen.
- `{{DOMAIN}}` und `{{DESEC_DOMAIN}}` sind in den meisten Setups identisch.
- Alle Platzhalter werden in Konfigurationsdateien, Scripts und Dokumentationskapiteln einheitlich verwendet.

