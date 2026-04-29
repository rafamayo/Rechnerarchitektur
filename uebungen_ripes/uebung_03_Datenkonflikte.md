# Übung 03 – Datenabhängigkeiten in der RISC-V Pipeline

**Lehrveranstaltung:** Rechnerarchitektur  
**Thema:** RAW-Hazards, Forwarding und Stalls in der 5-stufigen Pipeline  
**Werkzeug:** [Ripes RISC-V Simulator](https://github.com/mortbopet/Ripes)

---

## Motivation

Eine Pipeline steigert den Durchsatz einer CPU, indem mehrere Befehle
gleichzeitig in verschiedenen Stufen ausgeführt werden. Dieser Gewinn hat
jedoch einen Preis: Befehle, die voneinander abhängen, können nicht
uneingeschränkt überlappen. Wenn Befehl B das Ergebnis von Befehl A
benötigt, A aber noch nicht fertig ist, entsteht ein **Datenhazard**.

Die Hardware muss solche Konflikte **automatisch erkennen und auflösen** –
entweder durch **Forwarding** (Ergebnis direkt weiterleiten, ohne WB
abzuwarten) oder durch einen **Stall** (Pipeline anhalten, bis das
Ergebnis verfügbar ist). Welche Strategie wann greift, und was das für
die Ausführungszeit bedeutet, untersuchen wir in dieser Übung.

### Relevanz in der Praxis

Datenhazards und ihre Behandlung sind kein akademisches Detail –
sie haben direkten Einfluss auf die Leistung realer Prozessoren.

**Compiler-Scheduling.**
Moderne Compiler (GCC, Clang) kennen die Pipeline-Struktur der
Ziel-CPU und ordnen Befehle so um, dass Hazards vermieden werden.
Die Option `-O2` aktiviert unter anderem **Instruction Scheduling**:
der Compiler schiebt unabhängige Befehle zwischen abhängige, um
Stalls zu eliminieren – ohne die Korrektheit des Programms zu ändern.

**In-Order- vs. Out-of-Order-Prozessoren.**
Einfache Mikrocontroller (ARM Cortex-M0, RISC-V RV32I ohne Erweiterungen)
verwenden In-Order-Pipelines: Hazards führen direkt zu Stalls.
Hochleistungsprozessoren (Intel Core, AMD Zen, Apple M-Serie) verwenden
Out-of-Order-Execution mit Reservation Stations: Befehle warten nicht
in der Pipeline, sondern in einer Warteschlange und werden ausgeführt,
sobald ihre Operanden bereit sind.

**Eingebettete Systeme und Echtzeit.**
In sicherheitskritischen Systemen (Motorsteuerung, Avionik) müssen
Entwickler die genaue Ausführungszeit eines Codestücks kennen –
**Worst-Case Execution Time (WCET)**. Stalls durch Datenhazards
müssen dabei explizit berücksichtigt werden.

---

## Hintergrund: Die 5-stufige RISC-V Pipeline

```
┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐
│ IF  │──▶│ ID  │──▶│ EX  │──▶│ MEM │──▶│ WB  │
│Fetch│   │Deco-│   │ ALU │   │Mem- │   │Reg- │
│     │   │ de  │   │     │   │Zgrff│   │Write│
└─────┘   └─────┘   └─────┘   └─────┘   └─────┘
             ↑                                │
             └────────────────────────────────┘
                   Register-File: Lesen in ID,
                   Schreiben in WB
```

**Pipeline-Register** trennen die Stufen und speichern den Zustand
zwischen zwei Takten:

```
IF/ID  │  ID/EX  │  EX/MEM  │  MEM/WB
       │         │          │
       │ .rs1    │ .rd      │ .rd
       │ .rs2    │ .ALUres  │ .Ergebnis
       │ .rd     │ .MemRead │ .RegWrite
       │ ...     │ ...      │ ...
```

### Arten von Datenabhängigkeiten

```
RAW – Read After Write (echte Abhängigkeit):
  add x1, x2, x3    # schreibt x1
  sub x4, x1, x5    # liest x1  ← häufigster Hazard, muss aufgelöst werden

WAR – Write After Read (Anti-Abhängigkeit):
  sub x4, x1, x5    # liest x1
  add x1, x2, x3    # schreibt x1  ← in In-Order-Pipeline kein Problem

WAW – Write After Write (Ausgabe-Abhängigkeit):
  add x1, x2, x3    # schreibt x1
  sub x1, x4, x5    # schreibt x1  ← in In-Order-Pipeline kein Problem
```

In einer **In-Order-5-Stufen-Pipeline** sind nur **RAW-Hazards**
problematisch. WAR und WAW entstehen erst bei Out-of-Order-Execution.

### Forwarding-Bedingungen

Die Hazard Detection Unit vergleicht die Register-Nummern in den
Pipeline-Registern. Forwarding ist nötig, wenn:

```
EX-Forwarding  (1 Takt Abstand):
  EX/MEM.rd == ID/EX.rs1   →  ForwardA = 10  (MUX wählt EX/MEM-Ergebnis)
  EX/MEM.rd == ID/EX.rs2   →  ForwardB = 10

MEM-Forwarding (2 Takte Abstand):
  MEM/WB.rd == ID/EX.rs1   →  ForwardA = 01  (MUX wählt MEM/WB-Ergebnis)
  MEM/WB.rd == ID/EX.rs2   →  ForwardB = 01

Zusatzbedingung: rd ≠ x0  (x0 ist hardverdrahtet auf 0)
```

### Load-Use-Hazard (Stall unvermeidbar)

```
lw  x1, 0(x2)     # x1 kommt erst nach MEM-Stufe
add x3, x1, x4    # braucht x1 bereits in EX-Stufe
                  → 1 Takt Stall nötig, danach MEM-Forwarding
```

```
Takt:   1    2    3    4    5    6    7
lw:    IF   ID   EX  [MEM] WB
add:        IF   ID  [ID]  EX   MEM  WB
                      ↑
                   Stall: add wiederholt ID-Stufe für 1 Takt
                   danach: MEM/WB → Forward → add.EX
```

---

## Teil A – Theorie und Verständnis

### A1 – Abhängigkeiten identifizieren

Kennzeichnen Sie alle Datenabhängigkeiten im folgenden Codeabschnitt.
Geben Sie für jede Abhängigkeit den Typ (RAW, WAR, WAW) und die
betroffenen Befehle an.

```asm
(1)  add  x1, x2, x3
(2)  sub  x4, x1, x5
(3)  and  x6, x4, x1
(4)  or   x1, x6, x7
(5)  xor  x8, x1, x4
```

Tabelle zum Ausfüllen:

| Abhängigkeit | Typ | Schreibender Befehl | Lesender Befehl | Register |
|---|---|---|---|---|
| ? | ? | (?) | (?) | x? |
| | | | | |
| | | | | |
| | | | | |

Welche dieser Abhängigkeiten verursachen in einer In-Order-Pipeline
tatsächlich einen Hazard?

### A2 – Pipeline-Diagramm zeichnen

Zeichnen Sie das Pipeline-Diagramm für die folgenden zwei Befehle
**ohne Forwarding** (d.h. Ergebnis steht erst nach WB zur Verfügung).
Wie viele Stall-Zyklen sind nötig?

```asm
add  x1, x2, x3
sub  x4, x1, x5
```

```
Takt:   1    2    3    4    5    6    7    8    9
add:   IF   ID   EX   MEM  WB
sub:        IF   ID
            ...  (Stalls einfügen)
```

Wie viele Stall-Zyklen sind nötig, wenn **Forwarding** verfügbar ist?

### A3 – Forwarding-Pfade bestimmen

Für jede der folgenden Befehlssequenzen: Ist Forwarding möglich?
Wenn ja, aus welchem Pipeline-Register wird weitergeleitet (EX/MEM oder MEM/WB)?
Wenn nein, wie viele Stalls entstehen?

**Sequenz 1** (1 Befehl Abstand):
```asm
add  x1, x2, x3    # Befehl A
sub  x4, x1, x5    # Befehl B
```

**Sequenz 2** (2 Befehle Abstand):
```asm
add  x1, x2, x3    # Befehl A
and  x6, x7, x8    # Befehl X (unabhängig)
sub  x4, x1, x5    # Befehl B
```

**Sequenz 3** (Load-Use):
```asm
lw   x1, 0(x2)     # Befehl A
sub  x4, x1, x5    # Befehl B
```

**Sequenz 4** (Load, 1 Befehl Abstand):
```asm
lw   x1, 0(x2)     # Befehl A
and  x6, x7, x8    # Befehl X (unabhängig)
sub  x4, x1, x5    # Befehl B
```

Tabelle zum Ausfüllen:

| Sequenz | Forwarding? | Quelle | Stalls |
|---------|------------|--------|--------|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |

### A4 – Ausführungszeit berechnen

Eine Pipeline führt folgende Befehlssequenz aus:

```asm
(1)  lw   x1, 0(x0)
(2)  lw   x2, 4(x0)
(3)  add  x3, x1, x2
(4)  sw   x3, 8(x0)
(5)  add  x4, x3, x1
```

a) Identifizieren Sie alle RAW-Abhängigkeiten.

