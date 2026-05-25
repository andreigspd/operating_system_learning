# Modulul T5 — Ghid Dezvoltator (High-Level Overview)

## 📌 Ce se întâmplă în acest modul?
Modulul **T5** completează infrastructura multiproces a inventariatorului prin adăugarea unui **Control Plane dedicat (bazat pe conducte anonime de tip pipe)** și a unui **Signal Plane (pentru gestiunea semnalelor de operare)**. Acest modul demonstrează separarea completă a fluxului de date primare (care continuă să circule eficient prin memoria partajată `mmap`) de mesajele scurte de control (progres, erori) și semnalele asincrone de sistem necesare raportării stării la cerere sau opririi grațioase a întregii aplicații.

---

## 📁 Structura Codului Sursă (`src/`)

În folderul `src/` se află implementarea actualizată cu control plane și semnale:

1. **[main_fileops_manager.c](./src/main_fileops_manager.c)**:
   - Inițializează canalul de control de tip pipe anonim unidirectional înainte de pornirea workerilor.
   - Trece capătul de citire al pipe-ului pe modul non-blocant (`O_NONBLOCK` via `fcntl`) pentru a asigura scanarea asincronă a mesajelor text sosite de la copii fără a introduce întârzieri sau blocaje.
   - Implementează handlere asincrone sigure (bazate pe flag-uri de tip `volatile sig_atomic_t`) pentru:
     - `SIGUSR1`: generează o linie standardizată de `STATUS` în timp real în consolă cu progresul agregat.
     - `SIGCHLD`: curăță instantaneu copiii opriți sau prăbușiți (`waitpid` neblocant cu `WNOHANG`) prevenind procesele zombie.
     - `SIGINT` / `SIGTERM`: activează algoritmul de *Graceful Shutdown* care oprește depunerea de noi sarcini, transmite semnal de oprire workerilor, așteaptă o perioadă specificată de timeout (`--graceful-timeout`) și, ca ultim resort, elimină copiii blocați prin `SIGKILL`.
   - Asigură scrierea atomică a unei baze de date structural valide, dar marcată explicit cu flag-ul `complete=0` dacă inventarierea a fost oprită forțat înainte de finalizarea naturală.

2. **[main_fileops_worker.c](./src/main_fileops_worker.c)**:
   - Prinde descriptorul de scriere al pipe-ului de control transmis ca argument CLI (`--control-fd`).
   - Înregistrează un handler de semnal pentru `SIGTERM`. În timpul parcurgerii directoarelor sau a așteptării pe semafoare, verifică flag-ul de oprire grațioasă.
   - Trimite mesaje scurte, atomice și formatate asincron către manager sub forma protocolului `T5MSG` (mesaje `JOB_DONE` la fiecare director terminat, `WORKER_EXITING` la ieșire cu motivele `normal` sau `shutdown`, sau `ERROR` cu codurile `errno` ale apelurilor eșuate).
   - La oprire, își actualizează corect timpii de procesor în structura partajată și iese curat.

---

## 🛠️ Compilare și Execuție rapidă

Modulul este compilat și poate fi testat în siguranță cu semnale prin scriptul dedicat:

```bash
# Compilare utilitare T5
./tools/fileops.sh build

# Pornirea inventarierii cu timeout grațios de 10 secunde și salvarea PID-ului managerului
./tools/fileops.sh run -- fileops_manager --root /cale/director --workers 4 --ipc data/ipc.mmap --db data/inventory.db --graceful-timeout 10 --pid-file tmp/manager.pid

# Trimiterea unui semnal pentru obținerea statusului în timp real
# (PID-ul se citește din fișierul generat mai sus)
kill -USR1 $(cat tmp/manager.pid)

# Trimiterea unui semnal de oprire pentru declanșarea shutdown-ului grațios
kill -TERM $(cat tmp/manager.pid)
```

---

## 📚 Unde se găsesc specificațiile tehnice detaliate?
Specificațiile complete legate de protocolul de comunicare prin pipe, formatul exact al mesajelor text `T5MSG`, comportamentul diagramelor de tranziție a semnalelor în manager și workeri sau semantica bazei de date cu `complete=0` se găsesc în documentul:
👉 **[T5_CONTROL_PLANE.md](./doc/T5_CONTROL_PLANE.md)**
