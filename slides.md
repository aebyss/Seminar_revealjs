# File Synchronization mit jextract & libcurl

A Native Java Integration Projekt
**Edin Fidoris**  
Ostfalia University of Applied Sciences  
*Seminar Presentation – Sommersemester 2025*

---

## Motivation & Problemstellung

- Java hat keine moderne WebDAV-Client-Bibliothek mit voller Methodenableistung (z. B. `PROPFIND`)
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
  void upload_file(const char* path, const char* target_url);
  void list_folders(const char* url);
```
Note:
Hier sehen wir den Header sync.h, den ich für meine Anwendung geschrieben habe.
Er enthält 6 native Funktionen: Hier zeige ich nur eine zum Hochladen von Dateien per libcurl, und eine zum Abfragen von Verzeichnissen via PROPFIND
Das Besondere ist: Dieser Header ist so geschrieben, dass jextract ihn ohne Probleme verarbeiten kann – also nur C-Standardtypen, keine Makros, keine Pointer-Arithmetik, kein komplexes Preprocessing.
Genau das braucht jextract, um eine Java-API daraus zu generieren.

---

## jextract - von Header zu Java (CLI)


---

## Titel: „Generierung der Java-Bindings“

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
In dieser Folie zeige ich den vollständigen Build-Prozess – also wie ich meine C-Funktionen in eine Java-API überführe.

Zuerst die obere Zeile:
Hier verwende ich clang, um meine native Bibliothek libsync.so zu erzeugen. Das -shared-Flag erstellt eine Shared Library, und -fPIC sorgt dafür, dass der Code positionsunabhängig ist – was für dynamische Bibliotheken notwendig ist.
Mit -lcurl und -lxml2 linke ich gegen libcurl und libxml2, weil mein C-Code HTTP-Uploads und XML-Verarbeitung nutzt – zum Beispiel bei PROPFIND.

Wichtig: Diese .so ist später das eigentliche Ziel der Java-Aufrufe. Ohne sie würden die generierten Methoden ins Leere laufen.

Der zweite Block zeigt den Aufruf von jextract.
Hier wird meine Header-Datei sync.h analysiert und daraus wird automatisch Java-Code erzeugt.

Einige wichtige Optionen:

--library sync gibt an, dass die native Bibliothek libsync.so heißt – das braucht jextract, um die Methoden korrekt zu verlinken.
wie__ fragen an chatgpt gibt es eine mölichkeit nach unterschiedlichen platformen zu sorteiren
Unter windows soll das naders sein

-I include/ gibt den Pfad zu den Header-Dateien an – damit Clang alle Typen auflösen kann.

--use-system-load-library sorgt dafür, dass ich später in Java einfach System.loadLibrary("sync") nutzen kann – ohne Pfadmanipulation.

--output und --target-package bestimmen, wohin der generierte Java-Code geschrieben wird und in welches Java-Paket die Klasse kommt.

Am Ende entsteht eine Java-Klasse – bei mir heißt sie Sync.java – mit Methoden wie upload_file(...), die direkt mit dem nativen C-Code verbunden sind, aber über MemorySegment und ohne JNI funktionieren.

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

## Das Ergebniss:  Automatische Java-API

- Automatisch erzeugt durch jextract
- Enthält Methoden für jede C-Funktion im Header
- Nutzt `MemorySegment` für sicheren Zugriff auf nativen Speicher
- Führt über `MethodHandle` direkt zur .so-Funktion

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
Nach dem Aufruf von jextract entsteht unter anderem diese Datei: `sync_h.java`.  
Sie enthält eine Java-Methode für jede Funktion aus meinem Header `sync.h`.  
„Hier sehen wir, wie jextract aus der C-Funktion upload_file(...) eine nutzbare Java-Methode erzeugt hat.
Die Parameter sind MemorySegments – das ist eine sichere und kontrollierte Art, mit nativen Pointern in Java zu arbeiten.
Im Hintergrund steckt ein MethodHandle, der direkt auf libsync.so zeigt.
Wenn ich diese Methode in Java aufrufe, wird ohne JNI der C-Code ausgeführt – synchron und effizient.
Die Zeile mit invokeExact(...) ist also der Moment, in dem Java die Kontrolle an C übergibt.

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

  Durch diese Kombination kann ich moderne Java-Tools mit nativen Bibliotheken verbinden – ohne JNI, aber mit REST.



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

## Architektur: Wie alles zusammenspielt

- Java-Code ruft `sync_h.upload_file(...)`
- jextract leitet Aufruf an `libsync.so` weiter
- libsync.so nutzt `libcurl` für HTTP/WebDAV
- Ziel ist z. B. eine Nextcloud-Instanz

Note:
Diese Architektur ist leichtgewichtig, systemnah und portierbar – und braucht keine separate Java-WebDAV-Library.


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