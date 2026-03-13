
# Betrieb und Wartung

Ein selbst gehosteter Mailserver erfordert regelmäßige Aufmerksamkeit. Dieses Kapitel beschreibt die wiederkehrenden Aufgaben und sinnvolle Routinen für den stabilen Langzeitbetrieb.

---

## Regelmäßige Aufgaben im Überblick

| Aufgabe | Intervall | Server |
|---|---|---|
| Sicherheitsupdates einspielen | wöchentlich | beide |
| Mail-Logs auf Fehler prüfen | wöchentlich | beide |
| Blacklist-Status prüfen | monatlich | Relay |
| DMARC-Reports auswerten | monatlich | – |
| Zertifikatsgültigkeit prüfen | monatlich | beide |
| Rspamd-Statistiken prüfen | monatlich | Relay |
| Backup testen | quartalsweise | Heimserver |

---

## Sicherheitsupdates

```bash
apt update && apt upgrade -y
```

Auf beiden Servern ausführen. Für kritische Pakete (Postfix, Dovecot, Rspamd, OpenDKIM) sollte ein Neustart der betroffenen Dienste erfolgen:

```bash
systemctl restart postfix dovecot opendkim   # Heimserver
systemctl restart postfix rspamd             # Relay
```

Automatische Sicherheitsupdates mit `unattended-upgrades`:

```bash
apt install unattended-upgrades
dpkg-reconfigure unattended-upgrades
```

---

## Logs überwachen

> **Hinweis:** Postfix auf dem Heimserver loggt via **journald**, nicht nach `/var/log/mail.log`. Auf dem Relay hingegen loggt Postfix nach `/var/log/mail.log`.

**Mailflow auf dem Relay:**

```bash
# Letzte Zustellungen
grep "status=sent" /var/log/mail.log | tail -20

# Abgelehnte Mails
grep "status=bounced\|reject" /var/log/mail.log | tail -20

# Verbindungsfehler zum Heimserver
grep "Connection refused\|{{HOME_SMTP}}" /var/log/mail.log | tail -20
```

**Mailflow auf dem Heimserver:**

```bash
journalctl -u postfix --since "1 hour ago" | grep "status="
journalctl -u postfix --since "1 hour ago" | grep "reject"
```

**Spamfilter-Aktivität:**

```bash
rspamc stat
```

**DynDNS-Updates:**

```bash
tail -20 /var/log/dedyn-update.log
```

---

## Blacklist-Status prüfen

Die IP des Relay-Servers (`{{RELAY_IP}}`) sollte nicht auf Blocklisten stehen.

Manuelle Prüfung:

- [MXToolbox Blacklist Check](https://mxtoolbox.com/blacklists.aspx) – prüft ~100 Listen gleichzeitig
- [Spamhaus](https://check.spamhaus.org)

Wenn die IP gelistet ist: Beim jeweiligen Anbieter einen Delisting-Antrag stellen. Spamhaus und die meisten anderen Listen bieten ein Self-Service-Formular an.

---

## DMARC-Reports auswerten

Empfangende Mailserver senden täglich XML-Berichte an die `rua`-Adresse. Diese zeigen ob Mails von unbekannten Quellen im Namen der Domain versendet werden.

Auswertung über:

- [Postmark DMARC Digests](https://dmarc.postmarkapp.com) – kostenlos, übersichtlich
- [dmarcian](https://dmarcian.com) – detaillierte Analyse

Wenn Reports zeigen, dass alle legitimen Quellen SPF und DKIM bestehen: Policy auf `p=reject` verschärfen (siehe [DMARC konfigurieren](../03_Konfiguration/10_dmarc.md)).

---

## Zertifikatsgültigkeit prüfen

```bash
certbot certificates
```

Zertifikate mit weniger als 30 Tagen Restlaufzeit manuell erneuern:

```bash
certbot renew
```

Ablaufdatum direkt prüfen:

```bash
echo | openssl s_client -connect {{RELAY_HOSTNAME}}:25 -starttls smtp 2>/dev/null \
  | openssl x509 -noout -dates

echo | openssl s_client -connect {{HOME_IMAP}}:993 2>/dev/null \
  | openssl x509 -noout -dates
```

---

## Speicherplatz überwachen

Mailboxen wachsen. Speicherplatz auf dem Heimserver regelmäßig prüfen:

```bash
df -h /var/vmail
du -sh /var/vmail/*
```

---

## Konfigurationsänderungen dokumentieren

Bei Änderungen an Postfix, Dovecot oder Rspamd:

1. Änderung in der Obsidian-Dokumentation nachführen
2. Konfiguration exportieren und sichern:

```bash
# Postfix effektive Konfiguration
postconf -n > /backup/postfix.postconf-n.$(date +%Y-%m-%d).txt

# Dovecot effektive Konfiguration
doveconf -n > /backup/dovecot.doveconf-n.$(date +%Y-%m-%d).txt

# Rspamd effektive Konfiguration
rspamadm configdump > /backup/rspamd.configdump.$(date +%Y-%m-%d).txt
```

Diese Exporte zeigen die **tatsächlich aktive** Konfiguration – deutlich zuverlässiger als einzelne Konfigurationsdateien.

---

## Dienste nach Systemstart prüfen

Nach Reboots oder Updates sicherstellen, dass alle Dienste gestartet sind. Schnellcheck mit dem Statusskript:

```bash
/usr/local/bin/mail-status.sh
```

Das Skript zeigt den Status von Postfix, Dovecot und MariaDB farbig an und gibt einen Überblick über die Postfix-Queue.

Manuell:

**Heimserver:**

```bash
systemctl is-active postfix dovecot opendkim
```

**Relay:**

```bash
systemctl is-active postfix rspamd clamav-daemon clamav-freshclam unbound
```

Nach einem Reboot des Relay auch die Firewall prüfen – `update_relay_ip.sh` läuft per Cronjob, aber ein manueller Aufruf stellt sicher dass die Regeln sofort aktuell sind:

```bash
/usr/local/bin/update_relay_ip.sh
iptables -L INPUT -n --line-numbers
```

---

## Open-Resolver-Check

Der Relay betreibt Unbound als DNS-Resolver. Es sollte regelmäßig geprüft werden, dass Unbound nicht als Open Resolver erreichbar ist (d. h. nur vom Heimserver aus antworten darf).

Manueller Check:

```bash
dig +short test.openresolver.com TXT
# Erwartetes Ergebnis: KEIN "open-resolver-detected"
```

Auf dem Relay liegt das Diagnoseskript `/usr/local/bin/openresolver-watch.sh`, das den Check automatisiert und bei Entwarnung eine Mail sendet. Es ist für den einmaligen Einsatz nach Setup-Änderungen gedacht, nicht als dauerhafter Cronjob.

---

## ✅ Ergebnis

Mit den in diesem Kapitel beschriebenen Routinen bleibt der Mailserver langfristig stabil, sicher und zustellfähig.

---

## 🔁 Navigation

**← Zurück:** [Troubleshooting](../04_Betrieb/15_troubleshooting.md)  
**→ Weiter:** [Übersicht](../index.md)

