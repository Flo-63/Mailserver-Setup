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

## Lokal bauen

```bash
pip install mkdocs-material
mkdocs serve
```

Dann im Browser: `http://localhost:8000`