b) Markieren Sie, welche durch Forwarding aufgelöst werden können
   und welche einen Stall erfordern.

c) Zeichnen Sie das vollständige Pipeline-Diagramm mit Stalls
   (Forwarding ist vorhanden):

```
       1    2    3    4    5    6    7    8    9   10   11
(1):  IF   ID   EX   MEM  WB
(2):       IF   ID   EX   MEM  WB
(3):            IF   ID   ...
(4):
(5):
```

d) Wie viele Takte benötigt die Sequenz insgesamt?
   Wie viele Takte wären es ohne jede Abhängigkeit (ideale Pipeline)?

### A5 – Compiler-Optimierung durch Umsortierung

Der folgende Code enthält einen Load-Use-Hazard:

```asm
lw   x1, 0(x0)      # Lädt Wert A
add  x2, x1, x3     # RAW auf x1 → Stall!
lw   x4, 4(x0)      # Lädt Wert B
add  x5, x4, x6     # RAW auf x4 → Stall!
sw   x2, 8(x0)
sw   x5, 12(x0)
```

a) Wie viele Stall-Zyklen entstehen?

b) Sortieren Sie die Befehle so um, dass **keine Stalls** mehr entstehen,
   ohne die Semantik des Programms zu verändern.
   *(Hinweis: Welche Befehle sind voneinander unabhängig?)*

