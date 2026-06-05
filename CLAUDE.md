# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Cortalys DBC** ("Outils Clipper pour DBC") is a **WinDev 30** (PC SOFT) desktop application written in **WLanguage (French)**. It is a companion toolbox for the **Clipper** ERP: it connects to Clipper's HFSQL Client/Server database and provides operator tools. It builds a single 64-bit executable, `Cortalys_DBC_Tools.exe` (versioned copies live in `Exe/`, e.g. `Cortalys_DBC_Tools_1.0.14.exe`).

The five tools (see the menu arrays in `Cortalys_DBC.wdp` project init code) are:
- **Création des BL** (`FEN_CreationBL` / `Fen_CreerBL`) — delivery-note creation
- **Export Fabera** (`FEN_Export_Fabera`)
- **Import Qualitime** (`FEN_Import_Qualitime`)
- **Design d'étiquette** (`FEN_DesignEtiquetteEnsacheuse`) — label designer for the bagging machine ("ensacheuse")
- **Edition d'étiquette** (`FEN_EditionEtiquette`)

## Critical: how to work with this codebase

**These are WinDev IDE files, not plain source.** Each `.wd*` file is a text container (YAML-ish) where WLanguage code is embedded inside `code : |1-` / `code : |1+` blocks. The bulk of every file is `internal_properties` — **base64-encoded binary blobs the WinDev IDE owns**. Each editable file also has a parallel `*.cache` file mirroring it.

Consequences:
- **Prefer editing in the WinDev IDE.** Hand-editing WLanguage text inside `code:` blocks is possible for small, surgical changes, but never touch `internal_properties`, identifiers, or the `*.cache` files.
- The build is done by the **WinDev IDE** (Project > Generate). There is **no CLI build, lint, or test toolchain** in this repo — do not invent npm/make/gradle commands.
- Code and identifiers are **in French** (WLanguage keywords: `procédure`, `SI/ALORS/SINON/FIN`, `RENVOYER`, `POUR TOUT`, `est une chaîne`, `_ET_`/`_OU_`, etc.). Keep new code French and match surrounding style.
- Encrypted/secured string resources appear inline as `<§@...§>` tokens — these are managed multilingual/secured strings, **not** plain text to edit.

## File-type conventions

| Prefix / ext | Type | Examples |
|---|---|---|
| `FEN_` `.wdw` | Window (Fenêtre) | `FEN_Configuration`, `FEN_DesignEtiquetteEnsacheuse` |
| `CL_` `.wdc` | Class | `CL_Param`, `CL_Log` |
| `.wdg` | Procedure set (collection) | `ProcéduresGlobales`, `ProcéduresClipper`, `ProcéduresEtiquette` |
| `ETAT_` `.wde` | Report (État) | `ETAT_Etiquette` |
| `REQ_` `.WDR` | Query (Requête) | `REQ_EtiquetteEnsacheuse` |
| `.wdp` | Project file (entry point + global constants + startup code) | `Cortalys_DBC.wdp` |
| `Analyse/CLIPPER8.wda` | Analysis = HFSQL data model (the Clipper ERP schema) | |
| `.fic` `.ndx` `.mmo` | HFSQL data files (binary) | |

## Architecture & key flows

**Startup** (project init code in `Cortalys_DBC.wdp`): loops on `DbInit()` until the DB connection succeeds (opening `FEN_Configuration` on failure) → `ProcéduresClipper.Initialise()` → opens the window named by the `fen` command-line arg, else the configured default (`APP_DefaultWindow` param), else `FEN_Accueil`. Command-line keys: `fen` (window) and `operateur` (user).

**Database** (`ProcéduresGlobales.DbInit`): HFSQL Client/Server via a `DbcConnexion`. Connection settings come from parameters (server/port/user/password/name/crypto). The DB password is AES-128 encrypted via `ParamDécrypte` (key `CleCryptageIni`). `HChangeConnexion("*", DbcConnexion)` rebinds all files to this connection. The app reads/writes the live Clipper schema (`CLIPPER8`) — treat all data access as production ERP data.

**Parameters** (`CL_Param`, file `PARAM`): central key/value store (`COPAR` code → value). `CL_Param` caches all params in an associative array (`m_taParametres`) loaded by `Initialisation()`; `Rafraichit()` does an incremental reload using `HVersion`. HFSQL triggers call `ApresHAjouteParam` / `ApresHModifieParam` / `AvantHSupprimeParam` to keep the cache coherent. Helpers: `Lecture`, `LectureValeur`, `ParamOui`, `Sauve`, `Existe`, `CommencePar`/`FinitPar`/`Copar_Contient`. The global `ChargeParamètre`/secured-string helpers wrap this. Use `CL_Param`, don't re-read `PARAM` directly.

**Clipper integration** (`ProcéduresClipper`): holds Clipper-specific state and helpers (e.g. `FormatDecimal` builds input masks from decimal-count params like `DECIVTE`/`DECIACH`/`DECINOM`). Initialized once at startup.

**Label engine** (`ProcéduresEtiquette.DessineAperçu`): renders a label to an `Image` at 150 DPI (`nEchelle = 5.9055` px/mm) from a **JSON layout** (`stItemEtiquette` array). Item types: `Libellé`, `Champ`, `Image`, `Code barre`. `Champ`/barcode values are **dynamic WLanguage expressions** evaluated via `{stItem.sValeur}` indirection. Data is resolved by walking the Clipper chain: `gamme → engam → affaire → commande → art → client`. Barcode formats: EAN8/EAN13/EAN128, Datamatrix, QR Code, Code 39.

**Logging** (`CL_Log`): simple logger with INFO/WARN/ERR levels, writing to a file and/or a UI field, with optional auto-flush.

**OCR**: Tesseract trained data (`eng`/`fra`/`spa`) ships in `Exe/` — some tool(s) use OCR.

## Git notes

- `.gitignore` already excludes WinDev build/backup artifacts: `*.cache`, `*.exe`, `*.env`, `*.waudit`, `Sauvegarde*`, `Svg*`, `/Exe`, `/Historique`, `cache.gestion de sources`, etc. Note `.cache` files are ignored — commit only the source `.wd*` files.
- The `Analyse/` data dictionary is partly tracked (`defapp.fic` etc. appear in status); `*.mmo` blobs are not. Avoid committing HFSQL `.fic/.ndx/.mmo` data unless intentional.
- Folders like `Sauvegarde*`, `Svg_*`, `Sauvegarde_Version15/21` are IDE snapshots — ignore them when searching/reading; the live files are at repo root.
- Commit messages in history are in French.
