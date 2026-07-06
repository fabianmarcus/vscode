# VS Code Settings als Submodul

Hier sind meine persönlichen VS Code Settings zusammengefasst.

Die Datei `settings.json` dient als wiederverwendbare Vorlage fuer Editor-, Suche-, Git-, Terminal- und TypeScript-Defaults in verschiedenen Projekten.

## Verwendung als Git-Submodul

Dieses Repository kann als zentrale Quelle in andere Projekte eingebunden werden, z. B. unter `.vscode/vscode-shared`.

```bash
git submodule add https://github.com/fabianmarcus/vscode .vscode/vscode-shared
git submodule update --init --recursive
```

Danach liegt die zentrale Settings-Datei im Zielprojekt unter:

- `.vscode/vscode-shared/settings.json`

Wichtig:

- VS Code liest standardmaessig die Datei `.vscode/settings.json` im Zielprojekt.
- Die Datei aus dem Submodul ist eine zentrale Vorlage und muss in der Regel nach `.vscode/settings.json` uebernommen werden.
- Projektspezifische Anpassungen bleiben im Zielprojekt.
- Updates aus diesem Repository werden im Zielprojekt ueber ein normales Submodul-Update nachgezogen.

Damit die Einstellungen greifen, muss unter macOS- oder Linux-Systemen ein Symlink direkt im `.vscode`-Verzeichnis gesetzt werden:

```bash
ln -sf .vscode/vscode-shared/settings.json .vscode/settings.json
```

Hinweis:

- Die Symlink-Variante ist fuer Unix-artige Umgebungen gut geeignet, aber weniger robust fuer plattformuebergreifende Teams.

## Git-Submodul aktualisieren

Wenn das Submodul bereits im Zielprojekt eingebunden ist, kann es spaeter auf den neuesten Stand aktualisiert werden.

Ein typischer Ablauf im Zielprojekt ist:

```bash
git submodule update --remote --merge
git add .vscode/vscode-shared
git commit -m "Update shared VS Code settings"
```

Alternativ kann das Submodul direkt im eingebundenen Verzeichnis aktualisiert werden:

```bash
cd .vscode/vscode-shared
git pull
cd ../..
git add .vscode/vscode-shared
git commit -m "Update shared VS Code settings"
```
