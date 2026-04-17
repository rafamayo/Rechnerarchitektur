# Übung 01 – Cache-Konflikte und Assoziativität

**Lehrveranstaltung:** Rechnerarchitektur  
**Thema:** Cache-Organisation und ihr Einfluss auf die Leistung  
**Werkzeug:** [Ripes RISC-V Simulator](https://github.com/mortbopet/Ripes)

---

## Motivation

Ein Cache ist nur dann effektiv, wenn die Daten, die ein Programm benötigt,
tatsächlich dort abgelegt werden können – und dort *bleiben*, bis sie wieder
gebraucht werden. Die **Kapazität** eines Caches allein ist kein Garant dafür.
Entscheidend ist auch die **Organisation**: Wie werden Adressen auf Cache-Sets
abgebildet? Können mehrere Daten denselben Platz belegen?

In dieser Übung untersuchen wir einen Fall, bei dem zwei Arrays mit einem
bestimmten Adressabstand dazu führen, dass sie sich im Cache **gegenseitig
verdrängen** – obwohl rechnerisch genug Platz vorhanden wäre. Wir nennen
das einen **Konflikt-Miss** (Conflict Miss).

### Relevanz in der Praxis

Konfliktmisses sind kein akademisches Konstrukt – sie treten in realen
Systemen regelmäßig auf und können die Laufzeit eines Programms um den
Faktor 2–10 verschlechtern, ohne dass der Fehler im Code selbst sichtbar ist.

**Betriebssystem-Kernel.**
Der Linux-Kernel hat historisch Konfliktmisses im Page-Cache erlebt,
wenn zwei Dateipuffer zufällig auf dieselben Cache-Sets abgebildet wurden.
Kernel-Entwickler verwenden deshalb gezielt `__attribute__((aligned(...)))`
und Padding zwischen häufig gemeinsam genutzten Datenstrukturen.

**Eingebettete Systeme und DSPs.**
Mikrocontroller und digitale Signalprozessoren (DSPs) verwenden häufig
direkt-abgebildete Caches, wegen der einfacheren Hardware und des
geringeren Energieverbrauchs. In der Audiocodierung (MP3, AAC) oder
in Motorsteuergeräten greifen Algorithmen auf Eingabe- und
Ausgabepuffer gleicher Größe zu. Liegen diese mit einem Abstand,
der einem Vielfachen der Cache-Größe entspricht, entsteht Thrashing, mit messbaren Latenzschwankungen in Echtzeitsystemen.

**Grafikprozessoren (GPUs) und Shader-Code.**
In GPU-Shadern werden Textur-Lookup-Tabellen und Ergebnispuffer oft
in separaten Speicherbereichen gleicher Größe abgelegt. Bei
Textur-Caches mit direkter Abbildung führt das zu Konflikten,
die Grafikentwickler durch explizites Speicher-Layout-Management
(z.B. Padding, `__restrict__`-Qualifier) vermeiden.

**Hashing und Hashtabellen.**
Hashtabellen mit Tabellengrößen, die Zweierpotenzen sind (256, 512, 1024 ...),
erzeugen systematisch Adressen, die sich auf wenige Cache-Sets konzentrieren.
Eine bekannte Faustregel in der Systemprogrammierung lautet deshalb:
*„Wähle Hashtabellen-Größen als Primzahlen, nicht als Zweierpotenzen"* –
einer der Gründe dafür ist die gleichmäßigere Cache-Nutzung.

---

## Hintergrund: Cache-Adressierung

Bei einem **direkt-abgebildeten Cache** (Assoziativität 1) hat jede
Speicheradresse genau einen möglichen Platz im Cache. Der Cache-Index
ergibt sich aus der Adresse wie folgt:

```
Cache-Index = (Byte-Adresse / Block-Größe) mod Anzahl-Lines
```

Mit 16 Lines und 16 Bytes pro Line (4 Wörter):

```
Cache-Index = (Adresse / 16) mod 16
```

Zwei Adressen, die sich um ein Vielfaches von `16 × 16 = 256 Bytes`
unterscheiden, landen auf **exakt demselben Cache-Index**.

Bei einem **2-fach assoziativen Cache** mit 8 Sets und 2 Ways:

```
Cache-Set = (Adresse / 16) mod 8
```

Jetzt hat jeder Set **zwei Plätze** (Ways). Zwei Adressen mit Abstand
256 Bytes landen im gleichen Set, können aber in verschiedenen Ways
abgelegt werden – kein Konflikt.

---

## Aufgabe 1 – Konflikt-Miss verstehen

### Szenario

Wir greifen abwechselnd auf zwei Arrays `X` und `Y` zu.
Der Abstand zwischen den Startadressen beträgt exakt **256 Bytes** –
gleich der Cache-Kapazität.

```
Cache-Konfiguration: 16 Lines, Assoziativität 1, 4 Wörter/Line
Cache-Kapazität:     16 × 16 Bytes = 256 Bytes
```

### Zugriffsmuster

```
Iteration 0:   lw X[0]   →  Cache-Index 0,  MISS, lade X[0..3]
               lw Y[0]   →  Cache-Index 0,  MISS, verdränge X[0..3]  ← !
Iteration 1:   lw X[1]   →  Cache-Index 0,  MISS, verdränge Y[0..3]  ← !
               lw Y[1]   →  Cache-Index 0,  MISS, verdränge X[0..3]  ← !
...
```

Obwohl der Cache 256 Bytes fasst und beide Arrays zusammen nur 128 Bytes
belegen, ist jeder einzelne Zugriff ein Miss.

### Visualisierung: direkt-abgebildet vs. 2-fach-assoziativ

```
Direkt-abgebildet (16 Lines, Assoz. 1):
┌────────────────────────────────────────────────────┐
│ Set  0 │ [X[0..3]]  →  [Y[0..3]]  →  [X[0..3]] ... │  ← Thrashing!
│ Set  1 │ [X[4..7]]  →  [Y[4..7]]  →  [X[4..7]] ... │  ← Thrashing!
│ Set  2 │ ...                                       │
│  ...   │                                           │
└────────────────────────────────────────────────────┘
Jeder Zugriff verdrängt den vorherigen → Miss-Rate ≈ 100%

2-fach-assoziativ (8 Sets, Assoz. 2, gleiche Kapazität):
┌──────────────────────────────────────────────────┐
│ Set 0 │ Way 0: [X[0..3]] │ Way 1: [Y[0..3]]      │   ← beide bleiben!
│ Set 1 │ Way 0: [X[4..7]] │ Way 1: [Y[4..7]]      │   ← beide bleiben!
│  ...  │        ...       │         ...           │
└──────────────────────────────────────────────────┘
X[i] und Y[i] koexistieren im gleichen Set → Hits ab Runde 2
```

### C-Programm

```c
/* array_conflict.c
 * Demonstriert Konflikt-Misses bei direkt-abgebildetem Cache.
 *
 * X und Y liegen 256 Bytes auseinander (= Cache-Groesse).
 * => X[i] und Y[i] haben identischen Cache-Index.
 * => Bei Assoziativitaet 1: jeder Zugriff verdraengt den anderen.
 * => Bei Assoziativitaet 2: beide Werte passen gleichzeitig in den Cache.
 */
#define N 32

int X[N];
int Y[N];   /* Startadresse von Y = Startadresse von X + 256 Bytes  */
            /* Sichergestellt durch Padding im Assembler-Programm.  */

int main(void) {
    int i, sum = 0;

    /* Initialisierung */
    for (i = 0; i < N; i++) {
        X[i] = i;
        Y[i] = i * 2;
    }

    /* Kritische Schleife: abwechselnder Zugriff auf X und Y.
     * Bei Assoziativitaet 1:  X[i] verdraengt Y[i] und umgekehrt.
     * Bei Assoziativitaet 2:  beide Werte bleiben im Cache. */
    for (i = 0; i < N; i++) {
        sum += X[i];   /* laedt Cache-Line fuer X[i]      */
        sum += Y[i];   /* verdraengt X[i] bei Assoz. 1 !  */
    }

    return sum;
}
```

### RISC-V Assembler für Ripes

```asm
# array_conflict.s
# Konflikt-Miss Demonstration: zwei Arrays mit Abstand = Cache-Groesse
#
# Adressabstand X zu Y: 256 Bytes (= 16 Lines x 16 Bytes)
# => X[i] und Y[i] haben bei beiden Cache-Konfigurationen denselben Set-Index.
#
# Register:
#   s0 = &X,  s1 = &Y,  s2 = sum,  t0..t3 = temporaer,  a0 = Zaehler i

.equ N,            32    # Anzahl Elemente pro Array
.equ CACHE_SIZE,  256    # Abstand X zu Y in Bytes = Cache-Groesse

.data
X: .zero 128             # 32 x 4 Bytes = 128 Bytes
   .zero 128             # Padding: X + 128 + 128 = X + 256 = Startadresse Y
Y: .zero 128             # 32 x 4 Bytes

.text
_start:
    la   s0, X
    la   s1, Y
    li   s2, 0           # sum = 0

    # --- Initialisierung: X[i] = i,  Y[i] = 2*i ---
    li   a0, 0
init:
    li   t0, N
    bge  a0, t0, init_done
    slli t1, a0, 2       # Byte-Offset = i * 4
    add  t2, s0, t1
    sw   a0, 0(t2)       # X[i] = i
    slli t3, a0, 1       # t3 = 2 * i
    add  t2, s1, t1
    sw   t3, 0(t2)       # Y[i] = 2 * i
    addi a0, a0, 1
    j    init
init_done:

    # --- Kritische Schleife ---
    li   a0, 0
loop:
    li   t0, N
    bge  a0, t0, done
    slli t1, a0, 2

    add  t2, s0, t1
    lw   t2, 0(t2)       # laedt X[i]  <-- Cache-Zugriff
    add  s2, s2, t2

    add  t2, s1, t1
    lw   t2, 0(t2)       # laedt Y[i]  <-- verdraengt X[i] bei Assoz. 1!
    add  s2, s2, t2

    addi a0, a0, 1
    j    loop

done:
    mv   a0, s2          # Ergebnis in a0 sichtbar
    li   a7, 10
    ecall
```

---

## Aufgabe 2 – Simulation in Ripes

### Schritt-für-Schritt Anleitung

1. Ripes öffnen → Registerkarte **Editor** → Assembler-Code einfügen
2. Assemblieren (Hammer-Symbol)
3. **Cache** konfigurieren: Registerkarte **Cache** → **Data Cache**

### Versuch A: Direkt-abgebildeter Cache

| Parameter | Wert |
|-----------|------|
| Lines | 16 |
| Assoziativität | **1** |
| Wörter/Line | 4 |
| **Kapazität** | **256 Bytes** |

Simulation starten → **Hit-Rate notieren**.

### Versuch B: 2-fach-assoziativer Cache (gleiche Kapazität)

| Parameter | Wert |
|-----------|------|
| Lines (Sets) | **8** |
| Assoziativität | **2** |
| Wörter/Line | 4 |
| **Kapazität** | **256 Bytes** |

Simulation starten → **Hit-Rate notieren**.

### Erwartete Ergebnisse

```
Versuch A (Assoz. 1):  Hit-Rate ≈  0%   ← Thrashing
Versuch B (Assoz. 2):  Hit-Rate ≈ 50%   ← X und Y koexistieren
```

> **Hinweis:** Die Hit-Rate in Versuch B ist nicht 100%, weil die
> Initialisierungsschleife am Anfang ebenfalls gezählt wird (Cold-Start-Misses).
> In der eigentlichen Zugriffsschleife sind fast alle Zugriffe Hits.

---

## Aufgabe 3 – Analyse und Berechnung

### 3.1 Cache-Index berechnen

Die Arrays liegen im `.data`-Segment. In Ripes beginnt das `.data`-Segment
typischerweise bei Adresse `0x10000000`.

Berechnen Sie den Cache-Index für folgende Zugriffe
(Cache: 16 Lines, 4 Wörter/Line = 16 Bytes/Line):

| Zugriff | Adresse | Index-Formel | Cache-Index |
|---------|---------|--------------|-------------|
| X[0] | `0x10000000` | `(0x10000000 / 16) mod 16` | ? |
| Y[0] | `0x10000100` | `(0x10000100 / 16) mod 16` | ? |
| X[1] | `0x10000004` | | ? |
| Y[1] | `0x10000104` | | ? |

Sind X[0] und Y[0] auf demselben Cache-Index? Was folgt daraus?

### 3.2 Theoretische Miss-Rate

Für die kritische Schleife (`sum += X[i]; sum += Y[i]`) mit N=32:

- Wie viele Speicherzugriffe (`lw`) werden insgesamt ausgeführt?
- Wie viele Misses entstehen theoretisch bei Assoziativität 1?
- Wie viele Misses entstehen theoretisch bei Assoziativität 2?  
  *(Hinweis: Berücksichtigen Sie Cold-Start-Misses beim ersten Durchlauf.)*

### 3.3 Kapazität vs. Konflikt

Welcher **Miss-Typ** dominiert in diesem Beispiel – Kapazitätsmiss
oder Konfliktmiss? Begründen Sie Ihre Antwort anhand der Datengröße
und der Cache-Kapazität.

---

## Reflexionsfragen

**F1.** Warum hilft in diesem Beispiel eine **höhere Assoziativität**,
obwohl die Cache-Kapazität identisch bleibt?

**F2.** Würde eine Verdopplung der Cache-Kapazität (32 Lines, Assoz. 1)
das Problem ebenfalls lösen? Warum oder warum nicht?

**F3.** Angenommen, der Abstand zwischen X und Y wäre nicht 256 Bytes,
sondern 300 Bytes. Würde das Thrashing-Problem verschwinden?
Berechnen Sie die Cache-Indizes für X[0] und Y[0] in diesem Fall.

**F4.** Moderne CPUs verwenden für den L1-D-Cache typischerweise eine
4-fach bis 12-fach Assoziativität. Warum wird ein direkt-abgebildeter
Cache in der Praxis kaum noch eingesetzt?

**F5.** In welchen Situationen würde auch ein 2-fach-assoziativer Cache
**nicht** helfen, obwohl ein Konflikt-Problem vorliegt?

---

## Zusammenfassung

| Eigenschaft | Kapazitätsmiss | Konfliktmiss |
|-------------|---------------|--------------|
| Ursache | Daten passen nicht in den Cache | Zwei Adressen konkurrieren um denselben Set |
| Lösung (Hardware) | Größerer Cache | Höhere Assoziativität |
| Lösung (Software) | Cache-Blocking | Padding zwischen Datenstrukturen |
| Dieses Beispiel | Nein (128 B < 256 B) | **Ja** |

> **Kernaussage:** Assoziativität ist kein Ersatz für Kapazität –
> aber sie ist die effiziente Antwort auf Konfliktmisses,
> ohne die Cache-Größe zu erhöhen.

---
