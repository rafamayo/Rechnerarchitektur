# Übung 02 – Matrizenmultiplikation und Cache-Blocking

**Lehrveranstaltung:** Rechnerarchitektur  
**Thema:** Zugriffsmuster, Kapazitätsmisses und cache-aware Algorithmen  
**Werkzeug:** [Ripes RISC-V Simulator](https://github.com/mortbopet/Ripes)

---

## Motivation

In Übung 01 haben wir gesehen, dass **Konfliktmisses** durch höhere
Assoziativität behoben werden können. In dieser Übung begegnen wir
einem anderen Problem: **Kapazitätsmisses** durch ungünstige
Zugriffsmuster.

Die naive Matrizenmultiplikation ist ein Standardbeispiel aus der
Praxis – sie steckt in BLAS-Bibliotheken, in neuronalen Netzen, in
Signalverarbeitung. Und sie hat ein fundamentales Cache-Problem,
das durch Hardware-Änderungen nur **teilweise** gelöst werden kann.
Die eigentliche Lösung ist algorithmisch: **Cache-Blocking** (Tiling).

### Relevanz in der Praxis

Kapazitätsmisses durch ungünstige Zugriffsmuster gehören zu den
häufigsten versteckten Leistungsproblemen in rechenintensiven Anwendungen.

**Wissenschaftliches Rechnen und HPC.**
Matrizenmultiplikation ist der zentrale Baustein in der numerischen
linearen Algebra (LAPACK, ScaLAPACK). Hochleistungsbibliotheken wie
OpenBLAS oder Intel MKL erzielen auf modernen CPUs Durchsatzraten
nahe dem theoretischen Maximum – ausschließlich durch mehrstufiges
Cache-Blocking, abgestimmt auf L1-, L2- und L3-Cache. Ohne Blocking
liegt die Leistung typischerweise bei 10–20% des Maximums.

**Deep Learning und neuronale Netze.**
Das Training und die Inferenz neuronaler Netze bestehen zu einem
Großteil aus Matrizenmultiplikationen (vollständig verbundene Schichten,
Faltungsoperationen als implizite Matmul). Frameworks wie PyTorch und
TensorFlow delegieren diese Operationen an optimierte BLAS-Bibliotheken,
die intern Cache-Blocking verwenden. Die GPU-Entsprechung heißt
„Shared Memory Tiling" in CUDA – dasselbe Prinzip auf einer anderen
Speicherhierarchie.

**Bildverarbeitung und Faltungsoperationen.**
2D-Bildfilter (Gauß, Sobel, Median) greifen auf Pixel-Nachbarschaften
zu – ein klassisches Muster mit schlechter Cache-Lokalität bei naiver
Implementierung. Optimierte Bildverarbeitungsbibliotheken (OpenCV, halide)
verwenden Cache-Blocking und SIMD-Vektorisierung gemeinsam, um den
Speicherdurchsatz zu maximieren.

**Datenbanksysteme.**
Hash-Joins und Sort-Merge-Joins in Datenbanken verarbeiten zwei
Relationen gleichzeitig. Bei großen Relationen überschreitet der
Speicherbedarf den Cache bei weitem. Datenbanksysteme wie PostgreSQL
und MonetDB partitionieren die Daten deshalb in cache-große Blöcke
(„cache-conscious query processing"), bevor sie die eigentliche
Join-Operation durchführen.

**Compiler-Optimierungen.**
Moderne Compiler (GCC, Clang, ICC) können unter bestimmten Bedingungen
automatisch Cache-Blocking anwenden (`-O3 -march=native`). Diese
**Auto-Vektorisierung** und **Loop-Tiling** genannte Transformation
ist jedoch nur möglich, wenn der Compiler die Zugriffsmuster statisch
analysieren kann – bei indirekten Zugriffen oder Zeigern gelingt
das oft nicht, und der Programmierer muss manuell eingreifen.

---

## Hintergrund: Zugriffsmuster bei Matrizenmultiplikation

Matrizen werden in C **zeilenweise** im Speicher abgelegt (row-major).
Element `M[i][j]` einer N×N-Matrix liegt bei:

```
Adresse = &M[0][0]  +  (i * N  +  j) * 4 Bytes
```

Die naive Matrizenmultiplikation `C = A × B` lautet:

```c
for (i = 0; i < N; i++)
    for (j = 0; j < N; j++)
        for (k = 0; k < N; k++)
            C[i][j] += A[i][k] * B[k][j];
```

### Zugriffsmuster der inneren k-Schleife

```
Feste Werte: i und j sind konstant, k läuft von 0 bis N-1

A[i][k]:  i=const, k=0,1,2,...          →  Adressen: &A[i][0], &A[i][1], ...
          Abstand pro Schritt: 4 Bytes  →  ZEILENWEISE (sequentiell, gut!)

B[k][j]:  j=const, k=0,1,2,...            →  Adressen: &B[0][j], &B[1][j], ...
          Abstand pro Schritt: N*4 Bytes  →  SPALTENWEISE (Sprünge, schlecht!)

C[i][j]:  i,j=const                       →  immer dieselbe Adresse  →  1 Miss, dann Hits
```

### Visualisierung der Zugriffsmuster (N=4, Beispiel)

```
Matrix A im Speicher (zeilenweise):        Zugriff A[1][k], k=0..3:
┌────┬────┬────┬────┐                      ↓    ↓    ↓    ↓
│ 00 │ 01 │ 02 │ 03 │  Zeile 0             □    □    □    □   keine Zugriffe
│ 10 │ 11 │ 12 │ 13 │  Zeile 1    →  →  →  ■    ■    ■    ■   sequentiell ✓
│ 20 │ 21 │ 22 │ 23 │  Zeile 2             □    □    □    □
│ 30 │ 31 │ 32 │ 33 │  Zeile 3             □    □    □    □
└────┴────┴────┴────┘
Speicher: [00][01][02][03][10][11][12][13][20]...
           ↑────────────────↑  Abstand: 4 Bytes pro Schritt (1 Wort)


Matrix B im Speicher (zeilenweise):        Zugriff B[k][1], k=0..3:
┌────┬────┬────┬────┐                           ↓              Spalte 1
│ 00 │ 01 │ 02 │ 03 │  Zeile 0             □    ■    □    □   ← k=0
│ 10 │ 11 │ 12 │ 13 │  Zeile 1             □    ■    □    □   ← k=1
│ 20 │ 21 │ 22 │ 23 │  Zeile 2             □    ■    □    □   ← k=2
│ 30 │ 31 │ 32 │ 33 │  Zeile 3             □    ■    □    □   ← k=3
└────┴────┴────┴────┘
Speicher: [00][01][02][03][10][11][12][13][20]...
               ↑               ↑  Abstand: N*4 = 16 Bytes pro Schritt
               B[0][1]         B[1][1]         = 1 Cache-Line bei 4 Wörtern/Line
```

Bei N=10: Abstand pro k-Schritt = **40 Bytes = 2.5 Cache-Lines**.
B allein belegt 400 Bytes. Das ist **1.56× die Cache-Kapazität** (256 Bytes).

---

## Aufgabe 1 – Naiver Algorithmus: drei Cache-Konfigurationen

### Das Programm

Das Programm `matmul_naive.s` implementiert die naive Matrizenmultiplikation.
Durch Ändern von `.equ N` wechselt man zwischen zwei Fällen:

```asm
# ============================================================
# matmul_naive.s  --  Naive Matrizenmultiplikation  C = A x B
#
# Zum Umschalten zwischen den Faellen NUR N aendern:
#
#   Fall 1 (N=4,  passt in Cache):  .equ N, 4
#   Fall 2 (N=10, Thrashing):       .equ N, 10
#
# Cache-Konfiguration (in Ripes einstellen):
#   16 Lines, Assoziativitaet 1, 4 Woerter/Line => 256 Bytes
#
# Speicherbedarf pro Matrix: N*N*4 Bytes
#   N= 4: 3 x  64 Bytes =  192 Bytes  < 256 Bytes  => passt
#   N=10: 3 x 400 Bytes = 1200 Bytes >> 256 Bytes  => Thrashing
#
# Speicherlayout:
#   A liegt bei Adresse X
#   B liegt bei Adresse X + 400  (= 10*10*4, groesste Matrix)
#   C liegt bei Adresse X + 800
#   => Abstand ist immer 400 Bytes, unabhaengig von N.
#   => Bei N=4 liegen A[0..3][0..3], B[0..3][0..3], C[0..3][0..3]
#      jeweils kompakt am Anfang ihres 400-Byte-Blocks.
#
# Register-Konventionen:
#   s0=&A   s1=&B   s2=&C
#   s3=i    s4=j    s5=k    s6=Akkumulator
#   t0..t4 = temporaer
# ============================================================

# *** NUR HIER AENDERN ***
.equ N, 4

.data
A: .zero 400           # 10*10*4 = 400 Bytes (groesste unterstuetzte Matrix)
B: .zero 400
C: .zero 400

.text
_start:
    la   s0, A
    la   s1, B
    la   s2, C

    # --- Matrizenmultiplikation C = A x B ---
    # A und B enthalten Nullen (.zero) -- die Zugriffsmuster
    # und damit die Cache-Hit-Rate sind unabhaengig von den Werten.
    li   s3, 0               # i = 0
loop_i:
    li   t0, N
    bge  s3, t0, done
    li   s4, 0               # j = 0
loop_j:
    li   t0, N
    bge  s4, t0, next_i
    li   s6, 0               # Akkumulator = 0
    li   s5, 0               # k = 0
loop_k:
    li   t0, N
    bge  s5, t0, store_c

    # A[i][k]: zeilenweise, +4 Bytes pro k-Schritt
    li   t0, N
    mul  t1, s3, t0
    add  t1, t1, s5
    slli t1, t1, 2
    add  t1, s0, t1
    lw   t3, 0(t1)

    # B[k][j]: spaltenweise, +N*4 Bytes pro k-Schritt
    li   t0, N
    mul  t1, s5, t0
    add  t1, t1, s4
    slli t1, t1, 2
    add  t1, s1, t1
    lw   t4, 0(t1)

    mul  t3, t3, t4
    add  s6, s6, t3
    addi s5, s5, 1
    j    loop_k

store_c:
    li   t0, N
    mul  t1, s3, t0
    add  t1, t1, s4
    slli t1, t1, 2
    add  t1, s2, t1
    sw   s6, 0(t1)
    addi s4, s4, 1
    j    loop_j
next_i:
    addi s3, s3, 1
    j    loop_i

done:
    li   a7, 10
    ecall
```

### Versuch 1 – N=4, Cache passt

Setzen Sie `.equ N, 4` und konfigurieren Sie den Data Cache:

| Parameter | Wert |
|-----------|------|
| Lines | 16 |
| Assoziativität | 1 |
| Wörter/Line | 4 |
| **Kapazität** | **256 Bytes** |

**Speicherbedarf:** 3 × 4×4 × 4 Bytes = **192 Bytes < 256 Bytes** ✓

Simulieren Sie und **notieren Sie die Hit-Rate**

### Versuch 2 – N=10, naiv, gleicher Cache

Setzen Sie `.equ N, 10`, Cache-Konfiguration **unverändert**.

**Speicherbedarf:** 3 × 10×10 × 4 Bytes = **1200 Bytes >> 256 Bytes**

Simulieren Sie und **notieren Sie die Hit-Rate**

### Versuch 3 – N=10, größerer Cache (Lösung A)

Setzen Sie `.equ N, 10`, Cache-Konfiguration ändern:

| Parameter | Wert |
|-----------|------|
| Lines | **32** |
| Assoziativität | 1 |
| Wörter/Line | 4 |
| **Kapazität** | **512 Bytes** |

**Warum 32 Lines?** Matrix B allein belegt 400 Bytes = 25 Lines.
Mit 32 Lines passt B vollständig in den Cache. Nach dem ersten
Durchlauf der j-Schleife (k=0..9 für j=0) sind alle B-Lines geladen
und bleiben verfügbar.

Simulieren Sie und **notieren Sie die Hit-Rate**

### Versuch 4 – N=10, höhere Assoziativität (Lösung B)

Setzen Sie `.equ N, 10`, Cache-Konfiguration ändern:

| Parameter | Wert |
|-----------|------|
| Lines (Sets) | **8** |
| Assoziativität | **2** |
| Wörter/Line | 4 |
| **Kapazität** | **256 Bytes** (gleich wie Versuch 2!) |

Simulieren Sie und **notieren Sie die Hit-Rate**

### Ergebnistabelle

| Versuch | N | Cache | Hit-Rate |
|---------|---|-------|----------|
| 1 | 4  | 16 Lines, Assoz. 1, 256 B |      |
| 2 | 10 | 16 Lines, Assoz. 1, 256 B |     |
| 3 | 10 | **32 Lines**, Assoz. 1, **512 B** |     |
| 4 | 10 | 8 Lines, **Assoz. 2**, 256 B |     |

---

## Aufgabe 2 – Analyse der Ergebnisse

### 2.1 Vergleich Versuch 3 vs. Versuch 4

Versuch 3 verdoppelt die Cache-Kapazität. Versuch 4 lässt die
Kapazität gleich und erhöht nur die Assoziativität.

- Welcher Versuch zeigt die bessere Hit-Rate?
- Warum hilft höhere Assoziativität hier kaum, obwohl sie in
  Übung 01 das Problem vollständig gelöst hat?

### 2.2 Dominanter Miss-Typ

Füllen Sie die Tabelle aus:

| Miss-Typ | Ursache | Betroffen in diesem Beispiel? |
|----------|---------|-------------------------------|
| Cold-Start-Miss | Erste Zugriffe auf jede Cache-Line | Ja, unvermeidbar |
| Kapazitätsmiss | Daten passen nicht in den Cache | ? |
| Konfliktmiss | Zwei Adressen konkurrieren um denselben Set | ? |

### 2.3 Warum hilft Lösung A (mehr Lines) besser als Lösung B?

Matrix B wird spaltenweise gelesen. Mit 32 Lines (512 Bytes):

```
B belegt 400 Bytes = 25 Lines.
Nach dem ersten vollständigen Durchlauf von B (alle k=0..9 für j=0):
  → alle 25 Lines von B sind im Cache
  → für j=1,2,...,9: B[k][j] ist bereits geladen → HIT
```

Zeichnen Sie schematisch, welche B-Lines für j=0 geladen werden
und welche davon bei j=1 noch im Cache sind – für beide
Cache-Konfigurationen (16 Lines und 32 Lines).

---

## Aufgabe 3 – Cache-Blocking (Lösung C)

### Grundidee

Statt die gesamten Matrizen zu durchlaufen, zerlegen wir sie in
kleine **Blöcke der Größe S×S**, die gleichzeitig in den Cache passen:

```
Bedingung: 3 × S² × 4 Bytes ≤ Cache-Kapazität
           3 × S² × 4  ≤  256
           S²  ≤  21
           S   ≤  4   →  S = 4 (BLOCK = 4)
```

### Visualisierung: naiv vs. geblockt

```
Naiver Algorithmus (N=10):
┌──────────────────────────┐   ┌──────────────────────────┐
│ A  →  →  →  →  →  →  →   │   │ B  ↓  □  □  □  □  □  □   │
│    →  →  →  →  →  →  →   │   │    ↓  □  □  □  □  □  □   │
│    →  →  →  →  →  →  →   │   │    ↓  □  □  □  □  □  □   │
│    ...                   │   │    ↓  □  □  □  □  □  □   │
└──────────────────────────┘   └──────────────────────────┘
A: sequentiell (gut)           B: spaltenweise Sprünge (schlecht)
Gesamter Cache wird durch B überflutet → Thrashing


Cache-Blocking (BLOCK=4):
┌──────────────────────────┐   ┌──────────────────────────┐
│ A  ████░░░░░░░░░░░░░░░░░ │   │ B  ████░░░░░░░░░░░░░░░░░ │
│    ████░░░░░░░░░░░░░░░░░ │   │    ████░░░░░░░░░░░░░░░░░ │
│    ████░░░░░░░░░░░░░░░░░ │   │    ████░░░░░░░░░░░░░░░░░ │
│    ████░░░░░░░░░░░░░░░░░ │   │    ████░░░░░░░░░░░░░░░░░ │
│    ░░░░░░░░░░░░░░░░░░░░░ │   │    ░░░░░░░░░░░░░░░░░░░░░ │
└──────────────────────────┘   └──────────────────────────┘
████ = aktiver 4×4-Block (64 Bytes)
3 aktive Blöcke (A+B+C) = 192 Bytes < 256 Bytes → alle im Cache!
Innerhalb jedes Blocks: fast nur Hits.
```

### Schleifenstruktur

```
Naiv:                 Cache-Blocking:
for i                 for ii (Schritte: BLOCK)
  for j                 for jj (Schritte: BLOCK)
    for k                 for kk (Schritte: BLOCK)
      C+=A*B                  for i (ii..ii+BLOCK)
                                for j (jj..jj+BLOCK)
                                  for k (kk..kk+BLOCK)
                                    C+=A*B
```

Die äußeren Schleifen `ii, jj, kk` wählen den aktiven Block. Die inneren Schleifen `i, j, k` arbeiten **komplett innerhalb des Blocks**, der im Cache liegt.

### Das Programm

```asm
# ============================================================
# matmul_blocked.s  --  Cache-Blocking (Tiling)  C = A x B
#
# Cache-Konfiguration (in Ripes, unveraendert wie Fall 1/2):
#   16 Lines, Assoziativitaet 1, 4 Woerter/Line => 256 Bytes
#
# Blockgroesse BLOCK=4 ist auf den Cache abgestimmt:
#   3 x BLOCK x BLOCK x 4 Bytes = 3 x 64 = 192 Bytes < 256 Bytes
#   => Drei Bloecke passen gleichzeitig in den Cache.
#
# Register-Konventionen:
#   s0=&A   s1=&B   s2=&C
#   s3=ii   s4=jj   s5=kk        (Block-Schleifen)
#   s6=i    s7=j    s8=k         (innere Schleifen)
#   s9=Akkumulator
#   t0..t4 = temporaer
# ============================================================

.equ N,     10
.equ BLOCK,  4

.data
A: .zero 400
B: .zero 400
C: .zero 400

.text
_start:
    la   s0, A
    la   s1, B
    la   s2, C

    # --- Cache-Blocking: aeussere Schleifen ueber Bloecke ---
    # A und B enthalten Nullen (.zero) -- die Zugriffsmuster
    # und damit die Cache-Hit-Rate sind unabhaengig von den Werten.
    li   s3, 0               # ii = 0
loop_ii:
    li   t0, N
    bge  s3, t0, done
    li   s4, 0               # jj = 0
loop_jj:
    li   t0, N
    bge  s4, t0, next_ii
    li   s5, 0               # kk = 0
loop_kk:
    li   t0, N
    bge  s5, t0, next_jj

    # --- Innere Schleifen: arbeiten innerhalb eines BLOCK x BLOCK Bereichs ---
    mv   s6, s3              # i = ii
loop_i:
    li   t0, BLOCK
    add  t0, s3, t0
    bge  s6, t0, next_kk     # i >= ii+BLOCK
    mv   s7, s4              # j = jj
loop_j:
    li   t0, BLOCK
    add  t0, s4, t0
    bge  s7, t0, next_i      # j >= jj+BLOCK

    # C[i][j] laden (akkumulierter Wert aus vorherigen kk-Runden)
    li   t0, N
    mul  t1, s6, t0
    add  t1, t1, s7
    slli t1, t1, 2
    add  t1, s2, t1
    lw   s9, 0(t1)

    mv   s8, s5              # k = kk
loop_k:
    li   t0, BLOCK
    add  t0, s5, t0
    bge  s8, t0, store_c     # k >= kk+BLOCK

    # A[i][k]
    li   t0, N
    mul  t1, s6, t0
    add  t1, t1, s8
    slli t1, t1, 2
    add  t1, s0, t1
    lw   t3, 0(t1)

    # B[k][j]
    li   t0, N
    mul  t1, s8, t0
    add  t1, t1, s7
    slli t1, t1, 2
    add  t1, s1, t1
    lw   t4, 0(t1)

    mul  t3, t3, t4
    add  s9, s9, t3
    addi s8, s8, 1
    j    loop_k

store_c:
    li   t0, N
    mul  t1, s6, t0
    add  t1, t1, s7
    slli t1, t1, 2
    add  t1, s2, t1
    sw   s9, 0(t1)
    addi s7, s7, 1
    j    loop_j
next_i:
    addi s6, s6, 1
    j    loop_i
next_kk:
    li   t0, BLOCK
    add  s5, s5, t0
    j    loop_kk
next_jj:
    li   t0, BLOCK
    add  s4, s4, t0
    j    loop_jj
next_ii:
    li   t0, BLOCK
    add  s3, s3, t0
    j    loop_ii

done:
    li   a7, 10
    ecall```

### Versuch 5 – Cache-Blocking, originaler Cache

Laden Sie `matmul_blocked.s`. Cache-Konfiguration **zurück auf Versuch 2**:

| Parameter | Wert |
|-----------|------|
| Lines | 16 |
| Assoziativität | 1 |
| Wörter/Line | 4 |
| **Kapazität** | **256 Bytes** |

Simulieren Sie und **notieren Sie die Hit-Rate**

---

## Aufgabe 4 – Gesamtvergleich

### Ergebnistabelle (vollständig)

| Versuch | N | Cache | Algorithmus | Hit-Rate |
|---------|---|-------|-------------|----------|
| 1 | 4  | 256 B, Assoz. 1 | naiv    |      |
| 2 | 10 | 256 B, Assoz. 1 | naiv    |      |
| 3 | 10 | **512 B**, Assoz. 1 | naiv  |      |
| 4 | 10 | 256 B, **Assoz. 2** | naiv  |      |
| 5 | 10 | 256 B, Assoz. 1 | **Blocking** |      |

### Visualisierung: Woher kommen die Misses?

```
Versuch 2 (naiv, N=10):
  Initialisierung: [Cold-Start-Misses für A, B, C]
  Multiplikation:
    für jedes (i,j)-Paar (100 Paare):
      k-Schleife (10 Schritte):
        A[i][k]: 1 Miss pro 4 Elemente    → ~25 Misses für alle A-Zeilen
        B[k][j]: 1 Miss pro Schritt fast  → ~10 Misses pro j-Iteration
                 (B verdrängt A-Lines!)      × 100 Iterationen = ~1000 Misses

Versuch 5 (Blocking, N=10, BLOCK=4):
  Pro Block-Tripel (ii,jj,kk): 3 × 4×4 Elemente = 192 Bytes geladen
  Innerhalb des Blocks: fast alle Zugriffe sind Hits
  Anzahl Block-Tripel: (10/4)³ ≈ 15 (mit Randbehandlung)
  → viel weniger Misses insgesamt
```

---

## Reflexionsfragen

**F1.** Warum verbessert höhere Assoziativität (Versuch 4) die Hit-Rate
bei der naiven Matrizenmultiplikation kaum, obwohl es in Übung 01
das Problem vollständig gelöst hat? Welcher Miss-Typ dominiert jeweils?

**F2.** Versuch 3 verdoppelt die Cache-Kapazität und verbessert die
Hit-Rate. Ist das eine nachhaltige Lösung? Was passiert, wenn N
auf 20 wächst?

**F3.** Die Blockgröße BLOCK=4 wurde berechnet als:
```
3 × BLOCK² × 4 ≤ 256  →  BLOCK ≤ 4
```
Berechnen Sie die optimale Blockgröße für einen Cache mit
512 Bytes (32 Lines, 4 Wörter/Line).

**F4.** Cache-Blocking ist ein **cache-aware** Algorithmus: die
Blockgröße hängt explizit von der Cache-Konfiguration ab.
Was wäre ein Nachteil dieses Ansatzes in der Praxis?
*(Hinweis: Was passiert, wenn das Programm auf einer anderen CPU
mit anderem Cache läuft?)*

**F5.** Würde Cache-Blocking auch bei dem Array-Konflikt-Problem
aus Übung 01 helfen? Begründen Sie Ihre Antwort.

**F6.** In der Praxis verwendet man für Hochleistungs-Matrizenmultiplikation
(z.B. OpenBLAS, Intel MKL) oft **mehrstufiges Blocking** – Blöcke
für L1-Cache, L2-Cache und L3-Cache. Warum reicht ein einziges
Blocking für L1 nicht aus?

---

## Zusammenfassung: Drei Lösungsansätze im Vergleich

```
Problem: Naive Matrizenmultiplikation, N=10, Cache 256 Bytes
                           Hit-Rate  Kapazität  Kosten
─────────────────────────────────────────────────────────
Ausgangspunkt (naiv)        ~70%      256 B      –
Lösung A: mehr Cache-Lines  ~??%      512 B      2× Hardware
Lösung B: Assoziativität 2  ~70%      256 B      etwas mehr HW
Lösung C: Cache-Blocking    ~??%      256 B      Algorithmus-Änderung
─────────────────────────────────────────────────────────
```

> **Kernaussagen:**
>
> 1. **Assoziativität** bekämpft Konfliktmisses – hilft nicht gegen
>    Kapazitätsmisses.
>
> 2. **Mehr Cache** hilft gegen Kapazitätsmisses – aber nicht skalierbar
>    (N=20 braucht wieder mehr Cache).
>
> 3. **Cache-Blocking** löst das Problem algorithmisch – mit dem
>    gleichen Cache wie das Original. Das ist die nachhaltige Lösung.
>
> 4. Cache-Blocking ist ein **cache-aware** Algorithmus: er kennt
>    und nutzt die Cache-Geometrie explizit. Der Preis ist, dass
>    die Blockgröße an den konkreten Cache angepasst werden muss.

---

## Weiterführende Themen

- **Cache-oblivious Algorithmen:** Rekursive Zerlegung, die gut
  performt ohne die Cache-Parameter zu kennen (z.B. rekursive
  Matrizenmultiplikation nach Frigo et al.)
- **BLAS-Bibliotheken:** OpenBLAS, Intel MKL implementieren
  mehrstufiges Blocking für L1/L2/L3-Cache automatisch
- **Prefetching:** Hardware- und Software-Prefetch-Strategien
  für spaltenweise Zugriffe

---
