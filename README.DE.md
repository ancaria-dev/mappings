<div align="center">

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org)
[![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)](https://ancaria.dev)
[![License](https://img.shields.io/badge/License-MIT-4B5563?style=for-the-badge)](LICENSE)

[Русский](README.md) · [English](README.EN.md)

</div>

# mappings

Dieses Repository verzeichnet, was über das Innere von Sacred Gold bekannt ist:
Adressen von Funktionen, Aufrufkonventionen, Felder an Struktur-Offsets und den
jeweiligen Stand der Verifizierung.

Alle Angaben gelten ausschließlich für `pureHD.exe` v2.0.2.118, 32 Bit, mit der
Image Base `0x00400000`. Diese Binärdatei ist ein Community-Wrapper, der
`pHD.dll` mitbringt. Die originale `Sacred.exe` ist eine andere Binärdatei. Für
jeden anderen Build sind die Adressen falsch.

Das Adressregister liegt in einem eigenen Repository. Dadurch kann sich der
verbrauchende Code ändern, ohne dass die Arbeit zur Ermittlung der Adressen
verloren geht.

| Datei | Zweck |
|---|---|
| `mappings.txt` | Das von Hand gepflegte Adressregister |
| `mappings.json` | Die daraus erzeugte Datei, die andere Repositories einlesen |
| `mappings.generator.py` | Der Generator und Validator für `mappings.json` |

## Zeile hinzufügen

Öffne `mappings.txt`, suche den passenden Abschnitt und füge dort eine Zeile
hinzu. Die Spalten sind zur besseren Lesbarkeit ausgerichtet. Für die Auswertung
trennt der Generator die ersten beiden Felder anhand von Leerraum. Jeder
Abschnitt hat eine eigene Kopfzeile:

```
# VA          RVA        NAME                              CONV/ARGS -> RET                        CONF       NOTES

0x00604380  0x204380   cObjectManager::getLocalHero      thiscall() -> cCreatureHero*             confirmed  [key=getLocalHero hooked] NULL on the main menu. The hero-capture point for every hook.
```

Codezeilen enthalten beide Adressen. Die RVA ist die VA minus `0x00400000`.
Frida verwendet die RVA, Ghidra und Cheat Engine verwenden die VA. Eine
Verwechslung löst keinen Fehler aus. Der Hook feuert dann einfach nie.

Für `CONF` sind genau drei Werte zulässig:

- `confirmed`: im laufenden Spiel beobachtet
- `static`: aus dem Disassembly abgelesen, aber nicht zur Laufzeit beobachtet
- `suspect`: plausibel, aber ungetestet, oder durch andere Erkenntnisse
  widerlegt

Eine falsche Zeile verursacht mehr Arbeit als eine fehlende. Wenn die Belege
nicht für `confirmed` reichen, bleibt sie `static` oder `suspect`. Die
Notizspalte hält die Begründung fest. Manche Zeilen dienen ausschließlich als
Warnung vor ungeeigneten Hook-Stellen:

```
0x00562D10  0x162D10   regen write                        mov [ebp+0x130], edi                     confirmed  DO NOT HOOK: per creature per tick (~10/s for the player alone).
```

Zeilen werden nicht gelöscht. Korrigiere eine falsche Zeile oder setze ihren
`CONF`-Wert auf `suspect` und dokumentiere den Grund in den Notizen.

### Benannte Exporte

Soll Code eine Zeile über einen Namen verwenden, erhält sie ein Export-Tag.
Nach der Konvention des Registers steht dieses Tag am Anfang der Notizspalte:

- `[key=addExperience]` exportiert die Adresse als `rva.addExperience`, oder als
  `va.addExperience`, wenn es sich um ein Global mit nur einer Adresse handelt.
- `[key=hpDamage hooked]` exportiert die Adresse ebenfalls und nimmt den Namen
  zusätzlich in `hooked` auf. Der Agent hängt an dieser Stelle einen
  Interceptor ein. `tools/hooksafe.py` prüft sie deshalb auf Gefahren für das
  Trampolin.

Zeilen ohne Tag bleiben reine Dokumentation und gelangen nicht in
`mappings.json`. Schlüssel müssen in der gesamten Datei eindeutig sein. Zwei
Zeilen mit derselben Adresse dürfen zusammen höchstens einen Schlüssel tragen.
Ein Global mit nur einer Adresse darf nicht als `hooked` markiert werden, da es
dort keinen Code zum Einhängen gibt.

Erzeuge anschließend die JSON-Datei neu und committe beide Dateien gemeinsam:

```
python mappings.generator.py
```

Bearbeite `mappings.json` nie von Hand. Der nächste Generatorlauf überschreibt
solche Änderungen.

## Generator

Der Generator benötigt nur die Python-Standardbibliothek und läuft mit Python
3.11 oder neuer.

```
python mappings.generator.py            # writes mappings.json
python mappings.generator.py --check    # exits 1 if the file on disk is stale
```

Mit `--check` erzeugt der Generator den erwarteten Inhalt im Speicher und
vergleicht ihn mit `mappings.json`. Der Befehl schreibt nichts. Bei einer
veralteten oder fehlenden Datei endet er mit Exit-Code 1, andernfalls gibt er
`mappings.json is up to date.` aus und endet mit Exit-Code 0. Dieses Repository
hat keinen CI-Workflow, daher muss die Prüfung hier manuell ausgeführt werden.

Beide Modi validieren das Register beim Einlesen. Der Lauf bricht mit einer
Zeilennummer ab, wenn eine Adresse fehlerhaft formatiert ist, ein mit `0x`
beginnendes zweites Feld keine gültige RVA enthält, die RVA nicht
`VA - 0x00400000` entspricht, ein `[key=...]` auf einer Zeile ohne Adresse
steht, ein Tag ungültig aufgebaut ist, ein Schlüssel doppelt vorkommt, ein
nacktes `[hooked]` verwendet wird oder ein Global als `hooked` markiert ist.
Auch ein beschädigtes Register ohne exportierte RVA wird abgelehnt.

Die Auswertung beginnt mit der ersten Zeile, die mit `##` anfängt. Der Kopf
davor enthält Beispiele für Tags und wird bewusst übersprungen. Eine
Adresszeile oberhalb dieser Markierung wird ebenfalls nicht gelesen. Bei einer
Zeile mit zwei Adressen muss die RVA im zweiten Feld stehen. Andernfalls
behandelt der Generator die Zeile als Global und exportiert ihre VA nach `va`.
Struktur-Offsets beginnen mit `+0x` und dürfen kein Export-Tag tragen.

Die funktionalen Daten der flachen Ausgabe bestehen aus `rva`, `va` und
`hooked`. Zusätzlich enthält die Datei die erläuternden Felder `_` und
`_hooked`:

```json
{
  "rva": { "getLocalHero": "0x204380", "hpDamage": "0x16FC44" },
  "va": { "xorMirror1": "0x182DDDC" },
  "hooked": ["getLocalHero", "hpDamage"]
}
```

Bei Zeilen mit zwei Adressen berechnet der Generator den exportierten RVA-Wert
aus der VA, nachdem er die RVA in der Zeile geprüft hat. Globals mit einer
Adresse landen in `va`. `hooked` ist eine Liste der Namen, deren Tag das Wort
`hooked` enthält.

## Verwendung

`coderpack/tools/paths.py` sucht `mappings.json` in einer festen Reihenfolge.
Zuerst kommt ein Pfad aus der Kommandozeile, dann `$CODERPACK_MAPPINGS`, das
benachbarte Verzeichnis `../mappings` und der Cache unter
`build/mappings/mappings.json`. Fehlt die Datei weiterhin, lädt das Werkzeug sie
aus dem mappings-Repository auf GitHub. Dabei verwendet es den Ref aus
`coderpack/.mappings-ref` oder `master`, wenn diese Datei fehlt oder leer ist.

`tools/addr.py` erzeugt `agent/src/gen/addr.js` aus `rva` und `va`. Eine von Hand
in den Agent-Code eingetragene Adresse gilt als Fehler.

`tools/hooksafe.py` liest alle Namen aus `hooked`, disassembliert die Binärdatei
des Spiels und meldet Gefahren für das Trampolin. Ein Sprungziel innerhalb der
ersetzten Bytes, eine von ihrem Branch getrennte Flag-Instruktion oder zwei
überlappende Hooks gelten als fatal. Das Werkzeug warnt außerdem vor verlegtem
Kontrollfluss, ESP-relativen Instruktionen und temporären Registern, deren Wert
den Patch überdauern muss.

Beim Skill-Write auf `+0x1827DA` ist die Instruktion vier Byte lang. Zwei `jmp`
zielen auf die direkt folgende Instruktion, die damit im Patch-Bereich liegt.
Beim Laden eines Charakters mit einem leeren Skill-Slot stürzte das Spiel ab.
Die sichere Stelle liegt bei `+0x1827DE`. Die Ursache ließ sich erst nach drei
Neustarts von Hand eingrenzen.

`launcher/tools/build.ps1` erzeugt `agent/src/gen/addr.js` neu, wenn es
Coderpack aus den Quellen baut und das Payload vorbereitet. Enthält der Pfad
`$Mappings` eine `mappings.json`, übergibt das Skript ihn an
`python tools/addr.py`. Sonst ruft es den Befehl ohne Pfad auf und Coderpack
verwendet seine normale Suchkette.

Die Adressen wurden mit Frida, Cheat Engine und Ghidra anhand einer lokal
besessenen Offline-Kopie ermittelt. Dabei wird keine Spieldatei auf dem
Datenträger gepatcht. Alle Änderungen der Hooks finden im Arbeitsspeicher statt
und verschwinden mit dem Ende des Prozesses.

## Lizenz

Die Lizenz ist MIT. Der vollständige Text steht in [LICENSE](LICENSE).

---

Das Projekt begann als Proof of Concept für die Frage, ob ein Java-Mod für ein
altes Lieblingsspiel möglich ist. Es besteht kein Anspruch auf Support.
