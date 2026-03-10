# Litt om Makefiles

## Intro

En Makefile er en konfigurasjonsfil som brukes av verktøyet Make for å automatisere bygging av programmer og andre repeterende oppgaver. Den brukes ofte i programvareutvikling for å kompilere kildekode, kjøre tester, generere dokumentasjon eller pakke prosjekter, men kan også erstatte tradisjonelle shell-scripts for å kjøre sett av kommandoer.

### Hva løser Make?

I større prosjekter kan det være mange filer som må bygges i riktig rekkefølge. Hvis én fil endres, trenger man ofte bare å bygge deler av prosjektet på nytt. Make analyserer **avhengigheter mellom filer** og bygger kun det som er nødvendig.

Dette gir:

- raskere bygging
- automatisert arbeidsflyt
- færre manuelle feil
- en standardisert måte å bygge prosjektet på

Selv om mange moderne byggesystemer eksisterer (som Just, CMake, Gradle eller npm-scripts), er Make fortsatt svært utbredt fordi det er:

- enkelt
- fleksibelt
- tilgjengelig på nesten alle Unix-lignende systemer
- godt egnet til små og mellomstore prosjekter

Makefiles brukes derfor fortsatt i mange open source-prosjekter, systemprogramvare og innebygde systemer.

Vi skal bl.a. se nærmere på:

- variabler og hvordan de brukes
- automatiske variabler
- mønsterregler (pattern rules)
- funksjoner
- tilordninger
- conditionals

❗ *Når vi snakker om Make i dette heftet, er det GNU Make vi snakker om*

---

### Grunnleggende struktur

En Makefile består vanligvis av regler (*rules*). En regel beskriver:

1. Et mål (*target*)
2. Avhengigheter (*dependencies*)
3. Kommandoer (*recipes*)

Den grunnleggende syntaksen for en regel er:


```makefile
target: dependency1 dependency2
    command1
    command2
```

  **Target** kan enten være
  - filer (som ofte skal produseres), eller
  - merkelapper (såkalte *Phony targets*).
  
  Phony targets deklareres med `.PHONY` for å unngå konflikter med eksisterende filer og slike blir alltid kjørt.
  
  ❗ *Make avgjør normalt om en regel må kjøres ved å sammenligne timestamps på filer. For phony targets finnes ingen slik fil, og dermed heller ingen timestamp. Make har derfor ingen måte å avgjøre om targetet allerede er oppdatert, og recipe kjøres derfor alltid.*

#### Dependencies

(valgfrie) er andre targets regelen er avhengig av.

- Hvis en dependency ikke har regel, antar Make at den representerer en fil
- Hvis en dependency-fil ikke eksisterer, rapporteres feil.
- For fil-targets bygges målet kun hvis det mangler, eller om en dependency er nyere enn target.

❗ *Man har også såkalte *order-only dependencies*, de som kommer etter en `|` De må være oppfylt før target bygges, men påvirker ikke Make sin vurdering av om targetet er oppdatert.*

```makefile
target: dependency1 dependency2 | dependency0
    command1
    command2
```

#### Recepies

er shell-kommandoer som kjøres. Disse må starte med en TAB, ikke mellomrom.

Make kjører kommandoene i et shell, vanligvis **/bin/sh**.

Eksempel:

```makefile
report.pdf: report.adoc
	asciidoctor-pdf report.adoc -o report.pdf
```

Målet **report.pdf** produseres her ved

```bash
% asciidoctor-pdf report.adoc -o report.pdf
```

(dersom PDF-filen mangler eller om **report.adoc** er nyere).

---

## Variable

En variabel defineres (f.eks.) slik:

```makefile
NAME = value
```

Variabler refereres vanligvis med:

```makefile
$(VARIABLE)
```

eller

```makefile
${VARIABLE}
```

Her ser vi et brukseksempel med to **targets**:

```makefile
BUILD_DIR = build

prepare:
    @mkdir -p $(BUILD_DIR)

clean:
    @rm -rf $(BUILD_DIR)
```

Prefikset `@` gjør at selve kommandoen ikke vises i terminalen.

Variabler kan også settes når Make kjøres:

