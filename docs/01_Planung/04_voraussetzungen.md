
# Voraussetzungen

Vor Beginn müssen folgende Komponenten vorhanden und zugänglich sein.

---

## Domain

Eine eigene Domain ist erforderlich, z. B. `example.com`.

Die Domain wird bei einem Domain-Registrar verwaltet. In der Referenzarchitektur: **Gandi**.

> **Wichtige Anforderung an den Registrar:** Er muss das Setzen eigener Nameserver erlauben (Custom NS / „Bring your own DNS"). Nicht alle Registrare unterstützen das. Bei Gandi ist es möglich.

Der Registrar wird im weiteren Verlauf nur für zwei Aufgaben benötigt:

- Nameserver auf deSEC umstellen
- DNSSEC DS-Record hinterlegen

Details zur Einrichtung: [Registrar und DNS-Delegation](../01_Planung/05b_registrar_dns.md)

---

## DNS Provider

Ein DNS-Provider mit DNSSEC-Unterstützung und API-Zugang wird benötigt.

In diesem Setup: **deSEC.io**

Relevante Funktionen:

- DNSSEC-Unterstützung
- REST-API für automatisierte DNS-Änderungen (DynDNS, Zertifikats-Hooks)
- Verwaltung aller Mail-relevanten Records (MX, SPF, DKIM, DMARC)

---

## Relay-Server

Ein VPS mit **statischer öffentlicher IP-Adresse** wird benötigt.

In diesem Setup: **Netcup VPS**

Mindestanforderungen:

| Ressource | Minimum |
|---|---|
| CPU | 1 vCPU |
| RAM | 1 GB |
| OS | Debian 12 oder Ubuntu 24.04 LTS |

Der Relay-Server übernimmt:

- SMTP-Empfang aus dem Internet (Port 25)
- SMTP-Versand ins Internet
- Spamfilterung (Rspamd, Postgrey)
- Weiterleitung zum Heimserver

> **Wichtig:** Der Hoster muss Port 25 für ausgehende Verbindungen freigeben. Bei einigen Providern ist das standardmäßig gesperrt und muss explizit beantragt werden, was allerdings einige Provider (z.B. Hetzner) ablehnen. Bei NetCup ist Port 25 frei nutzbar.

---

## Heimserver

Der Heimserver läuft im Heimnetz und speichert die E-Mails.

Anforderungen:

- Linux-System (Debian oder Ubuntu empfohlen)
- Dauerhaft laufend und mit dem Internet verbunden
- Router (Fritz!Box) mit Portweiterleitung für folgende Ports:

| Port | Zweck |
|---|---|
| 2525 | SMTP vom Relay-Server (eingehende Mails) |
| 465 / 587 | Submission für Mailclients |
| 993 | IMAPS |
| 22 | SSH |

> Port 25 wird auf dem Heimserver **nicht** benötigt und sollte nicht freigegeben werden. Eingehende Mails kommen ausschließlich vom Relay über Port 2525. Ausgehende Mails laufen über den Relay-Server.

---

## DynDNS

Da Heimanschlüsse dynamische IP-Adressen haben, muss die aktuelle IP automatisch im DNS aktualisiert werden.

In diesem Setup laufen zwei parallele DynDNS-Mechanismen:

- **No-IP Client** – aktualisiert `{{HOME_HOSTNAME}}.ddns.net`
- **deSEC-API-Skript** – aktualisiert `{{HOME_SMTP}}` und `{{DOMAIN}}` direkt als A-Records

Die Automatisierung wird in [Automatisierung](../04_Betrieb/13_automatisierung.md) beschrieben.

---

## Software

| Komponente | Software | Server |
|---|---|---|
| Mail Transfer Agent | Postfix | beide |
| IMAP Server | Dovecot | Heimserver |
| Spamfilter | Rspamd | Relay |
| Virenscan | ClamAV | Relay |
| DKIM-Signierung | OpenDKIM | Heimserver |
| Mailboxverwaltung | PostfixAdmin + MySQL | Heimserver |

---

## Grundkenntnisse

Diese Anleitung setzt voraus:

- Linux-Administration (Pakete installieren, Dienste starten, Logs lesen)
- SSH-Zugang zu Servern
- Grundverständnis von DNS
- Grundverständnis von SMTP und IMAP

---

## ✅ Ergebnis

Nach diesem Kapitel sind alle benötigten Komponenten bekannt und die Entscheidungen für Registrar, DNS-Provider und Hoster sind getroffen.

---

## 🔁 Navigation

**← Zurück:** [Begriffe und Komponenten](../00_Einleitung/03_glossar.md)  
**→ Weiter:** [DNS Setup](../01_Planung/05_dns_setup.md)

