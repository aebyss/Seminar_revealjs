# File Synchronization mit jextract & libcurl

A Native Java Integration Projekt
**Edin Fidoris**  
Ostfalia University of Applied Sciences  
*Seminar Presentation – Sommersemester 2025*

Note:
Hallo zusammen,  
ich freue mich, heute mein Projekt vorzustellen, bei dem ich mithilfe von **jextract** eine Brücke zwischen **Java** und **C** gebaut habe.

Im Zentrum steht eine kleine native Bibliothek, mit der ich Dateien direkt per **WebDAV auf meine Nextcloud** hochladen kann –  

Dabei ging es nicht nur um Technik, sondern auch darum, ein System zu schaffen, das **leichtgewichtig**, **schnell** und **gut wartbar** ist.

Ich zeige heute Schritt für Schritt, wie jextract funktioniert, wo seine Stärken liegen –  
und auch, an welchen Stellen es (noch) an Grenzen stößt.


---

## Inhaltsverzeichnis

1. Motivation & Ziel
2. Architektur & Komponenten
3. Die Rolle von jextract
4. Native Bindings & Code-Generierung
5. Einschränkungen & Fallstricke
6. Fazit & Ausblick

Note:
In dieser Präsentation führe ich euch durch mein Projekt zur nativen Interaktion zwischen Java und C mit jextract.  
Wir beginnen mit der Motivation, schauen uns die Architektur an, steigen dann in jextract ein,  
und besprechen die praktischen Erfahrungen – inklusive Herausforderungen.  
Zum Schluss gibt’s ein Fazit und einen kurzen Ausblick, wohin es noch gehen kann.


---

## Motivation & Problemstellung

- Java hat veraltete WebDAV-Client-Bibliotheken die z.B. (z. B. `PROPFIND`)
- libcurl ist extrem performant und unterstützt WebDAV vollständig – aber in C
- JNI ist komplex und fehleranfällig – besonders bei Authentifizierung, Streams und Pointern

Note:
WebDAV ist ein HTTP-basiertes Protokoll, das z. B. von Nextcloud verwendet wird.  
In Java gibt es zwar Bibliotheken wie Sardine, aber die unterstützen moderne Methoden wie `PROPFIND` oder `MKCOL` oft unvollständig oder schlecht dokumentiert.  
Mit libcurl in C kann ich beliebige WebDAV-Kommandos direkt absetzen, z. B. `PROPFIND` für Ordnerlisten oder `PUT` für Uploads.  
Aber: Ohne jextract müsste ich für den Zugriff auf libcurl JNI schreiben – und das ist umständlich und fehleranfällig.

Man kann’s machen – aber es ist teuer in Wartung und Entwicklung.


---


## Ziel & Lösungsansatz

- Ziel: C-Funktionen (z. B. libcurl) **einfach aus Java verwenden**
- Kein JNI schreiben oder pflegen müssen
- Lösung: **jextract** generiert Java-Bindings automatisch aus C-Headern

Note:
Deshalb habe ich mich gefragt:
Gibt es eine Möglichkeit, C-Code wie libcurl in Java zu nutzen – aber ohne den Aufwand von JNI?
👉 Und genau hier kommt jextract ins Spiel.

Nach dem wir gesehen haben, warum JNI problematisch ist, kommt die Frage: Wie kann man native Bibliotheken trotzdem in Java nutzen?  
Hier kommt jextract ins Spiel. Es analysiert C-Header-Dateien und erzeugt automatisch eine saubere, typisierte Java-API, ohne dass man JNI schreiben muss.  
Das reduziert die Komplexität erheblich und ist ideal für Integration von z. B. libcurl oder anderen C-Libs.

---
## Was ist jextract?

- Teil des Panama-Projekts (Java FFI)
- Generiert Java-API aus C-Header-Dateien
- Nutzt `libclang`, um Typen und Signaturen korrekt zu parsen
- Ergebnis: idiomatische Java-Klassen

Note:
jextract gehört zum Panama-Projekt, das Java direkten Zugriff auf nativen Code ermöglichen soll – ohne JNI.  
Statt händisch JNI zu schreiben, übernimmt jextract das: Es parst die Header mit libclang und erzeugt sofort nutzbare Java-Klassen.  
Dazu gehören Funktionen, Structs, Enums usw.

