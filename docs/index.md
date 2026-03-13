
# 📨 Self-Hosted Mailserver mit dynamischer IP

Diese Dokumentation beschreibt eine selbst betriebene Mailinfrastruktur mit einem **Relay-Server** (`{{RELAY_HOSTNAME}}`, statische IP bei Netcup) und einem **Heimserver** (`{{HOME_HOSTNAME}}`, dynamische IP via deSEC-DynDNS).

Eingehende Mails werden vom Relay empfangen, gefiltert und an den Heimserver weitergeleitet. Ausgehende Mails werden vom Heimserver signiert (OpenDKIM) und über den Relay ins Internet zugestellt.

---

## 📘 Inhalt

1. **Einleitung**
   - [Problemstellung](00_Einleitung/00_problem.md) – Warum ein Relay mit Heimserver?
   - [Wann diese Lösung nicht geeignet ist](00_Einleitung/01_wann_nicht_geeignet.md)
   - [Umfang dieser Dokumentation](00_Einleitung/02_scope.md)
   - [Begriffe und Komponenten](00_Einleitung/03_glossar.md)

2. **Planung**
   - [Voraussetzungen](01_Planung/04_voraussetzungen.md) – Hardware, Ports, Accounts
   - [DNS Setup (deSEC)](01_Planung/05_dns_setup.md) – Domain, DNSSEC, Grundeinrichtung
   - [Registrar und DNS-Delegation](01_Planung/05b_registrar_dns.md) – Gandi, Custom NS, deSEC-API-Token

3. **Infrastruktur**
   - [Relay-Server einrichten](02_Infrastruktur/06_relay_server.md) – Postfix, Rspamd, ClamAV auf `{{RELAY_HOSTNAME}}`
   - [Heimserver einrichten](02_Infrastruktur/07_home_server.md) – Postfix, Dovecot, OpenDKIM auf `{{HOME_HOSTNAME}}`

4. **Konfiguration**
   - [DNS Mail-Records](03_Konfiguration/08_dns_mail_records.md) – MX, SPF, DKIM, DMARC bei deSEC
   - [PostfixAdmin und MySQL](03_Konfiguration/08b_postfixadmin_mysql.md) – Mailboxverwaltung, virtuelle Domains
   - [DKIM einrichten](03_Konfiguration/09_dkim.md) – OpenDKIM auf dem Heimserver, Selektor `default`
   - [DMARC konfigurieren](03_Konfiguration/10_dmarc.md) – Policy, Reporting, Eskalation
   - [TLS für IMAP und SMTP](03_Konfiguration/11_tls_imap_smtp.md) – Let's Encrypt für `{{HOME_SMTP}}` und `{{HOME_IMAP}}`
   - [Spamfilter & Virenschutz](03_Konfiguration/12_spamfilter.md) – Rspamd, ClamAV, Greylisting

5. **Betrieb**
   - [Automatisierung](04_Betrieb/13_automatisierung.md) – DynDNS, DANE/TLSA, Certbot-Hooks, Backups
   - [Betrieb und Wartung](04_Betrieb/14_betrieb_und_wartung.md) – Routinen, Updates, Monitoring
   - [Troubleshooting](04_Betrieb/15_troubleshooting.md) – DNS, Logs, Rspamd, DKIM, Verbindungstest

6. **Referenz**
   - [Config Library](05_Referenz/config_library.md) – alle Konfigurationsdateien und Scripts
   - [Variablen und Platzhalter](00_Einleitung/00_variablen.md) – alle Platzhalter mit deinen Werten

---

## 🔄 Schnellnavigation

[→ Start bei der Problemstellung](00_Einleitung/00_problem.md)

