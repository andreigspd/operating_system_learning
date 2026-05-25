# Modulul T4 — Ghid Dezvoltator (High-Level Overview)

## 📌 Ce se întâmplă în acest modul?
Modulul **T4** înlocuiește modelul SPMD concurent din T3 cu o **arhitectură performantă multiproces de tip Manager-Worker**. Inventarierea directorului rădăcină este distribuită în mod dinamic către un număr `N` de procese copii. Comunicarea stării și a datelor se face extrem de rapid prin intermediul unei zone de memorie partajată în RAM, iar excluderea mutuală și controlul fluxului (backpressure) sunt gestionate prin semafoare POSIX proces-partajate.

---

## 📁 Structura Codului Sursă (`src/`)

În folderul `src/` se găsește logica de execuție a managerului și a workerilor:

1. **[main_fileops_manager.c](./src/main_fileops_manager.c)**:
   - Reprezintă coordonatorul execuției. Validează datele CLI și creează/inițializează fișierul IPC mapat în memorie (`data/ipc.mmap`).
   - Depune job-ul rădăcină în coadă și folosește `fork()` și `exec()` pentru a lansa cele `N` procese `fileops_worker`.
   - Într-o buclă neblocantă, managerul citește rezultatele depuse de workeri în bufferul circular din memoria partajată și le copiază într-un buffer local dinamic.
   - La terminare, culege asincron exit code-urile workerilor prin `waitpid()`, colectează statisticile de utilizare CPU partajate de aceștia și scrie atomic baza de date finală (`data/inventory.db`) folosind un fișier temporar redenumit la sfârșit.
   - Implementează modurile speciale `--verify` (pentru validarea structurală a DB-ului) și `--dump` (pentru interogarea metadatelor în format text simplu).

2. **[main_fileops_worker.c](./src/main_fileops_worker.c)**:
   - Reprezintă elementul de execuție paralelă. Fiecare worker atașează zona IPC comună prin `mmap`.
   - Preia directoarele de scanat din coada partajată. La descoperirea de subdirectoare le depune înapoi ca job-uri noi (dacă nu depășesc adâncimea maximă de scanare `--max-depth`).
   - La descoperirea de fișiere regulate, le extrage metadatele și le calculează hash-ul SHA256 prin rularea asincronă a comenzii Linux `sha256sum` printr-o conductă `popen()`, decodificând rezultatul binar.
   - Depune rezultatele în bufferul circular de rezultate, blocându-se în semafoarele de *Backpressure* dacă managerul nu citește suficient de repede.
   - Înainte de exit, citește statisticile proprii de resurse folosind `getrusage(RUSAGE_SELF)` și le publică în memoria partajată.

---

## 🛠️ Compilare și Execuție rapidă

Modulul poate fi rulat unitar prin intermediul scriptului centralizator:

```bash
# Compilare utilitare T4
./tools/fileops.sh build

# Rulare inventar în mod multiproces cu 4 workeri în paralel
./tools/fileops.sh run -- fileops_manager --root /cale/director --workers 4 --ipc data/ipc.mmap --db data/inventory.db

# Verificare structurală a bazei de date rezultate
./tools/fileops.sh run -- fileops_manager --db data/inventory.db --verify

# Dump de metadate
./tools/fileops.sh run -- fileops_manager --db data/inventory.db --dump
```

---

## 📚 Unde se găsesc specificațiile tehnice detaliate?
Specificațiile la nivel scăzut despre layout-ul memoriei partajate, semafoare, controlul backpressure-ului sau formatul binar al bazei de date pe disc se află în fișierele dedicate din directorul `doc/`:
👉 **[MMAP_PROTOCOL.md](./doc/MMAP_PROTOCOL.md)** — Protocolul memoriei mapate, structura cozilor circulare de joburi/rezultate și semafoarele.
👉 **[T4_DB_FORMAT.md](./doc/T4_DB_FORMAT.md)** — Structura bazei de date `inventory.db` (header, file records, worker stats) și algoritmul de validare din verify mode.
