# Conway's Game of Life - Paralelna Simulacija Celularnog Automata

**Student:** Milica Radic SV26/2022
**Predmet:** Napredne tehnike programiranja  
**Tema:** Računarstvo visokih performansi - Celularni automati

---

## 0. Uvod u Celularne Automate

### 0.1. Šta su celularni automati?

Celularni automati su diskretni matematički modeli koji se sastoje od pravilne mreže ćelija, gde svaka ćelija može biti u jednom od konačnog broja stanja. Sistem evoluira kroz diskretne vremenske korake, pri čemu se novo stanje svake ćelije određuje na osnovu njenog trenutnog stanja i stanja susednih ćelija prema unapred definisanim pravilima.

Iako su pravila jednostavna i lokalna (svaka ćelija "gleda" samo svoje susede), globalno ponašanje sistema može biti izuzetno kompleksno i nepredvidivo. Ova karakteristika čini celularne automate fascinantnim predmetom istraživanja u matematici, fizici, biologiji i računarstvu.

### 0.2. Conway's Game of Life

Conway's Game of Life, kreiran 1970. godine od strane britanskog matematičara Johna Hortona Conwaya, predstavlja najpoznatiji primer celularnog automata. Radi se o "igri nula igrača" (engl. zero-player game), gde nakon postavljanja početne konfiguracije, sistem evoluira autonomno prema fiksnim pravilima.

**Struktura:**
- Beskonačna dvodimenzionalna ortogonalna mreža kvadratnih ćelija
- Svaka ćelija može biti u jednom od dva stanja: **živa** (alive) ili **mrtva** (dead)
- Svaka ćelija ima 8 suseda (horizontalno, vertikalno i dijagonalno)

**Pravila evolucije (B3/S23):**

1. **Smrt usled pothranjivanja:** Živa ćelija sa manje od 2 živa suseda umire
2. **Preživljavanje:** Živa ćelija sa 2 ili 3 živa suseda ostaje živa u sledećoj generaciji
3. **Smrt usled prenaseljenosti:** Živa ćelija sa više od 3 živa suseda umire
4. **Rođenje:** Mrtva ćelija sa tačno 3 živa suseda oživljava

---

## 1. Definicija Problema i HPC Tema

Ovaj projekat se bavi problemom simulacije evolucije Conway's Game of Life na velikim mrežama kroz veliki broj generacija, koristeći tehnike računarstva visokih performansi.

### 1.1. Cilj projekta

Glavni cilj je implementirati efikasnu sekvencijalnu i paralelnu verziju Game of Life simulacije u programskim jezicima Python i Rust, izvršiti eksperimente skaliranja i vizualizovati rezultate, sve u skladu sa zahtevima predmeta.

---

## 2. Arhitektura Rešenja

### 2.1. Implementacija u Pythonu (25 poena)

#### Sekvencijalna verzija

**Tehnologije:** Python 3.10+, NumPy

**Struktura programa:**
- Inicijalizacija mreže proizvoljnih dimenzija (N×M)
- Implementacija pravila Game of Life kroz prebrojavanje suseda
- Generisanje nove generacije primenom pravila B3/S23
- **Boundary conditions:** Periodični granični uslovi (toroidna topologija) - leva ivica se "povezuje" sa desnom, gornja sa donjom

**Izlazni format:**
- Folder: `output/python_seq/`
- Za svaku N-tu generaciju (npr. svaku 10. ili 100.) čuva se stanje mreže
- Format: CSV datoteka gde 1 = živa ćelija, 0 = mrtva ćelija

#### Paralelizovana verzija

**Tehnologije:** Python multiprocessing biblioteka

**Strategija paralelizacije - Domain Decomposition:**
- Mreža se deli na horizontalne blokove (strip decomposition)
- Svaki proces dobija kontinualni deo redova za obradu

**Ghost cells (Halo zone):**
- Problem: Ćelije na granici bloka trebaju susede iz drugih blokova
- Rešenje: Svaki proces čuva dodatni red iznad i ispod svog bloka
- Pre svake generacije procesi razmenjuju granične redove

**Sinhronizacija:**
- Koristi se `multiprocessing.Barrier` za sinhronizaciju nakon svake generacije
- Procesi moraju završiti sa računanjem svoje zone pre nego što razmene podatke

**Izlazni format:**
- Folder: `output/python_par/`
- Identičan format kao sekvencijalna verzija
- Glavni proces agregira rezultate i čuva u CSV

---

### 2.2. Implementacija u Rustu (26 poena)

#### Sekvencijalna verzija

**Tehnologije:** Rust 1.75+, Cargo, clap (CLI parsing), csv crate