---
## sync.h → Sync.java

```c {highlightLines="2"}
  // sync.h
  #ifndef SYNC_H
  #define SYNC_H

  void upload_file(const char* path, const char* target_url);
  void list_folders(const char* url);

  #endif
```
Note:
Hier sehen wir den Header sync.h, den ich für meine Anwendung geschrieben habe.
Er enthält 6 native Funktionen: Hier zeige ich nur eine zum Hochladen von Dateien per libcurl, und eine zum Abfragen von Verzeichnissen via PROPFIND
Das Besondere ist: Dieser Header ist so geschrieben, dass jextract ihn ohne Probleme verarbeiten kann – also nur C-Standardtypen, keine Makros, keine Pointer-Arithmetik, kein komplexes Preprocessing.
Genau das braucht jextract, um eine Java-API daraus zu generieren.

---

## Generierung der Java-Bindings

```bash
# Kompilieren der C-Bibliothek (libsync.so)
clang -shared -fPIC -o libsync.so sync.c -lcurl -lxml2

# jextract erzeugt Java-Bindings aus sync.h
jextract \
  --library sync \
  -I include/ \
  --use-system-load-library \
  --output src/main/java \
  --target-package com.nativecloud.sync \
  include/sync.h
```

Note:
Diese Folie zeigt den vollständigen Build-Prozess: Wie aus C-Code eine nutzbare Java-API entsteht.

Zuerst wird mit clang die native Bibliothek libsync.so erzeugt.
Das -shared-Flag erzeugt eine dynamische Bibliothek (Shared Library), -fPIC macht sie positionsunabhängig.
Zusätzlich wird gegen libcurl und libxml2 gelinkt, weil mein Code HTTP- und XML-Funktionen nutzt – etwa für PROPFIND.

Der zweite Block zeigt den jextract-Aufruf.
Dieser analysiert die Header-Datei sync.h und erzeugt automatisch Java-Bindings.
Wichtig sind dabei folgende Optionen:

--library sync: Der Name der nativen Bibliothek (ohne lib und .so), also z. B. libsync.so unter Linux.
Unter Windows müsste die Datei z. B. sync.dll heißen. Die Plattform entscheidet, wie der Name intern aufgelöst wird.

--use-system-load-library: Sorgt dafür, dass später System.loadLibrary("sync") funktioniert – etwa wenn die .so oder .dll im java.library.path liegt.

-I include/: Gibt den Pfad zur Header-Datei an – notwendig für korrekte Typauflösung durch libclang.

--output und --target-package: Bestimmen Zielverzeichnis und Java-Paket für die generierte Klasse (z. B. com.nativecloud.sync.sync_h).

Plattformunterschiede beachten:
Unter Linux muss die Bibliothek libsync.so heißen.
Unter Windows wäre der Name z. B. sync.dll, und jextract würde automatisch System.loadLibrary("sync") verwenden – ohne Präfix.

Die erzeugte Java-Klasse enthält Methoden wie upload_file(...), die intern mit MemorySegment und MethodHandle arbeiten – ganz ohne JNI.

Wenn man das systemübergreifend einsetzen will (Linux + Windows), sollte man verschiedene Builds für .so und .dll erzeugen – oder den genauen Library-Pfad zur Laufzeit setzen.

Kurz zur positionunabhänigkeit
Normaler Code hat feste Adressen („absolute Adressierung“).
→ z. B. mov eax, [0x00401000]
Das funktioniert nur, wenn der Code genau dort im Speicher liegt.
Positionunabhängiger Code (PIC, position-independent code) verwendet relative Adressen
→ z. B. mov eax, [ebx + offset]
 Dadurch ist der Code flexibler, weil er egal wo im Speicher geladen werden kann.

 Warum ist das wichtig?
Für Shared Libraries wie libsync.so:
Mehrere Programme können dieselbe .so verwenden.

Das Betriebssystem lädt sie an beliebige Speicheradressen.

Dafür muss der Code positionunabhängig sein.

