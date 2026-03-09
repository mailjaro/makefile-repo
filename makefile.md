# Litt om Makefiles

## Intro

En **Makefile** er en konfigurasjonsfil som brukes av verktøyet `make` for å automatisere bygging av programmer og andre repeterende oppgaver. Den brukes ofte i programvareutvikling for å kompilere kildekode, kjøre tester, generere dokumentasjon eller pakke prosjekter, men kan også erstatte tradisjonelle shell-scripts for å kjøre sett av kommandoer.

### Hva løser Make?

I større prosjekter kan det være mange filer som må bygges i riktig rekkefølge. Hvis én fil endres, trenger man ofte bare å bygge deler av prosjektet på nytt. **make** analyserer **avhengigheter mellom filer** og bygger kun det som er nødvendig.

Dette gir:

- raskere bygging
- automatisert arbeidsflyt
- færre manuelle feil
- en standardisert måte å bygge prosjektet på

Selv om mange moderne byggesystemer eksisterer (som CMake, Gradle eller npm-scripts), er **make** fortsatt svært utbredt fordi det er:

- enkelt
- fleksibelt
- tilgjengelig på nesten alle Unix-lignende systemer
- godt egnet til små og mellomstore prosjekter

Makefiles brukes derfor fortsatt i mange open source-prosjekter, systemprogramvare og innebygde systemer.

Vi skal se nærmere på:

- spesialnotasjon i Makefiles
- variabler og hvordan de brukes
- automatiske variabler
- mønsterregler (pattern rules)
- vanlige konvensjoner og beste praksis

---

### Grunnleggende struktur

En Makefile består vanligvis av regler (*rules*). En regel beskriver:

1. Et mål (*target*)
2. Avhengigheter (dependencies)
3. Kommandoer (*recipes*)

Den grunnleggende syntaksen for en regel er:


```makefile
target: dependency1 dependency2
    command1
    command2
```

  **Target** kan enten være
  - filer (som typisk skal produseres), eller
  - merkelapper (såkalte *Phony targets*).
  
  (Phony targets deklareres med `.PHONY` for å unngå konflikter med eksisterende filer og blir alltid kjørt.)

**Dependencies** (valgfrie) er andre targets regelen er avhengig av.

- Hvis en dependency ikke har regel, antar **Make** at den representerer en fil
- Hvis en dependency-fil ikke eksisterer, rapporteres feil.
- For fil-targets bygges målet kun hvis det mangler, eller om en dependency er nyere enn target.

**Commands** er shell-kommandoer som kjøres. Disse må starte med en TAB, ikke mellomrom.

**Make** kjører kommandoene i et shell, vanligvis **/bin/sh**. Det betyr at man kan bruke vanlige shell-konstruksjoner som:

```makefile
build:
	mkdir -p build
	cp src/* build/
```

I dette eksempelet var target en phony target med navnet build.

Her ser vi et eksempel med et fil-target:

```makefile
report.pdf: report.adoc
	asciidoctor-pdf report.adoc -o report.pdf
```

Målet **report.pdf** produseres ved

```bash
% asciidoctor-pdf report.adoc -o report.pdf
```

dersom PDF-filen mangler eller om **report.adoc** er nyere.

---

## Variable

En variabel defineres slik:

```makefile
NAME = value
```

De refereres med

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
    mkdir -p $(BUILD_DIR)

clean:
    rm -rf $(BUILD_DIR)
```

Variabler kan også settes når **make** kjøres:

```bash
% make cfiles TARGET=Developers
```

Variabler kan også settes når **make** kjøres:

```
make hello TARGET=Developers
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

Anvendt på target **fil.txt**, representerer eksempelvis `$(suffix $@)` endelsen `.txt`. Dette er egentlig en funksjon, og vi skal se på noen flere slike etter hvert.

Men la oss se nærmere på de kanskje viktigste variablene for oss.

### `$@` 

`$@` representerer **målnavnet**.

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

`$<` representerer **første dependency** i regelen.

Eksempel:

```makefile
print-file: message.txt remember.txt
	cat $<
```

Her vil `$<` bli erstattet med `message.txt`, slik at innholdet av denne vil bli printet til skjerm ved kallet `make print-file`


### `$^`

`$^` representerer **alle dependencies** i regelen.

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

`$*` representerer basenavnet til målet.

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

Nært beslektet er såkalte *pattern rules*, eller mønsterregler, basert på `%`. Den er omtrent hva `*` er for Linux-mønstre for targets og dependencies.

F.eks. om man har definert

```makefile
m%n.c: math.h stat.h
    echo "Hello m$*n.c"
    echo "Target called was $@ with pattern $*" 
```
og kaller 1)

```bash
% make main.c.
```

da får man (med klammer innsatt for å understreke matchingene):

```text
Hello m[ai]n.c
Target called was [main.c] with pattern [ai]
```

Og  kaller man isteden 2)

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

Her er en oversikt over tilgjengelig funksjoner. De kan kan referere seg både til targets og til dependencies (som jo også er targets):

| Funksjon        | Hva den gjør            |
| --------------- | ----------------------- |
| `$(basename …)` | Fjerner suffix fra fil  |
| `$(suffix …)`   | Henter suffix           |
| `$(notdir …)`   | Fjerner katalog         |
| `$(dir …)`      | Henter katalog          |
| `$(wildcard …)` | Matcher filer           |
| `$(patsubst …)` | Bytter ut mønster       |
| `$(subst …)`    | Enkel tekstsubstitusjon |


Vi ser at `$*` og `$(basename …)` ser nokså like ut. Og for et enkelt fil-target er de identiske. Men for et pattern rule presenterer `$*` kun teksten som `%` matcher, mens `$(basename $@)` alltid representerer hele target-navnet uten suffix.


