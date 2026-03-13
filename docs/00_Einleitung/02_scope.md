
# Umfang dieser Anleitung

Diese Dokumentation beschreibt den Aufbau einer konkreten Architektur für einen selbst betriebenen Mailserver mit dynamischer IP-Adresse. Der Fokus liegt auf einer praktischen, reproduzierbaren Umsetzung – nicht auf allgemeiner Theorie.

---

## Was diese Anleitung beschreibt

- Architektur mit Relay-Server und Heimserver
- Einrichtung aller benötigten DNS-Einträge
- Konfiguration von Postfix und Dovecot
- DNSSEC, SPF, DKIM, DMARC
- TLS-Zertifikate mit Let's Encrypt
- Grundlegende Automatisierung für den laufenden Betrieb

Die Anleitung ist als **Schritt-für-Schritt-Umsetzung** aufgebaut und beschreibt eine konkrete, selbst betriebene Referenzimplementierung.

---

## Was diese Anleitung nicht erklärt

Diese Dokumentation ist **kein allgemeiner Leitfaden zu E-Mail-Infrastruktur**. Folgende Themen werden vorausgesetzt:

- Grundlagen des DNS
- Grundlagen des SMTP-Protokolls
- Funktionsweise von SPF, DKIM und DMARC im Detail
- Allgemeine Linux-Systemadministration
- Allgemeine Netzwerkkonfiguration

Für diese Themen existieren umfangreiche externe Dokumentationen.

---

## Ziel

Ziel ist eine **nachvollziehbare und reproduzierbare Referenzimplementierung** – mit konkreten Entscheidungen, konkreter Konfiguration und Erfahrungen aus dem praktischen Betrieb.

---

## ✅ Ergebnis

Nach diesem Kapitel ist klar, was diese Anleitung abdeckt und was sie bewusst ausspart.

---

## 🔁 Navigation

**← Zurück:** [Wann diese Lösung nicht geeignet ist](../00_Einleitung/01_wann_nicht_geeignet.md)  
**→ Weiter:** [Begriffe und Komponenten](../00_Einleitung/03_glossar.md)