Das wird mit dem Compiler-Flag -fPIC aktiviert.
---
## Die Rolle von Clang

- jextract ist kein eigener Compiler
- Es nutzt **Clang**, um C-Header-Dateien zu analysieren
- Clang verarbeitet Includes, Typen, Makros und erzeugt eine **AST**
- Diese AST ist die Grundlage für die generierten Java-Bindings

Note:
Hier erkläre ich, dass Clang in zwei Rollen eine zentrale Funktion hat:  
Erstens nutze ich Clang als ganz normalen C-Compiler, um aus meinem C-Code die Bibliothek `libsync.so` zu erstellen – ohne diese Datei würden die generierten Java-Bindings ins Leere laufen.  
Zweitens verwendet jextract intern `libclang`, also die Clang-Bibliothek, um meine Header-Dateien zu analysieren. Dabei wird kein Code kompiliert, sondern nur die Struktur und die Typen des C-Headers ausgelesen.  
Ohne Clang – oder eine korrekt installierte `libclang`-Umgebung – funktioniert jextract nicht.  
Es ist also ein technischer Schlüsselbestandteil im Hintergrund, auch wenn ich es nicht direkt aufrufe.

---

## Vom Header zur Java-API

sync.h
↓
Clang (erstellt AST)
↓
jextract (generiert Java-Code)
↓
Sync.java (MemorySegment-basierte API)


- Clang ist für das **Verstehen des C-Codes** zuständig
- jextract wandelt die Struktur in Java-Klassen um

Note:
Diese Grafik verdeutlicht, dass Clang den Header zuerst analysiert und daraus einen Syntaxbaum baut. jextract nimmt dann diesen Baum und erzeugt darauf basierend Java-Code. Wenn Clang den Header nicht versteht, scheitert auch jextract.

---

## Das Ergebnis: Automatische Java-API

- Automatisch erzeugt durch `jextract`
- Enthält Methoden für jede C-Funktion im Header
- Nutzt `MemorySegment` für sicheren Zugriff auf nativen Speicher
- Führt über `MethodHandle` direkt zur `.so`-Funktion

Note:
Nach dem Aufruf von `jextract` entsteht z. B. die Datei `sync_h.java`.  
Sie enthält eine Java-Methode für jede Funktion aus meinem Header `sync.h`.  
Die Parameter sind `MemorySegment`s – das ist eine sichere Art, mit nativen Pointern in Java zu arbeiten.  
Das Ganze wird zur direkten Java-API auf meine native Bibliothek.

---

## Beispiel: Generierte Methode


```java
//Sync.java
public static int upload_file(MemorySegment local_path, MemorySegment nextcloud_url, MemorySegment username, MemorySegment password) {
    var mh$ = upload_file.HANDLE;
    try {
        if (TRACE_DOWNCALLS) {
            traceDowncall("upload_file", local_path, nextcloud_url, username, password);
        }
        return (int)mh$.invokeExact(local_path, nextcloud_url, username, password);
    } catch (Throwable ex$) {
       throw new AssertionError("should not reach here", ex$);
    }
}
```

Note:
Was wir hier sehen, ist die von jextract erzeugte Java-Methode für die native C-Funktion `upload_file(...)`.

Die zentrale Rolle spielt der sogenannte `MethodHandle`, hier in der Zeile `var mh$ = upload_file.HANDLE`.

Dieser `MethodHandle` ist ein direkter Java-Zeiger auf die C-Funktion – er wurde beim Start der Anwendung erzeugt, mit Hilfe des Foreign Function & Memory APIs.

Wenn ich dann `mh$.invokeExact(...)` aufrufe, wird die native Funktion direkt aufgerufen – inklusive Übergabe der Parameter als `MemorySegment`.2

Optional kann ich mit dem `TRACE_DOWNCALLS`-Flag jeden Funktionsaufruf zur Laufzeit debuggen.
---

## Integration in Quarkus


