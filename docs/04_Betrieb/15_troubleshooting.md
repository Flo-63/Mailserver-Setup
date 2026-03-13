
# Troubleshooting

Viele Probleme beim Mailbetrieb lassen sich durch systematische Prüfung von DNS, Logs und Verbindungen schnell eingrenzen. Dieses Kapitel beschreibt die wichtigsten Diagnoseschritte.

---

## Schnellcheck: Systemstatus

Zuerst prüfen ob alle Dienste laufen:

**Relay-Server:**

```bash
systemctl status postfix rspamd clamav-daemon
```

**Heimserver:**

```bash
systemctl status postfix dovecot opendkim
```

---

## DNS prüfen

Die häufigste Fehlerquelle. Schnellcheck mit dem Diagnoseskript:

```bash
/usr/local/bin/mailcheck.sh
```

Das Skript prüft SPF, DKIM, DMARC, PTR, MTA-STS und Spamhaus-Listing in einem Durchgang. Die vollständige Version ist in der [Config Library](../05_Referenz/config_library.md) dokumentiert.

Einzelne Records manuell prüfen:

```bash
# MX
dig MX {{DOMAIN}}

# SPF
dig TXT {{DOMAIN}}

# DKIM
dig TXT default._domainkey.{{DOMAIN}}

# DMARC
dig TXT _dmarc.{{DOMAIN}}

# A-Record Relay
dig A {{RELAY_HOSTNAME}}

# Heimserver (DynDNS)
dig A {{HOME_SMTP}}
```

Online-Tools für eine vollständige Analyse:

