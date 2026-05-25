# FileOps & ProcOps Tools — Proiect Linux (SO 2026)

Acest repository conține realizarea incrementală a proiectului **FileOps & ProcOps Tools** pentru cursul de Sisteme de Operare (SO 2026). Proiectul constă într-un set complex de utilitare Linux care îmbină prelucrarea și indexarea arborilor de directoare (**FileOps**) cu monitorizarea și controlul proceselor active (**ProcOps**).

Aplicațiile sunt scrise exclusiv în **limbajul C** (standard C11, compilat cu `-Wall -Wextra -Werror`), utilizând doar apeluri de sistem standard **POSIX** și biblioteca standard C, fără dependințe externe, garantând portabilitatea și performanța maximă în medii Linux.

---

## 🚀 Structura și Evoluția Proiectului

Proiectul este structurat sub formă de module independente pentru fiecare etapă a laboratorului (T3, T4, T5). Fiecare modul aduce o arhitectură distinctă și o complexitate tehnică specifică:

### 📁 Organizarea directoarelor

Pentru fiecare temă, structura obligatorie de directoare este următoarea:
- `bin/` — Conține executabilele compilate.
- `src/` — Codul sursă C (`.c`).
- `include/` — Fișierele header C (`.h`).
- `data/` — Fișierele de date persistente (baze de date binare `.db`, protocolul `ipc.mmap`).
- `logs/` — Logurile de depanare și execuție.
- `reports/` — Rapoartele text generate (ex: diff-uri între baze de date).
- `tmp/` — Fișiere temporare necesare pentru asigurarea scrierilor atomice.
- `tests/` — Scripturile bash de testare automată.
- `doc/` — Documentația tehnică detaliată a formatelor binare și protocoalelor.
- `tools/` — Scriptul de orchestrare `fileops.sh`.

---

## 🛠️ Module de Implementare (T3, T4, T5)

### 📌 [Tema T3](./T3/) — Baze de date binare și actualizare concurentă (SPMD)
- **fileops_indexer**: Parcurge recursiv un arbore de directoare și salvează metadatele (cale absolută, tip, dimensiune, `mtime`, inod, device și un checksum XOR determinist al conținutului) într-o bază de date binară versionată (`data/index.db`).
- **proc_snapshot**: Capturează starea proceselor curente din `/proc` (PID, PPID, stare, nume, linie de comandă, RSS și CPU time în clock ticks) și le scrie într-o bază de date binară (`data/proc.db`).
- **Sincronizare Concurentă (SPMD)**: Multiple instanțe ale programelor pot rula în paralel, scriind în aceleași fișiere de baze de date. Sincronizarea este implementată direct pe fișier folosind lacăte exclusive și partajate prin apelul `fcntl(2)` (fără fișiere de lock externe).
- **db_diff**: Compară două snapshot-uri (vechi vs. nou) de același tip și generează rapoarte text detaliate (`reports/T3_filediff.txt` sau `reports/T3_procdiff.txt`) evidențiind intrările adăugate, șterse sau modificate semnificativ.

👉 *Pentru documentația detaliată a T3, accesați [README-ul din T3](./T3/README.md).*

---

### 📌 [Tema T4](./T4/) — Inventariere Multiproces în C (`fork`/`exec` & `mmap`)
- **Arhitectură Manager-Worker**: Un proces central (`fileops_manager`) coordonează `N` procese copii (`fileops_worker`) pornite prin `fork()` și `exec()`.
- **IPC prin Memorie Partajată**: Comunicarea dintre procese se realizează extrem de rapid în RAM printr-un fișier mapat cu `mmap(..., MAP_SHARED, ...)`, care conține:
  - Un header IPC cu configurări și stări globale.
  - O coadă circulară de joburi pentru directoarele ce urmează a fi scanate (suportă joburi dinamice adăugate de workeri).
  - Canale circulare de rezultate pentru file records.
  - Zona de statistici active per worker.
- **Sincronizare și Backpressure**: Sincronizarea resurselor partajate este asigurată de semafoare POSIX partajate între procese (`sem_t` în `mmap`). Se folosește o strategie de backpressure pentru a preveni pierderea de records sau suprascrierea bufferelor circulare.
- **Scriere Atomic•**: Managerul agregă toate rezultatele din memoria partajată și scrie atomic baza de date binară finală (`data/inventory.db`) prin tehnica temp file (`tmp/data_base_tmp.db`) urmată de `rename(2)`. Include moduri CLI `--verify` și `--dump`.

👉 *Pentru documentația detaliată a T4, accesați [README-ul din T4](./T4/README.md).*

---