```bash
% make cfiles TARGET=Developers
```

Dette overstyrer verdien definert i Makefile.

---

## Automatiske variabler

Make tilbyr flere **automatiske variabler** som kan knyttes til target og/eller dependencies på ulike måter. Disse er spesielt nyttige når man ønsker generelle eller gjenbrukbare regler.


```makefile
$@    - Navnet på målet (target)
$<    - Den første avhengigheten
$^    - Alle avhengigheter
$?    - Avhengigheter som er nyere enn målet
$*    - Basenavnet til målet (uten suffix)
$(@D) - Katalogdelen av målet
$(@F) - Filnavnet av målet
$(<D) - Katalogdelen av første avhengighet
$(<F) - Filnavnet av første avhengighet
```

For vanlige fil-targets (eller fil-dependencies) som `<filnavn>.txt`, representer `$*` bare `<filnavn>`. Det kan da også nevnes at man også da kan få tak i suffix `.txt` (inklusive punktum) ved 

- `$(suffix …)`

Anvendt på target **fil.txt**, representerer eksempelvis `$(suffix $@)` endelsen `.txt`. Dette er egentlig en funksjon, og vi skal se på flere slike etter hvert.

❗ *Order-only dependencies (de som kommer etter `|`) inngår ikke i i automatiske variable*

Men la oss se nærmere på de kanskje viktigste variablene for oss.

### `$@` 

Denne representerer **målnavnet**.

Eksempel:

```makefile
main.c:
	cp "Running target: $@"
```

Kjøring:

```bash
% make main.c
```

Output:

```text
Running target: main.c
```

### `$<`

Denne representerer **første dependency** i regelen.

Eksempel:

```makefile
print-file: message.txt remember.txt
	cat $<
```

Her vil `$<` bli erstattet med `message.txt`, slik at innholdet av denne vil bli printet til skjerm ved kallet `make print-file`


### `$^`

Denne representerer **alle dependencies** i regelen.

Eksempel:

```makefile
combine: a.txt b.txt
	cat $^ > combined.txt
```

Dette blir ekvivalent med:

```bash
% cat a.txt b.txt > combined.txt
```

### `$*`

`Denne representerer basenavnet til målet.

Eksempel:

```makefile
foo.o: foo.c
	gcc -c $< -o $@
	echo "Basenavn: $*"
```


Her har vi sammenhengene:

- `$@` → foo.o    (target)
- `$<` → foo.c    (dependency)
- `$*` → foo      (target uten sufix)

Man kan kalle regelen ved:

```bash
% make foo.o
```

med output:

```text
Basenavn: foo
```

## %-mønstre

Nært beslektet er såkalte *pattern rules*, eller mønsterregler, basert på `%`. Den er omtrent hva `*` er for Linux-mønstre, men for targets og dependencies.

F.eks. om man har definert

```makefile
m%n.c: math.h stat.h
    echo "Hello m$*n.c"
    echo "Target called was $@ with pattern $*" 
```
og kaller

```bash
% make main.c.
```

får man (med klammer innsatt for å understreke matchingene):

```text
Hello m[ai]n.c
Target called was [main.c] with pattern [ai]
```

Og  kaller man isteden

```bash
% make moon.c
```

får man

```text
Hello m[oo]n.c
Target called was [moon.c] with pattern [oo]
```

Tilsvarende kan `%` benyttes i dependencies, eller både i target og dependencies.

Eksemplene over er illustrative, men det er er gjerne ikke slik `%` utnyttes i praksis. Vanligere er at `%` representerer filnavn som kan ha ulike endelser.

Her er et eksempel:

```makefile
%.epub: %.md %.adoc
	pandoc $*.md -o $@
	asciidoctor-epub3 $*.adoc -o $@
```

Denne kan kalles f.eks. med

```bash
% make report
```

Da blir

- `$@` → report.epub
- `$*` → report

slik at

- `$*.md` → `report.md`
- `$*.adoc` → `report.adoc`

Dvs. man får utført:

```bash
% pandoc report.md -o report.epub
% asciidoctor-epub3 $*.adoc -o report.epub
```

Dette illustrer mønstrene greit, selv om strengt tatt output fra kommando 2 her skriver over output fra kommando 1.

Nå som vi har alle variablene våre tilgjengelig, kan vi godt produsere f.eks. **report-1.epub** fra første og **report-2.epub** fra andre kommando isteden:


```makefile
%.epub: %.md %.adoc
	pandoc $*.md -o $*-1.epub
	asciidoctor-epub3 $*.adoc -o $*-2.epub