- [MXToolbox](https://mxtoolbox.com) – MX, SPF, DMARC, Blacklists
- [mail-tester.com](https://www.mail-tester.com) – Gesamtbewertung einer Testmail
- [DNSViz](https://dnsviz.net) – DNSSEC-Analyse

---

## Reverse DNS prüfen

```bash
dig -x {{RELAY_IP}}
# Erwartete Ausgabe: {{RELAY_HOSTNAME}}
```

Forward und Reverse müssen übereinstimmen:

```
{{RELAY_HOSTNAME}}  →  {{RELAY_IP}}   (A-Record)
{{RELAY_IP}}   →  {{RELAY_HOSTNAME}}  (PTR-Record)
```

Der PTR-Record wird beim Hoster (Netcup) konfiguriert, nicht bei deSEC.

---

## SMTP-Verbindung testen

**Relay – Port 25:**

```bash
telnet {{RELAY_HOSTNAME}} 25
# Erwartete Ausgabe: 220 {{RELAY_HOSTNAME}} ESMTP Postfix

openssl s_client -connect {{RELAY_HOSTNAME}}:25 -starttls smtp
```

**Relay – Submission Port 587:**

```bash
openssl s_client -connect {{RELAY_HOSTNAME}}:587 -starttls smtp
```

**Heimserver – Submission Port 587:**

```bash
openssl s_client -connect {{HOME_SMTP}}:587 -starttls smtp
```

---

## IMAP-Verbindung testen

```bash
openssl s_client -connect {{HOME_IMAP}}:993
# Erwartete Ausgabe: * OK Dovecot ready.
```

---

## Logs prüfen

> **Wichtig:** Postfix auf dem **Heimserver** loggt via journald – nicht nach `/var/log/mail.log`. Auf dem **Relay** loggt Postfix nach `/var/log/mail.log`.

**Relay – Postfix live:**

```bash
tail -f /var/log/mail.log
```

**Heimserver – Postfix live:**

```bash
journalctl -u postfix -f
```

**Heimserver – Postfix letzte Stunde:**

```bash
journalctl -u postfix --since "1 hour ago"
```

**Nach einer bestimmten Mail suchen** (Queue-ID aus dem Log):

```bash
# Relay:
grep "QUEUE-ID" /var/log/mail.log

# Heimserver:
journalctl -u postfix | grep "QUEUE-ID"
```

**Rspamd-Log auf dem Relay:**

```bash
tail -f /var/log/rspamd/rspamd.log
```

**DynDNS-Log auf dem Heimserver:**

```bash
tail -f /var/log/dedyn-update.log
```

---

## Rspamd startet nicht

Rspamd auf dem Relay hat mehrere bekannte Fallstricke:

**1. Kein Newline am Ende von override.conf:**

```bash
cat -A /etc/systemd/system/rspamd.service.d/override.conf
# Letzte Zeile muss mit $ enden (= Newline vorhanden)
# Falls nicht:
echo "" >> /etc/systemd/system/rspamd.service.d/override.conf
systemctl daemon-reload
```

**2. `/run/rspamd/` fehlt:**

```bash
ls /run/rspamd/ 2>/dev/null || echo "fehlt"
mkdir -p /run/rspamd
chown _rspamd:_rspamd /run/rspamd
# Dauerhaft via tmpfiles.d:
echo "d /run/rspamd 0755 _rspamd _rspamd -" > /etc/tmpfiles.d/rspamd.conf
```

**3. Port-Konflikt:**

Port 12301 ist auf dem Relay von OpenDKIM belegt. Rspamd muss Port 11332 verwenden:

```bash
grep "bind_socket" /etc/rspamd/local.d/worker-proxy.inc
# Muss sein: bind_socket = "127.0.0.1:11332";
# Falls 12301: sed -i 's/12301/11332/' /etc/rspamd/local.d/worker-proxy.inc
```

Postfix muss denselben Port verwenden:

```bash
postconf smtpd_milters
# Muss sein: inet:localhost:11332
```

**Diagnose direkt als _rspamd:**

```bash
sudo -u _rspamd rspamd -c /etc/rspamd/rspamd.conf -f 2>&1
tail -20 /var/log/rspamd/rspamd.log
```

**Status prüfen:**

```bash
systemctl status rspamd
rspamc -h 127.0.0.1:11334 stat
ss -tlnp | grep -E "11332|11334"
```

---

## DKIM-Probleme

Checkliste bei fehlendem oder ungültigem DKIM:

1. DNS-Record vorhanden?

```bash
dig TXT default._domainkey.{{DOMAIN}}
```

2. OpenDKIM läuft und Port ist offen?

```bash
systemctl status opendkim
ss -tlnp | grep 12301
```

3. Schlüssel gültig?

```bash
opendkim-testkey -d {{DOMAIN}} -s default -vvv
# Erwartete Ausgabe: key OK
```

4. Milter in Postfix eingebunden?

```bash
postconf smtpd_milters
postconf non_smtpd_milters
# Erwartete Ausgabe: inet:localhost:12301
```

Wenn die Milter-Einträge leer sind, signiert OpenDKIM nicht – das ist die häufigste Fehlerursache. Korrektur in [DKIM einrichten](../03_Konfiguration/09_dkim.md).

---

## SPF-Probleme

```bash
dig TXT {{DOMAIN}} | grep spf
```

Häufige Fehler:

- Relay-IP nicht im SPF-Record enthalten
- Mehrere TXT-Records mit `v=spf1` (nur einer erlaubt)
- `-all` vs `~all` – Hardfail vs Softfail

---

## DMARC-Probleme

DMARC schlägt fehl wenn SPF **und** DKIM beide nicht passen. Einzeln prüfen:

```bash
dig TXT _dmarc.{{DOMAIN}}
```

Häufige Fehler:

- Alignment: Die Domain in `From:` muss mit der SPF/DKIM-Domain übereinstimmen
- `adkim=s` oder `aspf=s` (strict) schlägt fehl wenn Subdomains verwendet werden

---

## Mails landen im Spam

Wenn Mails zugestellt werden aber im Spam landen:

1. [mail-tester.com](https://www.mail-tester.com) – Testmail senden, Score und Diagnose lesen
2. DKIM im Header prüfen – `DKIM-Signature` muss vorhanden sein
3. IP-Reputation prüfen: [MXToolbox Blacklist Check](https://mxtoolbox.com/blacklists.aspx)
4. DMARC-Reports auswerten (siehe [DMARC konfigurieren](../03_Konfiguration/10_dmarc.md))

---

## IP auf Blacklist

```bash
# Schnellcheck über MXToolbox (manuell im Browser)
# Oder per dig gegen bekannte Blacklists:
dig 169.21.53.152.zen.spamhaus.org
# Keine Antwort = nicht gelistet
# Antwort 127.0.0.x = gelistet
```

> Die IP-Bytes werden für die Abfrage umgekehrt: `{{RELAY_IP}}` → `169.21.53.152`

---

## Weiterleitung vom Relay zum Heimserver prüfen

Wenn Mails den Relay erreichen aber nicht beim Heimserver ankommen:

```bash
# Auf dem Relay:
tail -f /var/log/mail.log | grep {{HOME_SMTP}}
```

Typische Fehler:

- Heimserver-IP hat sich geändert, DNS noch nicht aktualisiert → DynDNS-Log prüfen
- Port 2525 auf dem Heimserver nicht erreichbar → Portweiterleitung im Router prüfen
- `transport_maps` falsch konfiguriert → `postconf transport_maps` prüfen

---

## End-to-End-Test

Vollständiger Test des Mailwegs nach Einrichtung oder nach Änderungen.

### 1. Testmail senden und Header prüfen

Testmail an [mail-tester.com](https://www.mail-tester.com) senden – die Seite zeigt eine temporäre Adresse, an die die Testmail geschickt wird. Ergebnis zeigt Score und Diagnose für SPF, DKIM, DMARC, Blacklists.

Alternativ: Testmail an eine externe Adresse (z. B. Gmail) senden und den Header prüfen:

```
# Im Mailclient: "Original anzeigen" / "Show original"
# Folgende Felder müssen vorhanden und korrekt sein:

Received-SPF: pass
DKIM-Signature: v=1; a=rsa-sha256; d={{DOMAIN}}; s={{DKIM_SELECTOR}}; ...
Authentication-Results: ... dkim=pass; spf=pass; dmarc=pass
```

### 2. Mailweg im Log verfolgen

**Ausgehende Mail (Heimserver → Relay → Internet):**

```bash
# Auf dem Heimserver – Mail in die Queue eingeliefert:
journalctl -u postfix | grep "to=<testempfaenger"

# Auf dem Relay – Mail weitergeleitet:
grep "testempfaenger" /var/log/mail.log
```

**Eingehende Mail (Internet → Relay → Heimserver):**

```bash
# Auf dem Relay – Mail empfangen und weitergeleitet:
grep "to=<{{ADMIN_MAIL}}" /var/log/mail.log

# Auf dem Heimserver – Mail zugestellt:
journalctl -u postfix | grep "to=<{{ADMIN_MAIL}}"
```

### 3. Checkliste

| Prüfpunkt | Befehl / Tool | Erwartetes Ergebnis |
|---|---|---|
| SPF-Record | `dig TXT {{DOMAIN}}` | `v=spf1 mx -all` |
| DKIM-Record | `dig TXT {{DKIM_SELECTOR}}._domainkey.{{DOMAIN}}` | Langer `p=`-Wert |
| DMARC-Record | `dig TXT _dmarc.{{DOMAIN}}` | `v=DMARC1; p=...` |
| MX-Record | `dig MX {{DOMAIN}}` | `{{RELAY_HOSTNAME}}` |
| SMTP Relay erreichbar | `telnet {{RELAY_HOSTNAME}} 25` | `220 ... ESMTP` |
| Submission erreichbar | `openssl s_client -connect {{HOME_SMTP}}:587 -starttls smtp` | Zertifikat + `220` |
| IMAP erreichbar | `openssl s_client -connect {{HOME_IMAP}}:993` | `* OK Dovecot ready` |
| DKIM in Header | Mailheader prüfen | `DKIM-Signature` vorhanden |
| SPF-Ergebnis | Mailheader prüfen | `Received-SPF: pass` |
| DMARC-Ergebnis | Mailheader prüfen | `dmarc=pass` |
| Relay-IP nicht gelistet | [MXToolbox Blacklist](https://mxtoolbox.com/blacklists.aspx) | Keine Treffer |

---

## ✅ Ergebnis

Mit den in diesem Kapitel beschriebenen Prüfungen lassen sich die meisten Probleme diagnostizieren. Typische Ursachen sind fehlerhafte DNS-Konfiguration, fehlende oder nicht eingebundene DKIM-Signierung und Verbindungsprobleme zwischen Relay und Heimserver.

---

## 🔁 Navigation

**← Zurück:** [Automatisierung](../04_Betrieb/13_automatisierung.md)  
**→ Weiter:** [Betrieb und Wartung](../04_Betrieb/14_betrieb_und_wartung.md)