```java
// Quarkus-REST: Datei-Upload mit native Aufruf
@Path("/upload")
@POST
@Consumes(MediaType.MULTIPART_FORM_DATA)
public Response uploadFile(@MultipartForm UploadForm form) {
    try (Arena arena = Arena.ofConfined()) {
        var file = arena.allocateUtf8String(form.filePath);
        var url  = arena.allocateUtf8String(form.targetUrl);
        var user = arena.allocateUtf8String(form.username);
        var pass = arena.allocateUtf8String(form.password);
        int res = sync_h.upload_file(file, url, user, pass);
        return Response.ok("Upload result: " + res).build();
    }
}
```
Note:
  In meinem Projekt habe ich die durch jextract generierte Java-Klasse `sync_h` in einen Quarkus-Service integriert.

  Quarkus bietet mir eine leichtgewichtige, performante Runtime für REST-Endpunkte.  
  Hier sieht man ein Beispiel: ein POST-Endpunkt `/upload`, der Multipart-Daten entgegen nimmt, in `MemorySegment`s umwandelt, und dann die native Funktion `upload_file(...)` aufruft.

  Quarkus eignet sich besonders gut für solche systemnahen Tools, weil es schnell startet, wenig RAM braucht und sich gut in DevOps-Prozesse integrieren lässt – ideal auch für Raspberry Pi.

---

## Java ruft C auf – mit MemorySegment

```java
@ApplicationScoped
public class NextcloudNativeClient {
    public int upload(String localPath, String remoteUrl, String username, String password) {
        try (Arena arena = Arena.ofConfined()) {
            MemorySegment local = arena.allocateUtf8String(localPath);
            MemorySegment remote = arena.allocateUtf8String(remoteUrl);
            MemorySegment user = arena.allocateUtf8String(username);
            MemorySegment pass = arena.allocateUtf8String(password);

            return upload_file(local, remote, user, pass);
        }
    }
}
```
Note:
Diese Java-Methode wurde automatisch von jextract generiert.  
Sie erwartet `MemorySegment`-Objekte als Eingabe – das ist Teil der Foreign Function & Memory API von Java.

`Arena.ofConfined()` legt einen eigenen, kontrollierten Speicherbereich an.  
Damit wird sichergestellt, dass alle native Strings automatisch wieder freigegeben werden – kein Memory Leak.

Mit `allocateUtf8String(...)` werden normale Java-Strings in UTF-8-codierten Speicherblöcken abgelegt.  
Diese Speichersegmente werden dann direkt an die C-Funktion übergeben – in diesem Fall `upload_file(...)`.

Der Aufruf ist synchron und sicher – ganz ohne JNI.

---

## Ergebnis: Funktionierender Dateiupload

- Upload von Dateien zu Nextcloud über C + libcurl
- Aufruf aus Java über jextract-Bindings
- Läuft lokal und auf dem Raspberry Pi

---

## Architektur

![Architekturdiagramm](arch.png)
- Java-Code ruft `sync_h.upload_file(...)`
- jextract leitet Aufruf an `libsync.so` weiter
- libsync.so nutzt `libcurl` für HTTP/WebDAV
- Ziel ist z. B. eine Nextcloud-Instanz

Note:
Hier sieht man schön den Ablauf von Java bis zur Nextcloud-Instanz.  
Jede Komponente erfüllt eine klare Aufgabe: Java ruft die Methode auf, jextract gibt sie an libsync.so weiter,  
dort wird libcurl genutzt, um HTTP/WebDAV-Requests an Nextcloud zu senden.

---

## Einschränkungen von jextract

- **Funktions-Makros** wie `#define MIN(a,b)`  
  → werden **ignoriert**, keine Java-Methode
- **Bitfields** in `structs`  
  → werden **übersprungen**, keine Feld-Extraktion

Note:
Auch wenn jextract sehr mächtig ist, stößt man bei der praktischen Arbeit schnell auf ein paar Grenzen.  
Gerade bei nativen C-Headern, wie man sie oft in älteren Bibliotheken oder Systemkomponenten findet, kommen Features vor, die jextract aktuell nicht verarbeiten kann.

Ich zeige euch zwei typische Beispiele, die mir während der Entwicklung selbst begegnet sind – und wie ich damit umgegangen bin.

---
### Funktions-Makro