---

## Teil B – Simulation in Ripes

### Einrichtung

1. Ripes öffnen
2. Prozessor-Modell wählen: **„RISC-V 5-stage pipeline"**  
   *(nicht den Single-Cycle-Prozessor!)*
3. Im Menü **View → Show pipeline** aktivieren
4. Schrittweise ausführen mit dem **Einzelschritt-Button** (ein Takt pro Klick)

In der Pipeline-Ansicht sehen Sie:
- Welcher Befehl sich in welcher Stufe befindet
- **Forwarding-Pfade** als farbige Pfeile
- **Stalls** als graue „Bubble"-Einträge in der Pipeline

---

### B1 – RAW mit Forwarding beobachten

Laden Sie folgenden Code:

```asm
.text
_start:
    li   x2, 10
    li   x3, 20
    add  x1, x2, x3    # schreibt x1
    sub  x4, x1, x3    # liest x1  <- RAW, 1 Takt Abstand
    and  x5, x4, x1    # liest x4 und x1  <- RAW auf x4 (1 Takt), x1 (2 Takte)
    li   a7, 10
    ecall
```

**Aufgaben:**

a) Führen Sie den Code schrittweise aus. Beobachten Sie die Pipeline-Ansicht.
   Wie viele Stall-Zyklen treten auf?

b) Notieren Sie, welche Forwarding-Pfade Ripes für `sub` und `and` anzeigt.
   Aus welchem Pipeline-Register wird jeweils weitergeleitet?

