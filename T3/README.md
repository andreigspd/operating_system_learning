# Modulul T3 — Ghid Dezvoltator (High-Level Overview)

## 📌 Ce se întâmplă în acest modul?
Modulul **T3** implementează versiunea inițială a utilitarelor de scanare și monitorizare. Spre deosebire de versiunile ulterioare, **T3 folosește o arhitectură SPMD (Single Program, Multiple Data)**. Asta înseamnă că nu există un coordonator central (manager); în schimb, multiple instanțe independente ale aceluiași utilitar rulează în paralel și își coordonează accesul la fișierele de date direct prin blocaje la nivel de fișier (`fcntl`).

---

## 📁 Structura Codului Sursă (`src/`)

În subdirectorul `src/` se află logica în C a modulului:

1. **[main_fileops_indexer.c](./src/main_fileops_indexer.c)**:
   - Parcurge recursiv directoarele primite prin `--root`.
   - Pentru fiecare fișier, determină tipul, inodul, dimensiunea și timpul de modificare.
   - Calculează un checksum simplu prin citirea fișierelor binare în grupuri de 4 octeți și aplicarea operației XOR.
   - Actualizează baza de date `index.db` asigurându-se că nu se introduc dubluri. Sincronizarea cu alte instanțe paralele de indexer se face prin plasarea de lacăte exclusive (`F_WRLCK`) pe header sau pe recorduri individuale prin apelul `fcntl`.

2. **[main_proc_snapshot.c](./src/main_proc_snapshot.c)**:
   - Scanează directorul `/proc` pentru a detecta procesele active din sistem (subdirectoarele cu nume formate doar din cifre).
   - Pentru fiecare proces, citește datele din `/proc/[pid]/status` (PPID, nume, stare, RSS) și `/proc/[pid]/stat` (timpul CPU consumat în clock ticks).
   - Actualizează baza de date `proc.db` concurent sub lock-uri `fcntl`.

3. **[main_db_diff.c](./src/main_db_diff.c)**:
   - Utilitar administrativ care deschide două fișiere de baze de date de același tip (ambele index sau ambele procese).
   - Validează potrivirea versiunii și a markerului `magic`.
   - Compară înregistrările și identifică elementele adăugate, șterse sau modificate semnificativ (ex: dacă memoria RSS a unui proces a variat cu peste 1MB sau timpul CPU cu peste 100 ticks).

4. **[main_db_inspector.c](./src/main_db_inspector.c)**:
   - Un utilitar mic, de uz intern, care calculează dimensiunea teoretică pe care o bază de date ar trebui să o aibă pe disc. Este utilizat de scripturile de testare pentru a detecta eventuale coruperi sau suprapuneri defectuoase cauzate de sincronizarea concurentă.

---

## 🛠️ Compilare și Execuție rapidă

Modulul este orchestrat prin scriptul central din rădăcina proiectului:

```bash
# Compilare utilitare T3
./tools/fileops.sh build

# Pornire indexare recursivă
./tools/fileops.sh run -- fileops_indexer --root /cale/director --db data/index.db

# Generare instantaneu procese
./tools/fileops.sh run -- proc_snapshot --db data/proc.db

# Rularea diferenței între două snapshot-uri
./tools/fileops.sh run -- db_diff --old data/index_old.db --new data/index_new.db --out reports/T3_filediff.txt
```

---

## 📚 Unde se găsesc specificațiile tehnice detaliate?
Dări doriți să înțelegeți detaliile la nivel de octet și bit, structurile de date structurate din C, algoritmul exact de hashing sau regulile specifice de validare structurală, consultați documentul dedicat din folderul `doc`:
👉 **[Format_DB.md](./doc/Format_DB.md)**