```c
#define MIN(a, b) ((a) < (b) ? (a) : (b))
```
##### Alternative
```c
int min_int(int a, int b) {
    return (a < b) ? a : b;
}
```
Note:
In dieser Folie zeige ich, welche Dinge jextract nicht unterstützt.
Zuerst das bekannte Makro-Problem: #define MIN(a,b) ist zur Compile-Zeit aktiv, aber verschwindet im Preprocessor. Deshalb erkennt jextract es nicht – wir müssen echte C-Funktionen schreiben, wenn wir sie binden wollen.

Dann Bitfields – sie sind in C nützlich zur Platzersparnis, aber das Memory-Layout ist je nach Compiler und Plattform verschieden. Deshalb kann jextract sie nicht zuverlässig extrahieren. Auch hier hilft eine Umwandlung in normale int-Felder, wenn man sie in Java nutzen will.

Ziel dieser Folie: Bewusstsein schaffen, was nicht geht – und wie man trotzdem zum Ziel kommt.
---

### Bitfields

```c
struct Flags {
    unsigned int read  : 1;
    unsigned int write : 1;
    unsigned int exec  : 1;
};
```
*Ausgabe: Skipping Flags.read (bitfields are not supported)*

##### Alternative
```c
struct Flags {
    unsigned int read;
    unsigned int write;
    unsigned int exec;
};
```

Note:
Bitfields sparen Speicher, sind aber plattformabhängig im Layout. jextract überspringt sie, weil das Layout nicht garantiert nachgebildet werden kann. Nutze lieber einfache int-Felder, wenn du plattformunabhängig arbeiten willst.
---

## Demo / Code Snippets

void upload_file(const char* local_path, const char* url);

---

## Result & Learnings

- jextract ist nutzbar aber low-level C soll sauber und gut definiert sein
- Debugging vom native code ist immer noch sehr tricky
- Performance besser als Sardine in diesem Beispiel
- "Echtzeitsync" mit Raspberry-Pi funktioniert sehr gut mit kleinen Dateien

---

## Herausforderungen

- C code muss sehr strikt nach angaben von jextract definiert werden
- WebDAV quirks (e.g., PROPFIND) benötigt low-level handling
- Containerisierug von libsync.so war nicht trivial

Note:
In diesem Projekt war es entscheidend, den C-Code so zu schreiben, dass jextract ihn überhaupt verarbeiten kann.  
Das heißt: keine Makros mit Logik, keine Bitfields, keine undurchsichtigen Structs.

Ein zweiter Punkt war der Umgang mit WebDAV – speziell der PROPFIND-Request ist sehr speziell und wird von den meisten High-Level-Clients nicht richtig unterstützt. Deshalb musste ich direkt mit libcurl arbeiten und z. B. die XML-Antwort manuell verarbeiten.

Daher habe ich entschieden, die Bibliothek **zuerst lokal zu testen** – und die Containerisierung erst dann zu machen, wenn das was wichtig ist erstmal gut läuft.


---

## Fazit & Ausblick

- jextract ermöglicht effiziente und sichere Interop mit nativen C-Libraries
- Ideal für Systemintegration mit Performance-Anforderungen
- Projekt hat Spaß gemacht und praxisnah viele Herausforderungen gezeigt
- Nächstes Ziel: Dateisynchronisation zwischen Geräten mit ZeroMQ + Nextcloud

Note:
Dieses Projekt war für mich nicht nur technisch spannend, sondern hat auch richtig Spaß gemacht.  
Ich habe tief in die Welt von jextract und dem Foreign Function & Memory API eingetaucht und dabei gesehen, wie sauber und performant native Interop heute sein kann – ganz ohne JNI.

Besonders motiviert hat mich ein persönlicher Moment: Ich wollte automatische Datei-Uploads von meinem Handy zu Nextcloud einrichten – aber Google blockiert diese Funktion aktiv.  
Das war der Moment, wo ich dachte: „Okay – ich baue es eben selbst.“

Daraus entstand mein eigener nativer Dateisynchronisationsdienst mit libcurl, jextract und Quarkus – der nächste Schritt wird die asynchrone Kommunikation mit ZeroMQ sein, um mehrere Geräte zu vernetzen.

Also: viel gelernt, viel gelacht, ein Problem gelöst – und das nächste wartet schon.
---

## Fragen