c) Überprüfen Sie Ihre Vorhersage aus A3 (Sequenz 1).

---

### B2 – Load-Use-Hazard beobachten

Laden Sie folgenden Code:

```asm
.data
wert1: .word 42
wert2: .word 17

.text
_start:
    la   x10, wert1
    lw   x1, 0(x10)    # laedt wert1 in x1
    add  x2, x1, x1    # RAW auf x1: Load-Use-Hazard!
    lw   x3, 4(x10)    # laedt wert2 in x3
    add  x4, x3, x2    # RAW auf x3: Load-Use-Hazard!
    li   a7, 10
    ecall
```

**Aufgaben:**

a) Führen Sie den Code schrittweise aus.
   Wie viele Stall-Zyklen treten insgesamt auf?

b) Notieren Sie die Takte, in denen die Stalls auftreten.
   Was passiert in der Pipeline-Ansicht während eines Stalls?
   *(Was tut die IF-Stufe? Was tut die EX-Stufe?)*

c) Vergleichen Sie mit Code B3 unten: Wie verändert sich das Pipeline-Diagramm?

---

### B3 – Load-Use-Hazard durch Umsortierung eliminieren

Laden Sie den umsortierten Code:

```asm
.data
wert1: .word 42
wert2: .word 17

.text
_start:
    la   x10, wert1
    lw   x1, 0(x10)    # laedt wert1
    lw   x3, 4(x10)    # laedt wert2  <- vorgezogen!
    add  x2, x1, x1    # x1 jetzt 2 Takte alt -> kein Stall
    add  x4, x3, x2    # x3 jetzt 2 Takte alt -> kein Stall
    li   a7, 10
    ecall
```

**Aufgaben:**

a) Führen Sie den Code aus. Wie viele Stall-Zyklen treten auf?

b) Vergleichen Sie die Gesamttaktanzahl mit B2.
   Wie groß ist der Gewinn durch die Umsortierung?

c) Welche Forwarding-Pfade sind jetzt aktiv?
   Aus welchem Pipeline-Register wird für `add x2` und `add x4` weitergeleitet?

---

### B4 – Doppelter Load-Use und Compiler-Scheduling

Laden Sie folgenden Code, der eine einfache Berechnung `z = (a + b) * (a - b)` simuliert:

```asm
.data
a: .word 8
b: .word 3

.text
_start:
    la   x10, a
    lw   x1, 0(x10)    # x1 = a
    lw   x2, 4(x10)    # x2 = b
    add  x3, x1, x2    # x3 = a + b  <- RAW auf x2 (Load-Use!)
    sub  x4, x1, x2    # x4 = a - b  <- RAW auf x2 (Load-Use!)
    mul  x5, x3, x4    # x5 = (a+b)*(a-b)
    li   a7, 10
    ecall
```

**Aufgaben:**

a) Identifizieren Sie alle RAW-Abhängigkeiten und markieren Sie,
   welche Stalls verursachen.

b) Zählen Sie die Stall-Zyklen in Ripes.

c) Schreiben Sie eine optimierte Version, die alle Stalls eliminiert.
   *(Hinweis: Welcher Befehl kann zwischen `lw x2` und `add x3` eingefügt werden?)*

d) Verifizieren Sie Ihre optimierte Version in Ripes: Tritt kein Stall mehr auf?

---

### B5 – WAR und WAW: Kein Hazard in der In-Order-Pipeline

Laden Sie folgenden Code:

```asm
.text
_start:
    li   x1, 5
    li   x2, 3
    li   x3, 7
    sub  x4, x1, x2    # liest x1  (Befehl A)
    add  x1, x3, x2    # schreibt x1  <- WAR auf x1 bezüglich A
    add  x5, x1, x4    # liest x1 (neuer Wert), liest x4
    li   a7, 10
    ecall
```

**Aufgaben:**

a) Gibt es einen RAW-Hazard in diesem Code? Wenn ja, wo?

b) Gibt es eine WAR-Abhängigkeit? Zwischen welchen Befehlen?

