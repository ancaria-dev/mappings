<div align="center">

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org)
[![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)](https://ancaria.dev)
[![License](https://img.shields.io/badge/License-MIT-4B5563?style=for-the-badge)](LICENSE)

[Русский](README.md) · [English](README.EN.md)

</div>

# mappings

[![Lines of code](https://img.shields.io/endpoint?url=https%3A%2F%2Fancaria.dev%2Ffiles%2Fbadges%2Fmappings.json)](https://github.com/ancaria-dev/mappings)

Das Adressregister von Sacred Gold für alle, die dem Loader einen Hook
hinzufügen oder den Code des Spiels untersuchen.

Jede Zeile benennt eine Funktion, eine globale Variable oder ein Strukturfeld,
nennt Adresse und Aufrufkonvention und sagt, wie sicher die Angabe ist. Die
Notizen halten Belege, Grenzen und jede gescheiterte Hook-Stelle fest. So muss
niemand das Spiel zweimal aus demselben Grund abstürzen lassen.

Alle Adressen gelten für genau einen Build: die 32-Bit-`pureHD.exe` v2.0.2.118
mit Image Base `0x00400000`. Das ist ein Community-Wrapper, der `pHD.dll`
mitbringt. Die originale `Sacred.exe` ist eine andere Binärdatei, und für sie
wie für jeden anderen Build sind diese Adressen falsch.

Ich habe die Adressen mit Frida, Cheat Engine und Ghidra an meiner eigenen
Offline-Kopie gefunden. Der Loader ändert keine Spieldateien auf der
Festplatte. Seine Hooks leben nur im Arbeitsspeicher und verschwinden mit dem
Spielprozess.

| Datei | Was sie ist |
|---|---|
| `mappings.txt` | Das Register und die einzige Datei, die du von Hand bearbeitest |
| `mappings.json` | Wird aus dem Register erzeugt und von den anderen Repositories gelesen |
| `mappings.generator.py` | Erzeugt und prüft `mappings.json` |

## Erste Schritte

So fügst du eine Zeile hinzu:

1. Öffne `mappings.txt` und such den passenden Abschnitt.
2. Füg die Zeile ein. Die Spalten sind für Menschen ausgerichtet. Der Generator
   liest nur die ersten beiden durch Leerraum getrennten Felder.
3. Führ `python mappings.generator.py` aus.
4. Committe `mappings.txt` und `mappings.json` gemeinsam.

Jeder Abschnitt hat eine eigene Kopfzeile:

```
# VA          RVA        NAME                              CONV/ARGS -> RET                        CONF       NOTES

0x00604380  0x204380   cObjectManager::getLocalHero      thiscall() -> cCreatureHero*             confirmed  [key=getLocalHero hooked] NULL on the main menu. The hero-capture point for every hook.
```

Code-Zeilen tragen beide Adressen, und die RVA ist die VA minus `0x00400000`.
Frida arbeitet mit der RVA, Ghidra und Cheat Engine mit der VA. Verwechselst du
sie, gibt es keinen Fehler. Der Hook feuert einfach nie.

Globale Variablen tragen nur eine VA. Zeilen in `STRUCTURES` beginnen mit `+0x`
und beschreiben Offsets, keine Adressen.

`CONF` kennt genau drei Werte:

- `confirmed`: im laufenden Spiel beobachtet.
- `static`: aus dem Disassembly gelesen, zur Laufzeit noch nicht beobachtet.
- `suspect`: plausibel, aber ungeprüft oder im Widerspruch zu anderen Belegen.

Eine falsche Zeile kostet mehr als eine fehlende. Lass keine Zeile ohne Beleg
aus dem laufenden Spiel auf `confirmed`. Hält ein Beleg nicht mehr, stuf die
Zeile auf `suspect` herab und schreib den Grund in die Notizen.

Manche Zeilen gibt es nur, um vor einer gefährlichen Stelle zu warnen:

```
0x00562D10  0x162D10   regen write                        mov [ebp+0x130], edi                     confirmed  DO NOT HOOK: per creature per tick (~10/s for the player alone).
```

Lösch nie eine Zeile. Korrigier sie oder markier sie als `suspect` und halt
den Grund in den Notizen fest. Auch `mappings.json` bearbeitest du nie von
Hand: Der nächste Generatorlauf überschreibt die Datei.

### Wenn Code eine Zeile beim Namen braucht

Setz einen Export-Tag an den Anfang der Notizspalte:

- `[key=addExperience]` exportiert die Adresse als `rva.addExperience`, bei
  einer globalen Variable mit nur einer Adresse als `va.addExperience`.
- `[key=hpDamage hooked]` trägt den Namen zusätzlich in `hooked` ein. Der Agent
  setzt dort einen Interceptor, also muss `hooksafe.py` die Stelle prüfen.

Eine Zeile ohne Tag bleibt Dokumentation und landet nie in `mappings.json`.
Schlüssel sind in der ganzen Datei eindeutig. Beschreiben zwei Zeilen dieselbe
Adresse, darf nur eine davon einen Schlüssel tragen.

## Aufbau von mappings.json

Die Datei hat drei Arbeitsabschnitte und die erklärenden Felder `_` und
`_hooked`:

```json
{
  "rva": { "getLocalHero": "0x204380", "hpDamage": "0x16FC44" },
  "va": { "xorMirror1": "0x182DDDC" },
  "hooked": ["getLocalHero", "hpDamage"]
}
```

Zeilen mit zwei Adressen landen in `rva`, globale Variablen mit einer Adresse
in `va`. `hooked` listet die Interceptor-Stellen. Der Generator prüft die RVA
aus der Zeile und berechnet den exportierten Wert dann aus der VA.

## Wer das Register liest

`coderpack/tools/paths.py` sucht `mappings.json` in dieser Reihenfolge: ein Pfad
auf der Kommandozeile, `$CODERPACK_MAPPINGS`, das Nachbarverzeichnis
`../mappings`, der Cache unter `build/mappings/mappings.json` und zuletzt ein
Download von GitHub. Der Download nimmt die Revision aus
`coderpack/.mappings-ref`, oder `master`, wenn die Datei fehlt oder leer ist.

`agent/tools/addr.py` im Repository [agent](https://github.com/ancaria-dev/agent)
macht aus `rva` und `va` die Datei `agent/src/gen/addr.js`. Eine von Hand in den Agenten geschriebene
Adresse ist ein Fehler.

`coderpack/tools/hooksafe.py` disassembliert das Spiel an jedem Namen aus
`hooked` und meldet Gefahren für das Trampolin. Tödlich sind ein Sprung in die
überschriebenen Bytes, eine Flags setzende Instruktion, die von ihrem Sprung
getrennt wird, und überlappende Hooks. Außerdem warnt das Werkzeug vor
verschobenem Kontrollfluss, ESP-relativen Instruktionen und Hilfsregistern,
die den Patch überleben müssen.

Der Skill-Schreibzugriff bei `+0x1827DA` zeigt, warum die Sprungprüfung zählt.
Seine Instruktion ist vier Byte lang, die nächste liegt also im Patch, und zwei
Sprünge zielen genau darauf. Der Hook dort ließ das Spiel abstürzen, sobald ein
Charakter mit leerem Skill-Slot geladen wurde. Von Hand hat die Suche drei
Neustarts des Spiels gekostet. Die sichere Stelle ist `+0x1827DE`.

`launcher/tools/build.ps1` erzeugt `addr.js` neu, wenn es coderpack aus den
Quellen baut. `$Mappings` reicht es nur weiter, wenn dort `mappings.json`
liegt. Sonst sucht coderpack das Register auf dem üblichen Weg.

## Bauen

Der Generator braucht Python 3.11 oder neuer und nichts außer der
Standardbibliothek:

```
python mappings.generator.py            # schreibt mappings.json
python mappings.generator.py --check    # endet mit 1, wenn die Datei veraltet ist
```

`--check` baut das JSON im Speicher, vergleicht es mit der eingecheckten Datei
und schreibt nichts. Stimmen beide überein, gibt es
`mappings.json is up to date.` aus und endet mit 0. Sonst gibt es den Befehl zum
Neuerzeugen aus und endet mit 1. Die CI führt dieselbe Prüfung bei jedem Push
und Pull Request aus.

In beiden Modi prüft der Generator das Register und lehnt ab:

- eine fehlerhafte VA oder ein fehlerhaftes zweites Feld, das mit `0x` beginnt
- eine RVA, die nicht `VA - 0x00400000` ist
- `[key=...]` in einer Zeile ohne Adresse
- einen fehlerhaften Tag-Inhalt
- einen doppelten Schlüssel, samt der Zeile, in der er zuerst vorkam
- ein alleinstehendes `[hooked]`
- `hooked` bei einer globalen Variable mit einer Adresse
- ein Register ohne exportierte RVA

Fehler zu einer bestimmten Zeile nennen ihre Nummer.

Das Parsen beginnt bei der ersten Zeile, die mit `##` anfängt. Der Kopf darüber
enthält Tag-Beispiele und wird absichtlich übersprungen, genau wie jede
Adresszeile dort. In einer Zeile mit zwei Adressen muss die RVA im zweiten Feld
stehen. Sonst hält der Generator sie für eine globale Variable und exportiert
die VA unter `va`.

## Releases

Das Register hat keine Releases. Die Nutzer lesen `mappings.json` von `master`
oder von der Revision, die in `coderpack/.mappings-ref` festgelegt ist. Bei den
Spielern kommt eine Änderung mit dem nächsten coderpack-Release an, wie in der
[CONTRIBUTING](https://github.com/ancaria-dev/.github/blob/master/CONTRIBUTING.DE.md)
im Wurzel-Repository beschrieben.

## Lizenz

MIT, siehe [LICENSE](LICENSE).
