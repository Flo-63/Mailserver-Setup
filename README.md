# Self-Hosted Mailserver mit Relay

Anleitung für einen vollständigen, selbst gehosteten Mailserver mit dynamischer Heimnetz-IP und einem externen Relay-Server.

## Inhalt

- Architektur: Relay-Server (statische IP) + Heimserver (dynamische IP)
- Postfix, Dovecot, Rspamd, ClamAV, OpenDKIM
- DNS, DKIM, DMARC, MTA-STS, DANE/TLSA
- Firewall, GeoIP-Blocking, Blacklisting
- Automatisierung, Betrieb, Troubleshooting
- Vollständige Config Library mit allen Konfigurationsdateien und Scripts

## Dokumentation lesen

→ **[https://flo-63.github.io/Mailserver-Setup](https://flo-63.github.io/Mailserver-Setup)**

## Mitmachen & Feedback

Fragen, Verbesserungsvorschläge oder Erfahrungsberichte sind willkommen – am liebsten als [GitHub Discussion](https://github.com/Flo-63/Mailserver-Setup/discussions).

Aktuell läuft eine **Abstimmung**: [Soll ein Ansible-Playbook für dieses Setup entstehen?](https://github.com/Flo-63/Mailserver-Setup/discussions/2)

---

## Lokal bauen

```bash
pip install mkdocs-material
mkdocs serve
```

Dann im Browser: `http://localhost:8000`