c) Führen Sie den Code in Ripes aus. Tritt ein Stall aufgrund der
   WAR-Abhängigkeit auf? Erklären Sie warum (oder warum nicht).

d) Was wäre das Ergebnis in `x5`? Berechnen Sie es von Hand
   und vergleichen Sie mit dem Register-File in Ripes nach der Ausführung.

---

## Teil C – Vertiefung

### C1 – Hazard-Erkennung: Hardware-Logik

Die Forwarding-Bedingung lautet (vereinfacht):

```
EX-Forwarding aktiv, wenn:
  EX/MEM.RegWrite = 1
  AND EX/MEM.rd ≠ 0
  AND EX/MEM.rd = ID/EX.rs1   (→ ForwardA = 10)

MEM-Forwarding aktiv, wenn:
  MEM/WB.RegWrite = 1
  AND MEM/WB.rd ≠ 0
  AND MEM/WB.rd = ID/EX.rs1   (→ ForwardA = 01)
  AND NOT (EX-Forwarding aktiv für rs1)   ← EX hat Vorrang!
```

Für den folgenden Code zum Takt 5:

```asm
(1)  add  x1, x2, x3    # in Takt 1 gestartet
(2)  sub  x4, x5, x6    # in Takt 2 gestartet
(3)  and  x7, x1, x4    # in Takt 3 gestartet
```

Füllen Sie den Zustand der Pipeline-Register in **Takt 5** aus:

| Pipeline-Register | Feld | Wert |
|---|---|---|
| EX/MEM | rd | ? |
| EX/MEM | RegWrite | ? |
| MEM/WB | rd | ? |
| MEM/WB | RegWrite | ? |
| ID/EX  | rs1 | ? |
| ID/EX  | rs2 | ? |

Welche Forwarding-Signale (ForwardA, ForwardB) werden gesetzt?

### C2 – Kritische Pfadanalyse

Betrachten Sie folgende Schleife:

```asm
    li   x1, 0         # Summe = 0
    li   x2, 0         # i = 0
    li   x3, 10        # Grenze N = 10
    la   x4, array     # Basisadresse
loop:
    bge  x2, x3, done
    slli x5, x2, 2     # Byte-Offset = i * 4
    add  x6, x4, x5    # Adresse = array + Offset
    lw   x7, 0(x6)     # x7 = array[i]
    add  x1, x1, x7    # Summe += array[i]
    addi x2, x2, 1     # i++
    j    loop
done:
    li   a7, 10
    ecall

.data
array: .word 1,2,3,4,5,6,7,8,9,10
```

a) Identifizieren Sie alle RAW-Abhängigkeiten **innerhalb eines
   Schleifendurchlaufs**.

b) Welche Hazards werden durch Forwarding aufgelöst,
   welche erfordern einen Stall?

c) Wie viele Stall-Zyklen entstehen **pro Schleifendurchlauf**?

d) Führen Sie den Code in Ripes aus und überprüfen Sie Ihre Vorhersage.

e) Könnten Sie den Code durch Umsortierung von Befehlen verbessern?
   *(Hinweis: Gibt es einen unabhängigen Befehl, der zwischen `lw` und `add x1` passt?)*

### C3 – Vergleich: Mit und ohne Forwarding

Ripes erlaubt es, Forwarding zu **deaktivieren**:  
*Einstellungen → Prozessor-Optionen → Forwarding deaktivieren*

Laden Sie den Code aus B1 und führen Sie ihn **einmal mit, einmal ohne**
Forwarding aus.

| Messung | Mit Forwarding | Ohne Forwarding |
|---------|---------------|-----------------|
| Gesamttakte | | |
| Stall-Zyklen | | |
| Ausgeführte Befehle | | |
| CPI (Takte / Befehle) | | |

Erklären Sie den Unterschied im CPI-Wert.

---

## Reflexionsfragen

**F1.** Warum ist in einer In-Order-Pipeline nur RAW ein echtes
Problem, während WAR und WAW keinen Hazard verursachen?
Was würde sich ändern, wenn der Prozessor Out-of-Order-Execution
mit Registerumbenennung (Register Renaming) einsetzen würde?