**Optimizacije:**
- Koristi se `Vec<bool>` ili za bolju memorijsku efikasnost `Vec<u8>` sa bit-packingom
- Inline funkcije za kritične petlje
- Toroidna topologija sa modulo operacijom: `(x + width) % width`

**Izlazni format:**
- Folder: `output/rust_seq/`
- Format: Binarni (bincode) ili CSV za kompatibilnost

#### Paralelizovana verzija

**Tehnologije:** Rust rayon crate (data parallelism) ili std::thread

**Strategija - Rayon:**
```
Koristi se rayon::par_iter() za paralelizaciju:
- Mreža se automatski deli na chunks
- Svaki thread obrađuje svoj chunk
- Minimalna sinhronizacija (rayon interno upravlja)
- Par_iter omogućava funkcionalni pristup paralelizmu
```

**Konkretna implementacija:**
- Kreiranje thread pool-a sa N worker-a
- Svaki worker dobija slice od Vec<bool>
- Nakon obrade, rezultati se merge-uju u glavnu mrežu
- Barrier osigurava da svi thread-ovi završe generaciju pre prelaska na sledeću

**Izlazni format:**
- Folder: `output/rust_par/`
- Identičan format kao sekvencijalna Rust verzija

---

## 3. Eksperimenti Skaliranja (9 + 10 poena)

### 3.1. Amdalov Zakon (Jako Skaliranje)

Veličina problema je fiksna, i postavlja se pitanje koliko brzo se može uraditi posao. Vreme izvršavanja je ograničeno delom koda koji mora da se izvršava serijski.

**Definicija:** Fiksira se ukupan obim posla (N×N mreža), a povećava se broj procesorskih jezgara (P).

**Merenje:** Upoređuje se vreme izvršavanja T(1) sa T(P) (vreme izvršavanja na jednom jezgru i na P jezgara).

**Parametri eksperimenta:**
- Fiksna veličina mreže: N×N ćelija
- Broj generacija: T
- Random inicijalizacija sa zadatim procentom živih ćelija
- Broj jezgara: 1, 2, 4, 8, 16, ...
- Ponavljanja: ~30× po konfiguraciji

**Metrike:**
- Speedup: S(P) = T(1) / T(P)
- Efficiency: E(P) = S(P) / P
- Teorijski maksimum: S(P) = 1 / ((1-f) + f/P), gde je f paralelni deo koda

**Grafici:** Jako skaliranje u Python-u i Rust-u
- X-osa: broj jezgara
- Y-osa: speedup
- Linije: idealno skaliranje, teorijski maksimum, stvarno ubrzanje

---

### 3.2. Gustafsonov Zakon (Slabo Skaliranje)

Vreme izvršavanja je fiksno, suština je da za isto vreme uradimo što više posla. Veličina problema raste zajedno sa brojem jezgara.

**Definicija:** Fiksira se obim posla po jezgru (Nbase×Nbase ćelija), a istovremeno se povećava broj jezgara (P) i ukupan obim posla.

**Manipulacija Poslom:** Konstanta opterećenja po jezgru se postiže dinamičkim podešavanjem veličine mreže:
- width(P) = sqrt(P) × Nbase
- P=1: Nbase×Nbase
- P=2: sqrt(2)×Nbase × sqrt(2)×Nbase
- P=4: 2×Nbase × 2×Nbase
- P=k: sqrt(k)×Nbase × sqrt(k)×Nbase

**Parametri eksperimenta:**
- Bazna veličina po jezgru: Nbase×Nbase ćelija
- Broj generacija: T (konstantno)
- Random inicijalizacija sa zadatim procentom živih ćelija
- Ponavljanja: ~30× po konfiguraciji

**Metrike:**
- Scaled Speedup: S(P) = T(1) × P / T(P)
- Idealno: S(P) = P (vreme ostaje konstantno)

**Grafici:** Slabo skaliranje u Python-u i Rust-u
- X-osa: broj jezgara
- Y-osa: scaled speedup
- Idealna linija: y = x

---

## 4. Vizualizacija Rešenja (10 poena)

**Tehnologija:** Rust + Plotters biblioteka

**Tip 1: Statički prikaz pojedinačne generacije**
- Učitavanje CSV/binary datoteke
- Renderovanje mreže kao slike (crno-belo ili u boji)
- Žive ćelije: crne, mrtve ćelije: bele
- Rezolucija: svaka ćelija = 1 pixel za velike mreže

**Tip 2: Animacija evolucije**
- Učitavanje svih sačuvanih generacija
- Generisanje PNG frame-ova za svaku generaciju
- Kreiranje GIF animacije ili sekvence slika
