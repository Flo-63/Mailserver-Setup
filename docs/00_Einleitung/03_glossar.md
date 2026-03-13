
# Begriffe und Komponenten

Diese Seite erklärt zentrale Begriffe, die in dieser Dokumentation verwendet werden – bewusst kurz und praxisnah.

---

## Mail Transfer Agent (MTA)

Ein **Mail Transfer Agent** ist die Software, die E-Mails zwischen Mailservern überträgt.

Aufgaben eines MTA:

- Empfang von E-Mails über SMTP
- Weiterleitung an andere Mailserver
- Zustellung an lokale Mailboxen

In diesem Setup: **Postfix**

---

## Mail Relay

Ein **Mail Relay** ist ein Mailserver, der E-Mails entgegennimmt und an einen anderen Mailserver weiterleitet.

In dieser Architektur übernimmt der Relay-Server:

- Empfang von E-Mails aus dem Internet
- Weiterleitung an den Heimserver
- Versand ausgehender E-Mails

Der Relay befindet sich auf einem Server mit **statischer IP-Adresse**.

---

## Heimserver

Der Heimserver ist der eigentliche Mailserver des Systems. Er läuft im Heimnetz hinter einer dynamischen IP-Adresse.

Aufgaben:

- Speicherung der E-Mails
- Zugriff über IMAP
- Lokale Zustellung eingehender Mails

Er ist **nicht** direkt für SMTP-Kommunikation mit dem Internet verantwortlich – das übernimmt der Relay.

---

## SMTP

**SMTP (Simple Mail Transfer Protocol)** ist das Protokoll zur Übertragung von E-Mails zwischen Mailservern.

Relevante Ports:

| Port | Verwendung |
|---|---|
| 25 | Server-zu-Server (MX-Zustellung) |
| 587 | Submission (Mailclient → Mailserver) |
| 465 | SMTPS (veraltet, aber noch verbreitet) |

---

## SMTP Submission

**Submission** bezeichnet den Einlieferungsweg für Mailclients. Hierüber sendet ein Mailclient seine ausgehenden E-Mails an den eigenen Mailserver, der sie dann weiterleitet.

Port: **587** (mit STARTTLS und Authentifizierung)

Submission unterscheidet sich von Port 25: Port 25 ist für die Kommunikation zwischen Mailservern reserviert und wird von den meisten ISPs für Heimanschlüsse geblockt.

---

## IMAP

**IMAP (Internet Message Access Protocol)** ist das Protokoll, über das Mailclients auf Mailboxen zugreifen.

Im Gegensatz zu POP3 verbleiben E-Mails beim IMAP-Zugriff auf dem Server und können von mehreren Geräten genutzt werden.

Port: **993** (IMAPS, TLS)

In diesem Setup: **Dovecot**

---

## Milter

Ein **Milter (Mail Filter)** ist eine Schnittstelle, über die Postfix externe Filter einbinden kann. Filter können E-Mails prüfen, verändern oder ablehnen.

In diesem Setup werden Milter verwendet für:

- DKIM-Signierung (OpenDKIM)
- Spamfilterung (Rspamd)

---

## LMTP

**LMTP (Local Mail Transfer Protocol)** ist eine vereinfachte Variante von SMTP für die Übergabe von E-Mails innerhalb eines Servers – typischerweise von Postfix an Dovecot.

Anders als SMTP ist LMTP für lokale Zustellung optimiert und unterstützt keine Weiterleitung über das Internet.

---

## Maildir

**Maildir** ist ein Dateiformat zur Speicherung von E-Mails. Jede E-Mail wird als einzelne Datei in einer definierten Verzeichnisstruktur abgelegt.

Vorteile gegenüber dem älteren mbox-Format:

- Kein Locking bei gleichzeitigem Zugriff
- Einfachere Sicherung einzelner Mails
- Bessere Performance bei großen Postfächern

In diesem Setup: E-Mails werden unter `/var/vmail` im Maildir-Format gespeichert.

---

## DNS

Das **Domain Name System (DNS)** ordnet Domainnamen IP-Adressen zu.

Für Mailserver sind folgende DNS-Einträge relevant:

| Record | Zweck |
|---|---|
| MX | Mailserver einer Domain |
| A / AAAA | IP-Adresse eines Hostnamens |
| PTR | Reverse DNS (IP → Hostname) |
| TXT | SPF, DKIM, DMARC |

In dieser Architektur wird DNS über **deSEC.io** betrieben.

---

## PTR Record (Reverse DNS)

Ein **PTR Record** ordnet einer IP-Adresse einen Hostnamen zu – das Gegenteil eines A-Records.

Mailserver prüfen, ob der PTR-Record der sendenden IP-Adresse mit dem Hostnamen im SMTP-Banner übereinstimmt. Fehlendes oder falsch konfiguriertes Reverse DNS führt häufig zu Zustellproblemen.

Der PTR-Record wird beim Hoster (nicht beim DNS-Provider) konfiguriert.

---

## DNSSEC

**DNSSEC** erweitert DNS um kryptographische Signaturen. DNS-Einträge werden signiert und können von Empfängern verifiziert werden – Manipulation ist dadurch erkennbar.

In dieser Architektur ist DNSSEC über deSEC aktiviert.

---

## Dynamische IP-Adresse

Eine **dynamische IP-Adresse** wird regelmäßig vom Internetprovider geändert – bei Heimanschlüssen die Regel, nicht die Ausnahme.

Das in dieser Anleitung beschriebene Setup ist darauf ausgelegt, trotz dynamischer IP zuverlässig zu funktionieren: Der Relay-Server mit statischer IP übernimmt die Kommunikation mit dem Internet, der Heimserver ist nur intern erreichbar.

---

## ✅ Ergebnis

Nach diesem Kapitel sind alle zentralen Begriffe bekannt, die in den folgenden Kapiteln verwendet werden.

---

## 🔁 Navigation

**← Zurück:** [Umfang dieser Anleitung](../00_Einleitung/02_scope.md)  
**→ Weiter:** [Voraussetzungen](../01_Planung/04_voraussetzungen.md)