### 📌 [Tema T5](./T5/) — Control Plane, Semnale și Gestiune Grațioasă (Shutdown)
- **Separarea Planurilor**: Separă complet *Data Plane* (job queue, rezultate în `mmap`) de *Control Plane* (comunicare prin canal pipe anonim unidirectional de la workeri la manager) și de *Signal Plane* (semnale de sistem).
- **Protocolul Pipe (`T5MSG`)**: Workerii transmit mesaje scurte, atomice și asincrone (de progres: `JOB_DONE`, de finalizare: `WORKER_EXITING`, sau erori: `ERROR`) către Manager prin pipe. Managerul citește asincron folosind mod non-blocant (`O_NONBLOCK`).
- **Gestiune Semnale (Signal Plane)**:
  - `SIGUSR1`: Managerul afișează în timp real o linie de status stabilă și agregată în consolă (`STATUS queued_jobs=... active_jobs=...`).
  - `SIGINT` / `SIGTERM`: Inițiază un shutdown grațios. Managerul oprește alocarea de joburi, notifică copiii, le acordă un timeout grațios (`--graceful-timeout`), iar ca ultim resort curăță procesele prin `SIGKILL`.
  - `SIGCHLD`: Managerul colectează asincron statusurile workerilor prin `waitpid()` pentru a preveni apariția proceselor zombie.
- **Semantica DB Incomplet**: Dacă inventarierea este întreruptă controlat de utilizator prin semnale, managerul asigură scrierea unei baze de date valide structural, dar marcată explicit în header cu flag-ul `complete=0`.

👉 *Pentru documentația detaliată a T5, accesați [README-ul din T5](./T5/README.md).*

---

## 🛠️ Compilare, Rulare și Testare

Orchestrarea build-ului, rulării și testelor se face unitar prin intermediul scriptului centralizator `./tools/fileops.sh`.

### 1. Inițializarea structurii și compilarea surselor
```bash
# Creează directoarele necesare
./tools/fileops.sh init

# Compilează codul sursă C cu opțiunile -Wall -Wextra -Werror -std=c11
./tools/fileops.sh build
```

### 2. Rularea Utilitarelor (Exemple)

**Mod SPMD Concurent (T3):**
```bash
# Pornirea indexării unui director (pot fi lansate multiple instanțe concurente pe același DB)
./tools/fileops.sh run -- fileops_indexer --root /cale/director --db data/index.db

# Capturarea unui snapshot de procese din /proc
./tools/fileops.sh run -- proc_snapshot --db data/proc.db

# Compararea a două snapshot-uri
./tools/fileops.sh run -- db_diff --old data/index_old.db --new data/index_new.db --out reports/T3_filediff.txt
```

**Mod Manager-Worker Multiproces (T4 & T5):**
```bash
# Pornirea managerului de inventar cu 4 workeri și timeout de 5 secunde
./tools/fileops.sh run -- fileops_manager --root /cale/director --workers 4 --ipc data/ipc.mmap --db data/inventory.db --graceful-timeout 5 --pid-file tmp/manager.pid
```

**Verificarea și Dump-ul Bazelor de Date:**
```bash
# Validează integritatea structurală a bazei de date
./tools/fileops.sh run -- fileops_manager --db data/inventory.db --verify

# Afișează metadatele bazei de date sub formă de cheie=valoare
./tools/fileops.sh run -- fileops_manager --db data/inventory.db --dump
```

### 3. Rularea Testelor Automate
Fiecare modul conține teste neinteractive menite să valideze scenariile complexe de execuție concurentă, sincronizare mmap și tratare a semnalelor.
```bash
./tools/fileops.sh test
```

---

## 📚 Documentație Tehnică Detaliată

Pentru detalii tehnice aprofundate la nivel de protocol și format binar, vă rugăm să consultați fișierele din directoarele de documentație:
- [T3/doc/Format_DB.md](./T3/doc/Format_DB.md) — Structura exactă a headerelor și recordurilor pentru bazele de date din T3 (`index.db` și `proc.db`).
- [T4/doc/MMAP_PROTOCOL.md](./T4/doc/MMAP_PROTOCOL.md) — Layout-ul detaliat al memoriei partajate, structura cozilor circulare de joburi și rezultate din T4.
- [T4/doc/T4_DB_FORMAT.md](./T4/doc/T4_DB_FORMAT.md) — Formatul binar al bazei de date finale `inventory.db` produse de manager.
- [T5/doc/T5_CONTROL_PLANE.md](./T5/doc/T5_CONTROL_PLANE.md) — Structura canalelor pipe de control plane, formatul mesajelor `T5MSG` și mecanismele asincrone de semnalizare.

---
*Proiect realizat în cadrul laboratorului de Sisteme de Operare, 2026.*