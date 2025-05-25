# File Synchronization mit jextract & libcurl

A Native Java Integration Projekt
**Edin Fidoris**  
Ostfalia University of Applied Sciences  
*Seminar Presentation – Sommersemester 2025*

---

## Motivation & Problemstellung

- Java allein hat keine gute WebDAV-Client-Library (z.B. Sardine -> Langsam)
- libcurl ist extrem performant aber in C
- JNI ist komplex und fehleranfällig

---

## Lösung: Jextract für automatische Bindings ohne JNI

- WatchService detects file changes
- Native upload via jextract bindings
- REST API (Quarkus) for control

---

## sync.h to Sync.java

---

## Architektur

- WatchService detects file changes
- Native upload via jextract bindings
- REST API (Quarkus) for control

Note:
Hier erkläre ich, warum ich WatchService gewählt habe und wie der native Upload funktioniert.

---

## Demo / Code Snippets

void upload_file(const char* local_path, const char* url);

---

## Result & Learnings

- jextract ist nutzbar aber low-level C soll sauber und gut definiert sein
- Debugging vom native code ist immer noch sehr tricky
- Performance besser als Sardine in diesem Beispiel
- Echtzeitsync mit Raspberry-Pi funktioniert sehr gut mit kleinen Dateien

---

## Herausforderungen

- C code muss sehr strikt nach angaben von jextract definiert werden
- WebDAV quirks (e.g., PROPFIND) benötigt low-level handling
- Containerization von libsync.so und mounting in k3s war nicht trivial

---

## Fazit & Ausblick

- jextract ermöglicht produktive native Interop mit C

- Sehr hilfreich für Systemintegration, hohe Performance

- Nächstes Ziel: ZeroMQ für asynchrone Kommunikation zwischen Diensten (optional)

---

## Fragen