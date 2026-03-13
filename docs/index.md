
# 📨 Self-Hosted Mailserver - Provider Grade

Diese Dokumentation beschreibt die Implementierung einer professionellen Mail-Infrastruktur, die auf maximale Zustellbarkeit und Sicherheit optimiert ist. Ziel ist ein reibungsloser Mail-Austausch, der typische Hürden wie Blacklisting oder mangelnde Domain-Reputation durch ein durchdachtes Architektur-Design umgeht.

**Das Architektur-Konzept** Das Design basiert auf einer funktionalen Trennung:

- **Public Relay (Statische IP):** Ein schlanker Server bei einem Hoster, der als primärer Gateway für ein- und ausgehende E-Mails dient.
    
- **Home Server (Dynamische IP):** Der lokale Kern der Infrastruktur, der die tatsächlichen Postfächer verwaltet und die Datenhoheit behält.
    

**Sicherheits- & Validierungsstandards** Um den „Provider Grade“-Status zu erreichen, deckt diese Anleitung die Konfiguration folgender Standards ab:

- **Authentifizierung:** SPF, DKIM und DMARC gegen Spoofing.
    
- **Verschlüsselung & Vertrauen:** DANE und Let's Encrypt Zertifikate.
    
- **Resilienz:** Greylisting, Anti-Spam- und Anti-Virus-Integration.
    

**Genutzte Dienste** Für die Umsetzung werden beispielhaft **Gandi** (Registrar), **deSEC** (DNS mit Fokus auf Sicherheit), **NoIP** (DynDNS) sowie **Let's Encrypt** verwendet.



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