```

Og siden vi kan få ut suffix fra target også, kan man om ønskelig erstatte dette med:

```makefile
%.epub: %.md %.adoc
	pandoc $*.md -o $*-1$(suffix $@)
	asciidoctor-epub3 $*.adoc -o $*-2$(suffix $@)
```

---

## Funksjoner

Her er en oversikt over tilgjengelig funksjoner. De kan referere seg både til targets og til dependencies (som jo også er targets):

| Funksjon        | Hva den gjør            |
| --------------- | ----------------------- |
| `$(basename …)` | Fjerner suffix fra fil  |
| `$(suffix …)`   | Henter suffix           |
| `$(notdir …)`   | Fjerner katalog         |
| `$(dir …)`      | Henter katalog          |
| `$(wildcard …)` | Matcher filer           |
| `$(patsubst …)` | Bytter ut mønster       |
| `$(subst …)`    | Enkel tekstsubstitusjon |


`$*` og `$(basename …)` ser nokså like ut, og for et enkelt fil-target er de identiske. Men for et pattern rule presenterer `$*` kun teksten som `%` matcher, mens `$(basename $@)` alltid representerer hele target-navnet uten suffix.

Vi lar de to substitusjonsfunksjonene ligge til en annen gang, og de øvrige er relativt selvforklarende.

## Tilordninger

De fleste tilordningene er oppsummert i følgene tabell

| Oper. | Evaluering     | Kort forklaring                          |
| ----- | ---------------| ---------------------------------------- |
| `=`   | ved bruk       | evalueres hver gang variabel brukes      |
| `:=`  | ved definisjon | evalueres én gang når Make leser fila    |
| `?=`  | ved definisjon | settes bare hvis variabel ikke finnes    |
| `+=`  | typeavhengig   | legger til tekst i eksisterende variabel |
| `!=`  | ved definisjon | kjører shell-kommando og lagrer output   |


Fir å utdype:

- `A = B` (rekursiv / deferred)
	- Verdien til A bestemmes når variabelen brukes, ikke når den defineres.
	- Hvis B endrer seg senere, vil A bruke den nye verdien.
- `A := B` (simple / immediate)
  - B evalueres med en gang når linjen leses, og resultatet lagres i A.
  - Hvis B endrer seg senere, påvirker det ikke A.
- `A ?= B` (betinget tilordning)
  - A settes til B bare hvis A ikke allerede er definert. Dette gjelder også hvis A kommer fra:
    - kommandolinjen (`make A=x`)
    - miljøvariabler
    - tidligere i Makefile
- `A += B` (append)
  - B legges til (med space) på slutten av A.
  - Hvordan B evalueres avhenger av hvordan A opprinnelig ble definert:

Disse utnyttes ikke minst når man skal sette opp en sammensatt kompileringsjobb, med flere filer, biblioteker, opsjoner og den slags.

## Conditionals

Make har noen betingelseskonstruksjoner (conditionals) som brukes når Make leser Makefile, ikke når kommandoene kjøres. De kan anvendes på variable og automatiske variable. De kan ikke anvendes i recepies (men `sh` har selvsagt egne `if`-konstruksjoner som kan utnyttes der). En liste conditionals med en forklaringer er her:

| Konstruksjon  | Betydning                        |
| ------------- | -------------------------------- |
| `ifeq (A,B)`  | sann hvis `A` og `B` er like     |
| `ifneq (A,B)` | sann hvis `A` og `B` er ulike    |
| `ifdef VAR`   | sann hvis `VAR` er definert      |
| `ifndef VAR`  | sann hvis `VAR` ikke er definert |
| `else`        | alternativ gren                  |
| `endif`       | avslutter blokken                |

Ohg her ser vi et eksempel:

```makefile
CC ?= gcc
DEBUG ?= 0
CFLAGS = -Wall