**F2.** Forwarding löst RAW-Hazards fast immer – außer beim
Load-Use-Hazard. Warum ist dieser Fall grundsätzlich anders?
Könnte man durch eine tiefere Pipeline (mehr Stufen) das Problem
verschlimmern oder verbessern?

**F3.** In Aufgabe B3 haben wir durch Umsortierung von Befehlen
Stalls eliminiert. Der Compiler macht genau dasselbe automatisch.
Welche Voraussetzung muss gelten, damit zwei Befehle sicher
vertauscht werden dürfen?

**F4.** Ein Prozessor führt eine Schleife 1000-mal aus.
Pro Iteration entstehen 2 Stall-Zyklen durch Load-Use-Hazards.
Die Iteration hat 8 Befehle. Berechnen Sie:
- die Gesamttakte mit Stalls
- die Gesamttakte ohne Stalls (ideale Pipeline)
- den CPI-Wert in beiden Fällen

**F5.** Manche RISC-Architekturen (z.B. frühe MIPS-Prozessoren)
verwendeten **Delay Slots**: Der Befehl nach einem Load oder Branch
wird **immer** ausgeführt, unabhängig vom Hazard. Der Compiler
muss diesen Slot mit einem sinnvollen Befehl füllen (oder NOP).
Welchen Vorteil hat das gegenüber Hardware-Stalls?
Welchen Nachteil hat es für den Compiler und die Programmierbarkeit?

**F6.** In Aufgabe C2 haben Sie eine Schleife analysiert.
Angenommen, der Prozessor hätte **Branch Prediction** und würde
den Schleifenrücksprung korrekt vorhersagen: Welche zusätzlichen
Hazards könnten zwischen dem **letzten Befehl einer Iteration**
und dem **ersten Befehl der nächsten Iteration** entstehen?

---

## Zusammenfassung

```
Hazard-Typ      Ursache                    Lösung in Hardware
──────────────────────────────────────────────────────────────
RAW (ALU→ALU)   Ergebnis 1 Takt zu früh   EX-Forwarding (kein Stall)
RAW (ALU→ALU)   Ergebnis 2 Takte zu früh  MEM-Forwarding (kein Stall)
Load-Use        Ergebnis nach MEM nötig    1 Stall + MEM-Forwarding
WAR             In-Order: kein Problem     –
WAW             In-Order: kein Problem     –
──────────────────────────────────────────────────────────────
```

> **Kernaussagen:**
>
> 1. **RAW-Hazards** sind die einzigen Datenhazards, die in einer
>    In-Order-Pipeline Hardware-Maßnahmen erfordern.
>
> 2. **Forwarding** löst fast alle RAW-Hazards ohne Stall –
>    durch direkte Rückleitung des Ergebnisses an den ALU-Eingang.
>
> 3. **Load-Use-Hazards** erfordern immer genau 1 Stall-Zyklus,
>    da das Ergebnis erst nach der MEM-Stufe verfügbar ist.
>
> 4. **Compiler-Scheduling** kann Stalls durch Umsortierung
>    unabhängiger Befehle vermeiden – ohne Korrektheitsverlust.
>
> 5. Der **CPI-Wert** (Takte pro Befehl) ist das wichtigste Maß
>    für Pipeline-Effizienz: ideale Pipeline CPI = 1,
>    jeder Stall erhöht den CPI-Wert.

---

## Weiterführende Themen

- **Out-of-Order-Execution:** Tomasulo-Algorithmus, Reservation Stations,
  Register Renaming – eliminiert WAR und WAW, reduziert RAW-Stalls
- **Branch Hazards:** Kontrollflusshazards bei bedingten Sprüngen,
  Branch Prediction, Spekulative Ausführung
- **Strukturhazards:** Konflikte um gemeinsame Hardware-Ressourcen
  (z.B. ein gemeinsamer Speicher-Port für IF und MEM)
- **Superskalarität:** Mehrere Befehle pro Takt, mehrere Forwarding-Netzwerke

