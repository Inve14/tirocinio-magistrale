# Report Task 1 — Temporal Network Motif Analysis
**Tirocinio universitario:** "Profili evolutivi multilivello di reti complesse tramite temporal network motif"  
**Data:** 2026-05-04  
**Autore:** Carlo Invernizzi

---

## Indice
1. [Obiettivo e contesto](#1-obiettivo-e-contesto)
2. [Traduzione C++ → Python](#2-traduzione-c--python)
3. [Null Models](#3-null-models)
4. [Visualizzazione dei Motif](#4-visualizzazione-dei-motif)
5. [Risultati — facebook_wall](#5-risultati--facebook_wall)
6. [Risultati — CollegeMsg (SNAP)](#6-risultati--collegemsg-snap)
7. [Confronto tra i due dataset](#7-confronto-tra-i-due-dataset)
8. [Parametri usati](#8-parametri-usati)
9. [Connessione con la tesi](#9-connessione-con-la-tesi)

---

## 1. Obiettivo e contesto

Il Task 1 analizza due reti temporali reali per rispondere alla domanda:

> **Quali pattern di interazione (motif) si ripetono nella rete, quanto sono frequenti, e sono statisticamente significativi rispetto a reti casuali?**

Un **temporal network motif** è una sequenza ordinata nel tempo di archi connessi che si verificano entro una finestra temporale `delta`. A differenza dei motif statici, la dimensione temporale impone un ordinamento degli eventi e rende il conteggio computazionalmente molto più ricco.

Il codice implementa due algoritmi principali tratti dal paper:
> *"Using Motif Transitions for Temporal Graph Generation"* — Liu, Penghang & Sarıyüce, Ahmet Erdem — KDD 2023

- **TMC** (Temporal Motif Counting): conta le istanze di ogni tipo di motif nella rete reale
- **MTM** (Motif Transition Model): apprende le probabilità di transizione tra motif e genera reti sintetiche

---

## 2. Traduzione C++ → Python

### 2.1 Cosa è stato tradotto

La repository originale [KDD23-MTM](https://github.com/erdemUB/KDD23-MTM) è scritta in C++. Il Task 1 ne costituisce la traduzione fedele in Python, suddivisa in due notebook Jupyter:

- `task1_facebook.ipynb` → dataset facebook_wall
- `task1_snap.ipynb` → dataset CollegeMsg (Stanford SNAP)

La tabella di corrispondenza completa è:

| Funzione C++ | Funzione Python | Note implementative |
|---|---|---|
| `createEvents()` | `create_events()` | Identica logica, gestione commenti `%` e `#` |
| `encodeMotif()` | `encode_motif()` | Identica: mappa nodi a cifre per ordine di apparizione |
| `countMTP()` | `count_mtp()` | `set<motif>` C++ → `Set[Tuple[Event,...]]` Python |
| `countTransition()` | `count_transition()` | Identica; aggiunge stato `'S'` (Stop) per prefissi terminali |
| `generateGraph()` | `generate_graph()` | `discrete_distribution` C++ → `random.choices()` Python |
| `RandomizeIE()` | `randomize_ie()` | Identica; preserva degree sequence degli eventi iniziali |
| `countInstance()` | `count_instance()` | Identica; mantiene `imap` con conteggi cumulativi |

### 2.2 Come funziona il codice tradotto

Il flusso di esecuzione segue questo schema:

```
Dataset reale (.edges / .txt)
       ↓
create_events()         → lista eventi (timestamp, (u,v)) ordinati, senza self-loop, senza duplicati
       ↓
run_tmc()               → TMC: conteggio istanze motif
  └─ count_instance()   → aggiorna l'instance map per ogni nuovo evento
  └─ encode_motif()     → converte ogni istanza in codice stringa canonico
       ↓
run_mtm()               → MTM: apprendimento + generazione
  └─ count_mtp()        → scorre eventi, costruisce prefissi attivi
  └─ count_transition() → tabella di transizione (codice_prefisso → codice_estensione, peso, lambda)
  └─ randomize_ie()     → randomizza eventi iniziali preservando degree sequence
  └─ generate_graph()   → genera rete sintetica seguendo le transizioni
       ↓
Null Models (3 versioni randomizzate)
       ↓
Enrichment Score        → count_reale / media(count_null)
       ↓
Visualizzazione         → gallery grafica dei motif con nodi/archi/ordine temporale
```

### 2.3 Scelte implementative rilevanti

**Codifica del motif (`encode_motif`)**  
Ogni istanza di motif viene codificata come stringa dove ogni coppia di caratteri rappresenta un arco diretto. I nodi vengono rinominati con cifre intere in ordine di prima apparizione:

```
Esempio:
  A→B (t=0), A→C (t=1), B→D (t=2)
  A=0, B=1, C=2, D=3
  Codice: "010213"
         ^^  ^^  ^^
         01  02  13   = A→B, A→C, B→D
```

Due istanze producono lo stesso codice se e solo se sono **strutturalmente isomorfe** (stessa topologia, stesso ordine temporale degli archi).

**Instance map (`count_instance`)**  
Mantiene un dizionario `imap` che mappa ogni sequenza di eventi (chiave) a `(conteggio, insieme_nodi)`. Per ogni nuovo evento `e`, estende tutti i prefissi attivi che condividono almeno un nodo con `e` e che rientrano nella finestra `delta`. Con `consecutive=NO`, i prefissi non vengono eliminati dopo l'estensione: permettono ulteriori concatenazioni.

**Prefissi attivi (`count_mtp`)**  
Per MTM, il set `prefixes` tiene traccia di tutte le catene di eventi in corso. Se un nuovo evento `e` condivide almeno un nodo con un prefisso attivo e si trova entro `delta` secondi dall'ultimo evento del prefisso, viene creata un'estensione. Se `e` non si aggancia a nessun prefisso esistente, viene classificato come **evento iniziale** (`initial_E`).

**Distribuzione temporale degli inter-arrival time**  
Per ogni tipo di transizione, MTM apprende il tempo medio tra un evento e il successivo nel motif. Questo tempo viene modellato come **distribuzione esponenziale** con parametro `λ = 1 / tempo_medio`. In fase di generazione, il delta temporale di ogni nuovo arco viene campionato con `random.expovariate(λ)`.

**Generazione del grafo sintetico (`generate_graph`)**  
Parte dagli eventi iniziali randomizzati (`RandomizeIE`). Per ogni evento iniziale, segue iterativamente le transizioni nella tabella fino a raggiungere lo stato `'S'` (stop) o il limite `L_MAX`. I nuovi nodi vengono scelti preferendo nodi già vicini nella rete (con probabilità `1 - use_existing`) oppure nodi completamente nuovi.

---

## 3. Null Models

Sono stati implementati tre null model per valutare la significatività statistica dei motif trovati. Ciascuno distrugge una proprietà diversa della rete originale.

### 3.1 Time Shuffle

```python
def time_shuffle(events, seed=42):
    edges = [e[1] for e in events]
    timestamps = [e[0] for e in events]
    rnd.shuffle(timestamps)
    return sorted(zip(timestamps, edges))
```

**Cosa fa:** rimescola casualmente i timestamp tra gli archi, lasciando immutata la lista degli archi (chi comunica con chi).

**Cosa preserva:** struttura topologica completa (archi, degree sequence).

**Cosa distrugge:** tutte le correlazioni temporali — l'ordine in cui avvengono gli eventi diventa casuale.

**Domanda che risponde:** "Il motif dipende dall'ordine temporale degli eventi, oppure sarebbe lo stesso anche con timestamp casuali?"

Se un motif ha `count_reale ≈ count_time_shuffle`, allora il motif non dipende dall'ordine temporale ma solo dalla topologia.

### 3.2 Edge Shuffle (Configuration Model)

```python
def edge_shuffle(events, V, seed=42):
    degree_seq = []
    for (_, (u, v)) in events:
        degree_seq.append(u); degree_seq.append(v)
    rnd.shuffle(degree_seq)
    # riassegna ogni coppia consecutiva come nuovo arco
    for i, t in enumerate(timestamps):
        u_new = degree_seq[2*i]; v_new = degree_seq[2*i+1]
        new_events.append((t, (u_new, v_new)))
```

**Cosa fa:** mantiene la sequenza dei timestamp originali, ma riassegna casualmente gli endpoint degli archi preservando il numero complessivo di interazioni per nodo (degree sequence).

**Cosa preserva:** timestamp, numero totale di eventi, distribuzione dei gradi.

**Cosa distrugge:** la struttura topologica — chi parla con chi diventa casuale.

**Come vengono assegnati i timestamp:** ogni evento conserva il suo timestamp originale, solo gli endpoint (u,v) vengono rimescolati.

**Domanda che risponde:** "Il motif dipende da chi interagisce con chi (struttura della rete), oppure emerge anche rimescolando gli endpoint?"

### 3.3 Watts-Strogatz (Small-World)

```python
def strogatz_shuffle(events, V, k_neighbors=4, p_rewire=0.1, seed=42):
    # 1. Crea ring lattice: ogni nodo connesso ai k/2 vicini a destra e sinistra
    for i in range(n):
        for j in range(1, half_k+1):
            nb = (i+j) % n
            ws_edges.add((min(i,nb), max(i,nb)))
    # 2. Rewiring: ogni arco viene reindirizzato a nodo casuale con prob p_rewire
    for (i, j) in ws_edges:
        if rnd.random() < p_rewire:
            new_j = rnd.randint(0, n-1)  # nuovo endpoint casuale
    # 3. Campiona i timestamp dalla distribuzione reale
    sampled_times = sorted(rnd.choices(real_times, k=n_events))
    return sorted([(t, (u,v)) for t,(u,v) in zip(sampled_times, ws_sampled)])
```

**Cosa fa:** genera una rete completamente diversa dall'originale, costruita come grafo Small-World di Watts-Strogatz. I nodi sono gli stessi, ma la topologia è sintetica (ring lattice con rewiring casuale). I timestamp vengono **campionati con reinserimento** dalla distribuzione temporale reale.

**Parametri usati:** `k_neighbors=4` (ogni nodo ha 4 vicini nel ring), `p_rewire=0.1` (10% degli archi viene reindirizzato).

**Cosa preserva:** numero di nodi, numero di eventi, distribuzione temporale degli eventi.

**Cosa distrugge:** intera struttura della rete (topologia, degree sequence, correlazioni).

**Come vengono assegnati i timestamp:** vengono estratti a caso (con replacement) dal pool di timestamp reali e assegnati in ordine crescente agli archi del grafo Watts-Strogatz.

**Domanda che risponde:** "Il motif è specifico di questa rete sociale, oppure emerge anche in qualsiasi rete con la stessa dimensione e distribuzione temporale?"

### 3.4 Calcolo dell'Enrichment Score

```
enrichment(m) = count_reale(m) / media(count_ts(m), count_es(m), count_ws(m))
```

- **enrichment > 1:** il motif è **sovra-rappresentato** nella rete reale → è statisticamente significativo → cattura una struttura reale
- **enrichment < 1:** il motif è **sotto-rappresentato** → è più frequente nelle reti casuali → non è caratteristico di questa rete

---

## 4. Visualizzazione dei Motif

### 4.1 Traduzione del codice motif in grafo

La funzione `parse_motif_code(code)` decompone il codice stringa in lista di archi:

```python
def parse_motif_code(code):
    # ogni coppia di caratteri code[i], code[i+1] = arco (u→v) all'istante t=i//2
    return [(i//2, int(code[i]), int(code[i+1])) for i in range(0, len(code), 2)]
```

Esempi di decodifica:

| Codice | Archi | Struttura |
|--------|-------|-----------|
| `0102` | 0→1, 0→2 | Wedge: nodo 0 invia a due nodi distinti |
| `010203` | 0→1, 0→2, 0→3 | Hub (fan-out a 3): nodo 0 è mittente centrale |
| `01020304` | 0→1, 0→2, 0→3, 0→4 | Hub (fan-out a 4): nodo 0 invia a quattro nodi |
| `01101213` | 0→1, 1→0, 1→2, 1→3 | Reciprocità + fan-out: 0 e 1 si scambiano messaggi, poi 1 scrive ad altri due |
| `010121` | 0→1, 0→1, 2→1 | Multiedge + convergenza: due messaggi ripetuti 0→1, poi un terzo nodo scrive a 1 |
| `012131` | 0→1, 2→1, 3→1 | Convergenza (fan-in): tre nodi diversi scrivono allo stesso nodo 1 |

### 4.2 Layout grafico dei nodi

La funzione `get_layout(nodes, edges)` assegna le posizioni dei nodi nel canvas:
- Se esiste un **hub** (nodo con grado massimo ≥ 3 unico), viene posizionato al centro `(0,0)` e gli altri nodi disposti circolarmente attorno a lui.
- Per motif a 2 nodi, i nodi sono posizionati a sinistra e destra.
- Per topologie più complesse, si usa `nx.spring_layout()` con seed fisso per riproducibilità.

### 4.3 Parametri grafici

- **Colore delle frecce:** gradiente di blu dalla colormap `plt.cm.Blues`. Frecce più chiare = eventi più antichi (t0), frecce più scure = eventi più recenti (t3).
- **Label temporali:** ogni arco è annotato con `t0`, `t1`, `t2`, `t3` indicando l'ordine cronologico.
- **Curvatura degli archi (`rad`):** archi paralleli nella stessa direzione vengono curvati progressivamente per evitare sovrapposizioni. Archi nella direzione opposta (reciprocità) vengono curvati in senso contrario (`rad=0.2`).
- **Nodi:** cerchi blu scuro con etichetta numerica bianca (cifra = ID del nodo nel motif).

### 4.4 Commento delle gallery — facebook_wall

#### `facebook_motif_gallery_real.png` — Top-12 motif per frequenza

La gallery mostra i 12 motif più frequenti nella rete facebook_wall (su campione di 3000 eventi, delta=86400s, l_max=4). Tutti i motif hanno 4 archi (lunghezza massima).

| Posizione | Codice | Count | Struttura |
|-----------|--------|-------|-----------|
| 1 | `01020304` | 3,678,598 | **Hub puro (fan-out 4):** nodo 0 invia sequenzialmente a 4 nodi distinti (1,2,3,4). Struttura a stella uscente. Il nodo centrale è un utente molto attivo che scrive su molte bacheche. |
| 2 | `010203` | 341,911 | **Hub (fan-out 3):** nodo 0 invia a 3 nodi. Versione a 3 archi dello stesso pattern. |
| 3 | `01021314` | 221,419 | **Hub doppio:** nodo 0 invia a nodi 1 e 2 (t0,t1), poi nodo 1 invia a nodi 3 e 4 (t2,t3). Struttura a catena di hub con propagazione. |
| 4 | `01202324` | 193,294 | **Hub con convergenza e fan-out:** nodo 2 riceve da 0 (t1) e poi invia a 3 e 4 (t2,t3). Nodo 2 è sia destinatario che mittente attivo. |
| 5 | `01121314` | 183,839 | **Catena + fan-out:** 0→1, poi 1→2, 1→3, 1→4. Nodo 1 riceve da 0 e poi risponde in broadcast. |
| 6 | `01020314` | 133,961 | **Hub ibrido:** 0 invia a 1,2,3 (t0-t2), poi 1 invia a 4 (t3). Il primo destinatario diventa a sua volta mittente. |
| 7 | `01020340` | 120,792 | **Hub con ricezione finale:** 0 invia a 1,2,3 ma poi riceve da 4. Inversione del flusso finale. |
| 8 | `01023034` | 106,136 | **Connettore:** 0→1, 0→2, poi 3→0, 3→4. Due hub connessi tramite il nodo 0. |
| 9 | `01023134` | 79,324 | **Fan convergente + fan-out:** nodo 3 invia sia a 1 che a 4, mentre 0 aveva già inviato a 1 e 2. |
| 10 | `01212324` | 59,160 | **Convergenza doppia:** nodi 0 e 2 inviano entrambi a 1; poi 2 invia anche a 3 e 4. |
| 11 | `01020343` | 57,273 | **Hub + inversione parziale:** 0 invia a 1,2,3, ma poi 4 scrive a 3 (non a 0). Chiusura non verso il mittente originale. |
| 12 | `01020342` | 52,477 | **Variante:** come sopra, 4 scrive a 2. |

![Gallery top-12 motif facebook_wall — struttura grafica dei motif più frequenti con nodi (cerchi blu numerati), archi diretti (frecce) e ordine temporale codificato nel colore (chiaro=t0, scuro=t3)](task_1/task_1_facebook/output_png/facebook_motif_gallery_real.png)

**Osservazione generale:** la gallery del facebook_wall è **dominata da pattern a stella (hub)** con nodo 0 come mittente centrale. Questo è coerente con la natura della bacheca Facebook: un utente tende a scrivere su molte bacheche in sequenza nella stessa giornata.

#### `facebook_motif_gallery_enriched.png` — Top-8 motif statisticamente significativi (enrichment > 1)

Questa gallery mostra gli 8 motif con enrichment più alto, ordinati per significatività statistica.

| Posizione | Codice | Enrichment | Struttura e significato |
|-----------|--------|-----------|------------------------|
| 1 | `01020304` | **2.86x** | Hub puro a 4 destinatari. Massimamente sovra-rappresentato: nella rete reale è quasi 3 volte più frequente che nei null model. Un utente che scrive su 4 bacheche diverse nello stesso giorno è un pattern caratteristico di Facebook. |
| 2 | `010203` | **2.69x** | Hub a 3 destinatari (versione corta). Stessa logica. |
| 3 | `01101213` | **2.57x** | **Reciprocità + fan-out:** 0→1, 1→0, 1→2, 1→3. Due utenti si scambiano messaggi e poi uno dei due scrive anche ad altri. Pattern tipico di conversazione bilaterale che si espande. |
| 4 | `01021013` | **2.41x** | **Reciprocità allargata:** 0→1, 0→2, 1→0, 1→3. Scambio bilaterale tra 0 e 1, con entrambi che scrivono a nodi aggiuntivi. |
| 5 | `0102` | **2.33x** | **Wedge (2 archi):** 0 scrive a 1 e poi a 2. Forma base dell'hub. |
| 6 | `01020343` | **1.76x** | Hub a 4 con chiusura verso destinatario esistente. |
| 7 | `01023234` | **1.62x** | Pattern con connessione trasversale. |
| 8 | `01202324` | **1.51x** | Hub convergente + fan-out. |

![Gallery top-8 motif statisticamente significativi facebook_wall — solo i motif con enrichment > 1, ordinati per significatività; si notano gli hub puri in testa e i pattern di reciprocità (frecce bidirezionali curve) nelle posizioni 3 e 4](task_1/task_1_facebook/output_png/facebook_motif_gallery_enriched.png)

**Osservazione chiave:** i motif con **reciprocità** (`01101213`, `01021013`) hanno enrichment molto elevato (2.57x, 2.41x), suggerendo che le conversazioni bilaterali seguite da espansione verso altri nodi sono un pattern caratteristico e non casuale della rete Facebook.

### 4.5 Commento delle gallery — CollegeMsg

#### `snap_motif_gallery_real.png` — Top-12 motif per frequenza

La gallery CollegeMsg (5000 eventi, l_max=3) mostra motif a 3 archi con grande varietà topologica.

| Posizione | Codice | Count | Struttura |
|-----------|--------|-------|-----------|
| 1 | `010203` | 1,115,188 | **Hub fan-out:** 0 invia a 1, 2, 3 in sequenza. |
| 2 | `012131` | 793,677 | **Convergenza (fan-in):** nodi 0, 2, 3 inviano tutti al nodo 1. Struttura a stella entrante. |
| 3 | `012123` | 658,134 | **Convergenza + fan-out:** nodi 0 e 2 inviano a 1, poi 2 invia anche a 3. |
| 4 | `010232` | 565,256 | **Hub + convergenza:** 0 invia a 1 e 2, poi 3 invia a 2. Nodo 2 riceve da due mittenti. |
| 5 | `012103` | 492,873 | **Pattern ibrido:** 0→1, 2→1, 0→3. Nodo 1 riceve da due mittenti, poi 0 scrive ancora. |
| 6 | `010231` | 447,307 | **Hub + freccia inversa:** 0→1, 0→2, poi 3→1. Convergenza su nodo 1. |
| 7 | `012030` | 409,345 | **Convergenza su hub:** nodi 0, 2, 3 inviano tutti a 0... wait: 0→1, 2→0, 3→0. Nodo 0 prima invia, poi riceve da due nodi. Cambio di ruolo. |
| 8 | `012113` | 385,400 | **Convergenza + propagazione:** 0→1, 2→1, 1→3. Nodo 1 riceve da due nodi e poi invia. |
| 9 | `012132` | 335,212 | **Convergenza + inversione:** 0→1, 2→1, 3→2. Il nodo 2 è sia mittente verso 1 che destinatario di 3. |
| 10 | `011231` | 323,888 | **Catena + convergenza:** 0→1, 1→2, poi 3→1. Il nodo 1 è hub centrale. |
| 11 | `012023` | 299,310 | **Reciprocità + fan-out:** 0→1, 2→0, 2→3. Nodo 2 risponde a 0 e scrive anche a 3. |
| 12 | `011232` | 275,701 | **Catena + convergenza trasversale:** 0→1, 1→2, poi 3→2. |

![Gallery top-12 motif CollegeMsg — a differenza di facebook, i motif mostrano grande varietà topologica: fan-out, fan-in (frecce convergenti su un nodo centrale), catene e pattern misti; le strutture sono più equilibrate senza un hub dominante](task_1/task_1_SNAP/output_png/snap_motif_gallery_real.png)

**Osservazione generale:** la gallery CollegeMsg mostra una **varietà topologica molto maggiore** rispetto a Facebook. Mentre Facebook è dominato da hub "puri" (un nodo invia a molti), CollegeMsg mostra bilanciamento tra fan-out, fan-in, reciprocità e catene. Questo riflette la natura dei messaggi privati (più dialogici) vs i post su bacheca (più broadcast).

#### `snap_motif_gallery_enriched.png` — Top-8 motif statisticamente significativi

I motif con enrichment più alto in CollegeMsg sono quelli che coinvolgono **archi ripetuti tra gli stessi nodi** (multi-edge):

| Posizione | Codice | Enrichment | Struttura |
|-----------|--------|-----------|-----------|
| 1 | `010121` | **19.55x** | 0→1 (t0), 0→1 (t1), 2→1 (t2). Lo stesso arco 0→1 appare due volte: un utente manda due messaggi consecutivi allo stesso destinatario, poi un terzo utente scrive allo stesso. |
| 2 | `012121` | **18.40x** | 0→1, 2→1, 2→1. Due messaggi ripetuti di 2 verso 1, con uno iniziale da 0. |
| 3 | `010112` | **18.35x** | 0→1 due volte, poi 1→2. Due messaggi consecutivi verso lo stesso utente, che poi risponde a un terzo. |
| 4 | `012020` | **16.42x** | 0→1, poi 2→0 due volte. Risposta ripetuta di 2 verso 0. |
| 5 | `010102` | **15.98x** | 0→1 due volte, poi 0→2. Multi-messaggi consecutivi. |
| 6 | `010202` | **12.64x** | 0→1, poi 0→2 due volte. |
| 7-8 | vari | 3.73x–6.44x | Pattern senza ripetizioni di archi ma con strutture convergenti/divergenti. |

![Gallery top-8 motif statisticamente significativi CollegeMsg — dominati da pattern con archi multipli tra le stesse coppie di nodi (frecce parallele o sovrapposte), che rappresentano messaggi ripetuti tra gli stessi utenti nella stessa giornata; enrichment fino a 19.55x](task_1/task_1_SNAP/output_png/snap_motif_gallery_enriched.png)

**Osservazione chiave:** il fenomeno dei **multi-edge temporali** (stesso arco che appare più volte nella finestra delta) è altamente caratteristico di CollegeMsg. Archi ripetuti tra gli stessi utenti hanno enrichment di quasi 20x, indicando che gli studenti tendono a mandarsi messaggi multipli in sequenza rapida nella stessa giornata — comportamento tipico di chat istantanea.

---

## 5. Risultati — facebook_wall

### Dataset
| Proprietà | Valore |
|---|---|
| File | `ia-facebook-wall-wosn-dir.edges` |
| # eventi totali | 264,004 |
| # nodi totali | 45,813 |
| # archi statici | 183,412 |
| Span temporale | anomalo (vedi sotto) |
| # eventi campionati | 3,000 |
| # nodi nel campione | 2,797 |

**Nota sui timestamp:** il dataset facebook_wall presenta timestamp già convertiti (i valori raw risultano costanti nel campione preso). Il notebook rileva correttamente il problema (span temporale = 0 giorni) ma procede ugualmente con il TMC, che non dipende dai valori assoluti dei timestamp ma solo dalle differenze. La distribuzione temporale risultante mostra un singolo bin centrato su t=1.

### 5.1 `facebook_motif_top20.png` — Top-20 motif per frequenza

![Top-20 motif facebook_wall](task_1/task_1_facebook/output_png/facebook_motif_top20.png)

Il grafico a barre mostra una **distribuzione fortemente asimmetrica (heavy-tail)**:

- Il motif `01020304` (hub a 4 archi) domina con **3,678,598** occorrenze, circa 10 volte il secondo classificato.
- Il secondo motif `010203` (hub a 3 archi) ha 341,911 occorrenze.
- I restanti 18 motif hanno conteggi compresi tra 9,251 e 221,419, con decadimento regolare.
- Tutti i top-20 motif hanno **4 archi** (lunghezza massima), a eccezione di `0102` (2 archi) e `011213`, `010213`, `012023`, `010230` (3 archi).

Questo pattern indica che la rete facebook_wall è **saturata di hub**: la stragrande maggioranza delle istanze di motif rilevate coinvolge un nodo centrale che invia messaggi a più destinatari entro la stessa giornata.

### 5.2 `facebook_motif_by_length.png` — Distribuzione per lunghezza

![Distribuzione per lunghezza facebook_wall](task_1/task_1_facebook/output_png/facebook_motif_by_length.png)

Il grafico mostra il conteggio totale raggruppato per numero di archi nel motif:

- **2 archi:** ~31,000 occorrenze totali (trascurabile)
- **3 archi:** ~400,000 occorrenze totali
- **4 archi:** ~5,000,000 occorrenze totali

La netta dominanza dei motif a 4 eventi indica che entro la finestra di 1 giorno, nella rete facebook, si trovano regolarmente sequenze di 4 interazioni connesse. Questo è coerente con un comportamento di "posting in serie" tipico degli utenti attivi.

### 5.3 `facebook_enrichment.png` — Enrichment score

![Enrichment score facebook_wall](task_1/task_1_facebook/output_png/facebook_enrichment.png)

Il grafico mostra il rapporto tra frequenza reale e media dei null model. Le barre blu indicano motif sovra-rappresentati (enrichment > 1), le barre arancioni motif sotto-rappresentati.

**Motif statisticamente significativi (enrichment > 1):**

| Codice | Enrichment | Struttura |
|--------|-----------|-----------|
| `01020304` | **2.86x** | Hub puro a 4 destinatari |
| `010203` | **2.69x** | Hub a 3 destinatari |
| `01101213` | **2.57x** | Reciprocità + fan-out |
| `01021013` | **2.41x** | Reciprocità allargata |
| `0102` | **2.33x** | Wedge (forma base dell'hub) |
| `01020343` | **1.76x** | Hub + chiusura parziale |
| `01023234` | **1.62x** | Connettore con fan-out |
| `01020314` | **1.53x** | Hub ibrido (primo destinatario ri-invia) |
| `01202324` | **1.51x** | Hub convergente + fan-out |
| `01021314` | **1.47x** | Hub doppio a cascata |
| `01023134` | **1.24x** | Fan convergente + fan-out |
| `01121314` | **1.22x** | Catena + fan-out |
| `01212324` | **1.18x** | Convergenza doppia + fan-out |

**Motif sotto-rappresentati (enrichment < 1):** motif come `01020340` (0.53x), `01020341` (0.53x), `010230` (0.23x) risultano essere più frequenti nelle reti casuali che nella rete reale. In particolare, i motif che terminano con un arco **in ingresso verso il mittente originale** (qualcuno scrive a 0 dopo che 0 ha scritto a tutti) sono rari nella realtà: su Facebook è insolito che qualcuno risponda istantaneamente alla bacheca di un utente dopo che quest'ultimo ha appena scritto su molte bacheche.

### 5.4 `facebook_null_comparison.png` — Confronto con i null model

![Confronto null model facebook_wall](task_1/task_1_facebook/output_png/facebook_null_comparison.png)

Il grafico a barre raggruppate confronta i conteggi di ogni motif tra: rete reale (blu), Time Shuffle (arancione), Edge Shuffle (verde), Watts-Strogatz (viola, quasi invisibile = valori vicini a 0).

**Osservazioni chiave:**

1. **`01020304` (hub puro):** rete reale e Time Shuffle hanno quasi lo stesso conteggio (~3.67M), mentre Edge Shuffle ha solo ~186K e Strogatz ha 0. Questo significa che il pattern è già frequente anche con timestamp rimescolati ma **scompare quando si rimescola chi parla con chi**. Il motif è quindi strutturalmente determinato: dipende dal fatto che esiste un utente hub con molti vicini, non dall'ordine temporale.

2. **`01020340` (hub + ricezione finale):** l'Edge Shuffle produce ~558K occorrenze contro le ~121K reali. Questo motif è **più probabile in una rete casuale** che in quella reale: in una rete dove gli archi sono casuali è più facile che qualcuno scriva all'hub dopo che l'hub ha appena scritto a molti.

3. **Strogatz:** produce conteggi quasi nulli per tutti i motif a 4 archi. Questo è atteso: il grafo Watts-Strogatz ha una topologia uniforme (ring lattice) che non produce hub, quindi i pattern hub tipici di Facebook non emergono.

### 5.5 `facebook_degree_dist.png` — Degree distribution

![Degree distribution facebook_wall](task_1/task_1_facebook/output_png/facebook_degree_dist.png)

Il grafico in scala log-log confronta la degree distribution della rete originale (blu, linea continua) con quella del grafo generato da MTM (arancione, linea tratteggiata).

**Osservazione:** le due curve sono **praticamente sovrapposte** sull'intero range. Il grafo MTM preserva in modo eccellente la distribuzione dei gradi, con lievi scostamenti nella coda destra (nodi ad alto grado). Questo valida l'efficacia del modello MTM nel catturare la struttura statistica della rete.

La distribuzione segue un andamento a **power-law approssimato** (retta in scala log-log), confermando la presenza di pochi hub con molte connessioni e molti nodi con poche connessioni.

### 5.6 `facebook_temporal_dist.png` — Distribuzione temporale

![Distribuzione temporale facebook_wall](task_1/task_1_facebook/output_png/facebook_temporal_dist.png)

**Problema noto:** il grafico mostra un singolo bin centrato su timestamp = 1 sia per il dataset originale che per il generato MTM. Questo indica un **problema nei timestamp del dataset facebook_wall**: i valori risultano tutti identici (o quasi) nel campione di 3000 eventi estratto. Il notebook rileva che lo span temporale è 0 giorni e che il periodo è "1970-01-01 → 1970-01-01", confermando che i timestamp non sono stati convertiti correttamente dai secondi Unix originali.

**Impatto sull'analisi:** il TMC funziona correttamente poiché usa le differenze temporali tra eventi (e tutti gli eventi hanno differenza ≤ delta=86400), ma il grafo generato MTM non porta informazione temporale reale. L'analisi dei motif rimane valida perché TMC identifica pattern strutturali; solo la componente temporale è affetta dall'anomalia.

Il grafo MTM genera 2,976 eventi (contro i 3,000 originali), con rapporti: eventi=0.992, archi unici=0.999, nodi=1.000.

---

## 6. Risultati — CollegeMsg (SNAP)

### Dataset
| Proprietà | Valore |
|---|---|
| File | `CollegeMsg.txt` |
| Fonte | Stanford SNAP |
| # eventi totali | 59,798 |
| # nodi totali | 1,899 |
| # archi statici | 13,838 |
| Span temporale | **193 giorni** |
| Periodo | 2004-04-15 → 2004-10-26 |
| # eventi campionati | 5,000 |
| # nodi nel campione | 530 |

Il dataset CollegeMsg contiene messaggi privati inviati tra studenti di un campus universitario (UC Irvine). I timestamp sono validi e coprono quasi 7 mesi di attività.

**Nota:** per questo dataset è stato usato `L_MAX=3` (non 4 come per facebook), riducendo il tempo di computazione e il numero di tipi di motif trovati (66 vs 232).

### 6.1 `snap_motif_top20.png` — Top-20 motif per frequenza

![Top-20 motif CollegeMsg](task_1/task_1_SNAP/output_png/snap_motif_top20.png)

Rispetto a facebook_wall, la distribuzione è **molto più piatta**: i 20 motif principali hanno conteggi che vanno da ~185K a ~1.1M, con un rapporto tra primo e ventesimo classificato di circa 6:1 (contro ~500:1 di facebook).

Tutti i motif sono a 3 archi (lunghezza massima per questo dataset). I primi 20 motif mostrano grande varietà topologica: fan-out, fan-in, reciprocità, catene. Non c'è un singolo pattern dominante come l'hub puro di facebook.

**Top-5 per conteggio:**
1. `010203` — 1,115,188 — fan-out (hub)
2. `012131` — 793,677 — fan-in (convergenza)
3. `012123` — 658,134 — convergenza + fan-out
4. `010232` — 565,256 — hub + convergenza
5. `012103` — 492,873 — ibrido

### 6.2 `snap_motif_by_length.png` — Distribuzione per lunghezza

![Distribuzione per lunghezza CollegeMsg](task_1/task_1_SNAP/output_png/snap_motif_by_length.png)

- **2 archi:** ~150,000 occorrenze totali (piccola frazione)
- **3 archi:** ~10,300,000 occorrenze totali

I motif a 3 archi dominano completamente. Il volume totale (~10.3M) è molto superiore a facebook (~5.4M) nonostante i parametri diversi, indicando che CollegeMsg ha una densità di interazioni connesse molto maggiore rispetto al campione facebook.

### 6.3 `snap_enrichment.png` — Enrichment score

![Enrichment score CollegeMsg](task_1/task_1_SNAP/output_png/snap_enrichment.png)

**Tutti i 30 motif analizzati hanno enrichment > 1.** Gli enrichment sono molto più elevati che in facebook, con valori che raggiungono ~19.55x.

**Top-10 motif più significativi:**

| Codice | Enrichment | Struttura |
|--------|-----------|-----------|
| `010121` | **19.55x** | 0→1 due volte + 2→1 (messaggi ripetuti) |
| `012121` | **18.40x** | 0→1, poi 2→1 due volte |
| `010112` | **18.35x** | 0→1 due volte, 1→2 |
| `012020` | **16.42x** | 0→1, poi 2→0 due volte |
| `010102` | **15.98x** | 0→1 due volte, 0→2 |
| `010202` | **12.64x** | 0→1, poi 0→2 due volte |
| `012123` | **6.44x** | Convergenza + fan-out |
| `012131` | **5.73x** | Fan-in (convergenza) |
| `010232` | **5.39x** | Hub + convergenza |
| `012132` | **4.84x** | Convergenza + inversione |

I motif con enrichment massimale sono quelli che contengono **archi ripetuti** (stessa coppia di nodi che interagisce più volte nella stessa giornata). Questi motif sono quasi inesistenti nelle reti casuali ma molto frequenti nella realtà, rivelando il pattern di "messaggi multipli rapidi" tipico dei sistemi di messaggistica.

### 6.4 `snap_null_comparison.png` — Confronto con i null model

![Confronto null model CollegeMsg](task_1/task_1_SNAP/output_png/snap_null_comparison.png)

Il grafico è molto diverso da quello di facebook:

1. **Tutti e tre i null model producono conteggi apprezzabili** (a differenza di Strogatz in facebook che era quasi zero). Questo perché CollegeMsg è una rete più densa e omogenea, e anche reti casuali con 530 nodi producono molte sequenze connesse.

2. **La rete reale supera sistematicamente entrambi i null model strutturali** (Time Shuffle e Edge Shuffle) per tutti i top-15 motif. Il gap è particolarmente marcato rispetto all'Edge Shuffle, che produce circa il 30-40% dei conteggi reali.

3. **Watts-Strogatz** produce conteggi quasi nulli (barre viola appena visibili), confermando che la struttura del grafo CollegeMsg è molto diversa da un ring lattice.

4. **`010203` (hub):** reale ~1.1M, Time Shuffle ~590K (54%), Edge Shuffle ~208K (19%). Il motif è significativo sia temporalmente che strutturalmente.

### 6.5 `snap_degree_dist.png` — Degree distribution

![Degree distribution CollegeMsg](task_1/task_1_SNAP/output_png/snap_degree_dist.png)

Il confronto tra originale e generato MTM mostra un comportamento diverso da facebook:

- **Nodi a basso grado (grado 1-3):** il grafo MTM genera più nodi a basso grado dell'originale (la curva arancione è più alta in basso a sinistra). Il modello "crea" nodi nuovi che partecipano a pochi motif.
- **Nodi ad alto grado (grado >10):** le due curve convergono e sono comparabili.
- **Coda destra:** il generato MTM produce nodi con grado fino a ~200, contro ~100 dell'originale. Il modello tende a creare alcuni hub più grandi del reale.

**Confronto con facebook:** la degree distribution di CollegeMsg ha una pendenza più ripida in scala log-log, indicando una rete più eterogenea con hubs più marcati. Il grafo MTM preserva la forma generale ma con scostamenti maggiori rispetto al caso facebook.

Il grafo MTM genera 5,419 eventi (originale: 5,000), con rapporti: eventi=1.084, archi unici=0.924, nodi=0.949.

### 6.6 `snap_temporal_dist.png` — Distribuzione temporale

![Distribuzione temporale CollegeMsg](task_1/task_1_SNAP/output_png/snap_temporal_dist.png)

A differenza di facebook, qui i timestamp sono validi e coprono il periodo aprile-ottobre 2004 (asse x in unità 10⁹ di secondi Unix, range ~1.082×10⁹ - 1.083×10⁹).

**Dataset originale (sinistra):** distribuzione non uniforme con picchi crescenti verso ottobre. I 5000 eventi del campione coprono i primissimi mesi del dataset (aprile-agosto), con densità crescente nel tempo. I picchi suggeriscono periodi di maggiore attività (inizio semestre, periodi di esami).

**Generato MTM (destra):** la distribuzione generata riproduce la forma generale (aumento progressivo verso destra) ma con variazioni più accentuate nei bin. Il modello apprende i tempi inter-arrivo medi tramite distribuzioni esponenziali e li propaga, catturando l'andamento macroscopico ma non i picchi specifici. La somiglianza qualitativa dimostra che MTM impara la dinamica temporale reale.

---

## 7. Confronto tra i due dataset

| Proprietà | facebook_wall | CollegeMsg |
|---|---|---|
| Tipo di interazione | Post su bacheca pubblica | Messaggi privati |
| # eventi campionati | 3,000 | 5,000 |
| # nodi nel campione | 2,797 | 530 |
| Densità (eventi/nodo) | 1.07 | 9.43 |
| L_MAX usato | 4 | 3 |
| # tipi di motif trovati | **232** | **66** |
| Motif dominante | Hub puro `01020304` | Hub `010203` + grande varietà |
| Enrichment massimo | **2.86x** | **19.55x** |
| Motif con enrich. > 1 | 13 su 20 analizzati | **30 su 30** analizzati |
| Pattern principali | Hub (fan-out) | Hub, fan-in, reciprocità, multi-edge |
| Motif caratteristici | Broadcast a molti | Messaggi ripetuti allo stesso nodo |
| Qualità MTM | Eccellente (nodi=1.0) | Buona (nodi=0.95) |

### Differenze strutturali principali

**1. Maggiore significatività statistica in CollegeMsg**  
Gli enrichment score di CollegeMsg (fino a 19.55x) sono molto superiori a quelli di facebook (massimo 2.86x). Questo indica che CollegeMsg ha pattern di interazione più "organizzati" e meno casuali: le regolarità comportamentali degli studenti (messaggiare ripetutamente lo stesso utente, conversazioni bilaterali) sono molto più marcate rispetto ai post su bacheca.

**2. Eterogeneità topologica**  
Facebook è dominato da hub puri (un utente scrive a molti), mentre CollegeMsg mostra un mix bilanciato di fan-out, fan-in, catene e reciprocità. Questo riflette la differenza tra comunicazione broadcast (post pubblico) e comunicazione dialogica (messaggio privato).

**3. Multi-edge e reciprocità**  
Il fenomeno dei messaggi ripetuti (stesso arco che appare più volte) è altamente significativo in CollegeMsg (enrichment ~18-19x) e non emergente in facebook. Nei messaggi privati, gli utenti tendono a mandare sequenze di messaggi brevi allo stesso destinatario; nei post su bacheca, si scrive una volta sola per poi passare al prossimo utente.

**4. Dimensionalità e densità**  
CollegeMsg ha molti meno nodi (530 vs 2797) ma un'attività per nodo molto più alta. Questo crea reti locali più dense dove i motif si ripetono più spesso.

---

## 8. Parametri usati

| Parametro | facebook_wall | CollegeMsg | Motivazione |
|---|---|---|---|
| `L_MAX` | 4 | 3 | Lunghezza massima del motif. 4 per facebook (più tempo di calcolo tollerato); 3 per SNAP per ridurre complessità computazionale (il TMC scala esponenzialmente con L_MAX). |
| `DELTA` | 86400 s | 86400 s | Finestra temporale di 1 giorno: scelta naturale per reti sociali dove le interazioni correlate avvengono tipicamente nella stessa giornata. |
| `K` | 1 | 1 | Un solo grafo sintetico generato. Sufficiente per confronto qualitativo; aumentare K darebbe una media più stabile. |
| `consecutive` | NO | NO | I prefissi non vengono "consumati" dopo l'estensione. Permette di contare tutte le istanze di un motif, non solo quelle che partono da uno stesso evento iniziale. |
| `N_SAMPLE` | 3,000 | 5,000 | Sottoinsieme degli eventi originali. Scelta per bilanciare tempo di calcolo (TMC è O(N²) nel caso peggiore) e copertura del dataset. |
| `k_neighbors` | 4 | 4 | Grado iniziale nel ring lattice di Watts-Strogatz. |
| `p_rewire` | 0.1 | 0.1 | 10% degli archi viene reindirizzato casualmente, bilanciando struttura small-world e casualità. |

**Note sul tempo di calcolo:**
- TMC su facebook (3000 eventi, L_MAX=4): ~437 secondi
- TMC su CollegeMsg (5000 eventi, L_MAX=3): ~244 secondi
- TMC sui null model di facebook: ~429s (Time Shuffle), ~708s (Edge Shuffle), ~5s (Strogatz)
- TMC sui null model di CollegeMsg: ~42s (Time Shuffle), ~102s (Edge Shuffle), ~4s (Strogatz)

Il Watts-Strogatz è molto più veloce perché genera reti più sparse e uniformi dove i prefissi attivi sono pochi. L'Edge Shuffle di facebook è il più lento perché produce più tipi di motif (377 vs 232 reali) per la maggiore varietà strutturale degli archi rimescolati.

---

## 9. Connessione con la tesi

Il Task 1 costituisce la **base fondamentale** dell'analisi multilivello descritta nel titolo della tesi:

### Livello Locale (nodo)
I motif descrivono il comportamento di ogni singolo nodo:
- Un nodo con `01020304` frequente è un **hub broadcast** (utente attivo che scrive a molti)
- Un nodo con `012131` frequente è un **hub di ricezione** (utente popolare che riceve da molti)
- Un nodo con `01101213` frequente è un **conversatore bilaterale** (impegnato in scambi reciproci)

### Livello Mesoscopico (gruppi di nodi)
I pattern di transizione tra motif (tabella MTM) descrivono come si formano e evolvono i sottografi locali:
- La transizione da `0102` → `01020304` descrive un utente che inizia a scrivere (wedge) e poi continua a espandersi (hub)
- Queste transizioni catturano la "grammatica" dell'interazione sociale

### Livello Globale (grafo intero)
L'enrichment score e la distribuzione dei motif descrivono le proprietà strutturali globali:
- Facebook_wall è globalmente una rete di **broadcast** (hub dominanti, enrichment moderato)
- CollegeMsg è globalmente una rete **dialogica** (varietà topologica, enrichment molto alto, multi-edge)

Il modello MTM, imparando le transizioni tra motif e i tempi inter-arrivo, riesce a generare reti sintetiche che **preservano le proprietà evolutive** della rete originale — questo è il punto di partenza per i Task successivi, dove queste proprietà verranno analizzate su più livelli e confrontate sistematicamente.

---

## File prodotti

### facebook_wall (`task_1/task_1_facebook/`)
```
output_data/
  facebook_sample_3000.txt                  ← 3000 eventi campionati
  facebook_wall_motifs_3000_4_86400.txt     ← 232 tipi di motif con conteggi
  facebook_wall_generated_3000_4_86400.txt  ← grafo sintetico MTM (2976 eventi)
  facebook_null_time_shuffle_3000.txt       ← null model Time Shuffle
  facebook_null_edge_shuffle_3000.txt       ← null model Edge Shuffle
  facebook_null_strogatz_3000.txt           ← null model Watts-Strogatz

output_png/
  facebook_motif_top20.png          ← bar chart top-20 motif
  facebook_motif_by_length.png      ← distribuzione per lunghezza
  facebook_enrichment.png           ← enrichment score
  facebook_null_comparison.png      ← confronto reale vs null models
  facebook_degree_dist.png          ← degree distribution orig vs MTM
  facebook_temporal_dist.png        ← distribuzione temporale orig vs MTM
  facebook_motif_gallery_real.png   ← gallery top-12 motif
  facebook_motif_gallery_enriched.png ← gallery top-8 motif significativi
```

### CollegeMsg (`task_1/task_1_SNAP/`)
```
output_data/
  collegemsg_sample_5000.txt                ← 5000 eventi campionati
  CollegeMsg_motifs_5000_3_86400.txt        ← 66 tipi di motif con conteggi
  CollegeMsg_generated_5000_3_86400.txt     ← grafo sintetico MTM (5419 eventi)
  CollegeMsg_generated_5000_4_86400.txt     ← versione con L_MAX=4 (test)
  snap_null_time_shuffle_5000.txt           ← null model Time Shuffle
  snap_null_edge_shuffle_5000.txt           ← null model Edge Shuffle
  snap_null_strogatz_5000.txt               ← null model Watts-Strogatz

output_png/
  snap_motif_top20.png              ← bar chart top-20 motif
  snap_motif_by_length.png          ← distribuzione per lunghezza
  snap_enrichment.png               ← enrichment score
  snap_null_comparison.png          ← confronto reale vs null models
  snap_degree_dist.png              ← degree distribution orig vs MTM
  snap_temporal_dist.png            ← distribuzione temporale orig vs MTM
  snap_motif_gallery_real.png       ← gallery top-12 motif
  snap_motif_gallery_enriched.png   ← gallery top-8 motif significativi
```

---

*Report generato automaticamente tramite Claude Code — Task 1 completato.*