ifeq ($(DEBUG),1)
CFLAGS += -g
else
CFLAGS += -O2
endif

prog: prog.c
	$(CC) $(CFLAGS) -o $@ $<
```

Dette skjer:

1. `CC ?= gcc` : Kompilator settes til **gcc**, om ikke brukeren sier annet i kallet
2. `DEBUG ?= 0`: Setter en default-verdi hvis brukeren ikke har satt den i Make-kallet.
3. `ifeq ($(DEBUG),1)`: Make sjekker om verdien av DEBUG er 1 mens Makefile leses.
4. Avhengig av resultatet får CFLAGS enten verdien 
`-Wall -g` eller `-Wall -O2`

Dvs. at om bruker utfører

```bash
% make prog DEBUG=1
```

ender man opp med å utføre:

```bash
% gcc -Wall -g -o prog prog.c
```

Minner om at

- `$@` er navnet på target
- `$<` er den første avhengigheten

Dersom brukeren også hadde ønsket annen kompilator, kunne han utført:

```bash
% make prog DEBUG=1 CC=clang
```

## Parsing av Makefile

Når man kjører en kommando som `make epub`, gjørs det følgende:

Make 

1. leser hele Makefile fra toppen og ned  (og evt. inkluderte filer)

2. evaluerer conditionals (ifeq, ifdef, ...)

3. registrerer
	- variabler
	- targets
	- dependencies
	- pattern rules

4. bygger en dependency-graf

5. bestemmer hvilke targets som må bygges + utfører immediate evaluering (`:=` og `!=`)
6. *recepies* begynner å kjøre
   - automatiske variabler får verdier (`$@`, `$<`, `$^`, `$*`)
   - variabler ekspanderes
   - shell-kommandoer kjøres ( i `sh`)

Conditionals påvirker dermed hva som finnes i Makefile, ikke hva som skjer i recipes

Dersom man bare gjør kallet

```bash
% make
```

vil første target kjøres. Mange har derfor tidlig i Makefilen

```makefile
.PHONY: all ... clean
all: target1 target2 ... clean
```

som sørger for disse målene bygges i denne rekkefølgen (så fremt de er endret og må bygges), samt at **all** og **clean** aldri blir tolket som fil.

Man man kan også eksplisitt sette default target ved `.DEFAULT_GOAL := <target>`.

## Generelle tips

- Del opp Makefile i seksjoner med kommentarer.

- Bruk `?=` for defaults som brukeren kan overstyre via `make VAR=value`.

- Bruk automatic variables (`$@`, `$<`, `$^`, `$*`) for generiske regler.

- Bruk pattern rules (`%`) for å redusere repetisjon.

- Legg variabler og conditionals øverst – gjør Makefile mer lesbar.

- Test Makefile med `make -n` for å se hva som ville kjørt uten å faktisk kjøre shell-kommandoene.

- Bruk `make -B` eller `touch` for å tvinge rebuild for testing.


### Makefile-eksempel

```Makefile
#==============================================
# Klassisk Makefile-layout
# ==============================================

# 1️⃣ Default goal
.DEFAULT_GOAL := all

# 2️⃣ Phony targets (merkelapper, ikke filer)
.PHONY: all clean build test

# 3️⃣ Variabler
CC      ?= gcc             # Kompilator, kan overstyres på kommandolinje
CFLAGS  := -Wall           # Vanlige flags
DEBUG   ?= 0               # Default debug=0

ifeq ($(DEBUG),1)
CFLAGS += -g               # Debug-symboler
else
CFLAGS += -O2              # Optimalisering
endif

# 4️⃣ Hoved-target (all bygger alle nødvendige mål)
all: build test

# 5️⃣ Fil-targets
prog.o: prog.c util.h
	$(CC) $(CFLAGS) -c $< -o $@

util.o: util.c util.h
	$(CC) $(CFLAGS) -c $< -o $@

build: prog.o util.o
	$(CC) $(CFLAGS) $^ -o prog

# 6️⃣ Phony targets
test:
	@echo "Running tests..."
	# her kan du legge til testkommandoer

clean:
	rm -f prog *.o
```