# CLAUDE.md — Contesto del tirocinio

**Autore:** Carlo Invernizzi  
**Tirocinio:** "Profili evolutivi multilivello di reti complesse tramite temporal network motif"  
**Ultima modifica:** 2026-05-12  
**Stato:** Task 1 ✅ — Task 2 ✅ — Task 3+ da definire

---

## Ambiente

- **OS:** macOS ARM64 (Apple Silicon) — Python 3.9.6
- **venv:** `~/venv` — kernel Jupyter registrato come "Python (venv)"
- **IDE:** VS Code con estensione Jupyter
- **Docker Desktop:** installato e funzionante (v29.4.1) — necessario per eseguire i binari Linux x86-64 di EvoMine

---

## Struttura del progetto

```
lab/
├── CLAUDE.md                              ← questo file (aggiornare ad ogni task)
├── task_1/                                ← COMPLETATO
│   ├── report_task1.md                    ← report completo con gallery e grafici
│   ├── spiegazione.txt
│   ├── task_1_facebook/
│   │   ├── ia-facebook-wall-wosn-dir.edges
│   │   ├── task1_facebook.ipynb
│   │   ├── output_data/                   ← motif, grafo MTM, null models
│   │   └── output_png/                    ← gallery, enrichment, degree dist...
│   └── task_1_SNAP/
│       ├── CollegeMsg.txt
│       ├── task1_snap.ipynb
│       ├── output_data/
│       └── output_png/
└── task_2/                                ← COMPLETATO
    ├── report_task2.md                    ← report completo con immagini embedded
    ├── task2_notes.md                     ← documentazione tecnica algoritmo EvoMine
    ├── task2_evomine.ipynb                ← notebook 12 sezioni
    ├── output_data/
    │   ├── college_evomine_20000_3.txt    ← 14 regole CollegeMsg (support=20000)
    │   ├── facebook_evomine_500_3.txt     ← 27 regole Facebook (support=500)
    │   └── dblp0305_profiles_rel.p        ← profilo DBLP serializzato
    ├── output_png/
    │   ├── temporal_dist_both.png
    │   ├── dblp_evolutionary_profile.png
    │   ├── dblp_all_patterns_gallery.png
    │   ├── dblp_tspan.png
    │   ├── CollegeMsg_profile.png
    │   ├── Facebook_profile.png
    │   └── comparison_heatmap.png
    └── EvoMine/                           ← repo della correlatrice (GERANIO toolkit)
        ├── algorithms/evomine             ← ELF 64-bit Linux x86-64 (non gira su macOS)
        ├── algorithms/germ                ← ELF 32-bit Linux i386 (non gira su macOS)
        ├── input-files/
        │   ├── collegemsg_ger.txt         ← 1899 nodi, 59835 archi, bucket giornalieri (1..194)
        │   └── facebook_ger.txt           ← 3757 nodi, 10000 archi, bucket settimanali (1..82)
        ├── output-files/                  ← log grezzi dei run Docker
        └── analysis.py / drawing.py / file_converters.py / general_mapping.py / running_algorithms.py / canonical.py
```

---

## Dataset

| Nome | Path | Archi | Nodi | Span | Formato |
|------|------|-------|------|------|---------|
| Facebook Wall | `task_1/task_1_facebook/ia-facebook-wall-wosn-dir.edges` | 855,542 | 45,813 | 1561 giorni | `% commento`, poi `src dst weight timestamp` (tab/spazio misto) — saltare righe `%`, colonne 0,1,3, rimuovere self-loop |
| CollegeMsg | `task_1/task_1_SNAP/CollegeMsg.txt` | 59,835 | 1,899 | 193.7 giorni | `src dst timestamp` (spazio, no header) — pulito, conversione diretta |

---

## Task 1 — TMC/MTM ✅ COMPLETATO

**Paper:** Liu & Sarıyüce, KDD 2023 — *"Using Motif Transitions for Temporal Graph Generation"*  
**Notebook:** `task1_facebook.ipynb`, `task1_snap.ipynb`  
**Report:** `task_1/report_task1.md`

### Cos'è stato fatto
- Traduzione C++ → Python dell'algoritmo TMC/MTM (7 funzioni mappate)
- TMC: conteggio temporal motif in finestra δ=86400s (1 giorno)
- MTM: apprendimento transizioni tra motif + generazione rete sintetica
- 3 null model: Time Shuffle, Edge Shuffle, Watts-Strogatz (k=4, p=0.1)
- Enrichment score = count_reale / media(count_null)
- Gallery grafica dei motif (nodi colorati, archi con ordine temporale)

### Parametri usati
| Parametro | facebook | CollegeMsg |
|-----------|----------|-----------|
| L_MAX | 4 | 3 |
| DELTA | 86400s | 86400s |
| N_SAMPLE | 3,000 | 5,000 |

### Risultati chiave — facebook_wall
- **232 tipi di motif** trovati (3000 eventi campionati)
- Motif dominante: `01020304` (hub fan-out a 4 nodi) — 3,678,598 occorrenze
- **13 motif con enrichment > 1**, max **2.86x** (hub puro)
- Motif significativi: hub puri e pattern di reciprocità (es. `01101213` = 2.57x)
- Motif sotto-rappresentati: quelli con arco finale verso il mittente originale
- Grafo MTM: 2,976 eventi generati, degree distribution quasi identica all'originale
- **Problema noto:** timestamp facebook_wall anomali (span=0) nel campione — TMC funziona ma distribuzione temporale MTM non è informativa

### Risultati chiave — CollegeMsg
- **66 tipi di motif** trovati (5000 eventi campionati)
- Motif più frequente: `010203` (hub) — 1,115,188 occorrenze — ma con varietà molto maggiore
- **Tutti e 30 i motif analizzati hanno enrichment > 1**, max **19.55x**
- Motif caratteristici: archi ripetuti tra stesse coppie (`010121` = 19.55x, `012121` = 18.40x) — messaggi multipli rapidi allo stesso utente
- Grafo MTM: 5,419 eventi generati, degree distribution simile con leggero eccesso di hub grandi

### Confronto tra i due dataset (Task 1)
| Aspetto | facebook_wall | CollegeMsg |
|---------|--------------|-----------|
| Pattern dominante | Hub broadcast (1→molti) | Mix bilanciato (hub, fan-in, reciprocità, multi-edge) |
| Enrichment max | 2.86x | 19.55x |
| Motif > 1 enrichment | 13/20 | 30/30 |
| Caratteristica unica | Hub "puri" massicci | Multi-edge (messaggi ripetuti ~20x più frequenti del caso) |
| Tipo di rete | Broadcast pubblico | Dialogo privato |

---

## Task 2 — EvoMine/GERM ✅ COMPLETATO

**Paper:** Galdeman, Zignani, Gaito — IEEE DSAA 2023 — *"Unfolding temporal networks through statistically significant graph evolution rules"*  
**Toolkit:** GERANIO — `task_2/EvoMine/`  
**Notebook:** `task2_evomine.ipynb` (12 sezioni)  
**Report:** `task_2/report_task2.md`

### Cos'è una GER
Regola predittiva: `Precondizione (body, t=0) → Postcondizione (head, t>0)`  
Supporto misurato con **MIB (Minimum Image-Based)** — standard del frequent subgraph mining.  
Due algoritmi: **GERM** (non diretto) e **EvoMine** (diretto, con label, con rimozione archi).

### Criticità tecniche risolte

**1. Binari Linux su macOS ARM64**  
I binari `evomine` e `germ` sono ELF Linux x86-64 → usare Docker:
```bash
docker run --rm -w /EvoMine \
  -v /Users/inve14/Desktop/UnInve/Tirocinio/lab/task_2/EvoMine:/EvoMine \
  ubuntu:22.04 \
  bash -c "/EvoMine/algorithms/evomine -s 20000 -e 3 -T full -t -d \
  -f /EvoMine/input-files/collegemsg_ger.txt \
  > /EvoMine/output-files/collegemsg_raw.txt 2>&1"
```
Output GER scritto nella working dir del container: `collegemsg_ger.txt.out.evomine.FULL.20000.3`  
→ copiare in `task_2/output_data/college_evomine_20000_3.txt`

**2. Timestamp: obbligatoria la discretizzazione in bucket**  
EvoMine crea uno snapshot per ogni timestamp unico. Con Unix epoch → OOM (exit 137).  
Soluzione obbligatoria prima della conversione GER:
```python
# CollegeMsg → bucket giornalieri
bucket = (ts - ts_min) // 86400 + 1       # → valori 1..194

# Facebook (10k) → bucket settimanali  
bucket = (ts - ts_min) // 604800 + 1      # → valori 1..82
```

**3. Support threshold**  
Con support=1000 o 5000 il mining si blocca su singoli pattern per ore (NP-hard).  
Su CollegeMsg usato support=20000 per ottenere risultati in tempi ragionevoli.

**4. Bug fix in running_algorithms.py**  
Riga 13: docstring con 5 spazi invece di 4 → `IndentationError`. Già corretto nel file.

### Parametri EvoMine usati
| Dataset | support | max_edges | modalità | bucket |
|---------|---------|-----------|----------|--------|
| CollegeMsg | 20,000 | 3 | directed | giornalieri (1..194) |
| Facebook (10k) | 500 | 3 | directed | settimanali (1..82) |
| DBLP (precomputed) | 5,000 | 3 | undirected (GERM) | annuali (1..3) |

### Risultati — DBLP precomputed (GERM, riferimento)
- **20 regole** trovate, support 5,054–109,044, mediana 10,297
- Profilo: decadimento power-law, top-5 regole = ~74% del peso totale
- T-span: 25% t=0 (statici), 40% t=1, 35% t=2
- Gallery: righe 1-4 tutte blu (solo precondizione), righe 5+ con archi verdi (postcondizione)

### Risultati — CollegeMsg (EvoMine, support=20000)
- **14 regole** trovate
- Support range: 147,998–214,115 (mediana 181,792, peso totale 2,535,031)
- **Profilo piatto**: frequenza relativa tra 0.058 e 0.085 — nessuna regola dominante
- Interpretazione: rete piccola e densa (1899 nodi), interazioni equilibrate tra tutti gli utenti

### Risultati — Facebook Wall 10k (EvoMine, support=500)
- **27 regole** trovate
- Support range: 510–43,467 (mediana 4,225, peso totale 167,433)
- **Profilo a legge di potenza**: regola 0 pesa ~26%, decadimento rapido verso zero
- Più regole trovate (27 vs 14) su un campione più piccolo → maggiore varietà strutturale

### Confronto tra i due dataset (Task 2)
| Aspetto | CollegeMsg | Facebook Wall |
|---------|-----------|--------------|
| Regole trovate | 14 | 27 |
| Support range | 148k–214k (lineare, compatto) | 510–43k (logaritmico, ampio) |
| Forma profilo | Piatto (~7% per regola) | Legge di potenza (26% per regola 0) |
| Regola dominante | Nessuna | Sì (regola 0 domina nettamente) |
| Heatmap | Colore distribuito su tutte le colonne | Concentrato nelle prime 2 colonne |
| Interpretazione | Rete omogenea, evoluzione bilanciata | Rete scale-free, evoluzione eterogenea |

### Confronto TMC/MTM vs EvoMine (Task 1 vs Task 2)
| | TMC/MTM | EvoMine |
|---|---------|---------|
| Output | Conteggi motif + enrichment | Regole predittive body→head con supporto |
| Causalità | Osservazionale | Predittiva |
| Finestra | Sliding window δ esatta | Bucket temporali discretizzati |
| Supporto | Conteggio grezzo | MIB |
| Uso | Caratterizzare la rete | Predire evoluzione futura |

---

## Note tecniche permanenti

- **Parser GER:** `file_converters.from_ger_output()` accetta sia `v 0 0` (GERM) che `v 0` (EvoMine)
- **Drawing:** le celle che usano `drawing.draw_rule()` richiedono `os.chdir(EVOMINE_DIR)` perché cercano `imgs/` relativa
- **Output EvoMine:** il binario scrive `<input_basename>.out.evomine.FULL.<sup>.<maxedge>` nella working dir del container. Copiare manualmente in `output_data/` con il nome che il notebook si aspetta
- **Dipendenze venv installate:** networkx==3.2.1, matplotlib, numpy, pandas, seaborn, mycolorpy, igraph, scipy, tqdm, ipykernel

---

## Task 3+ — Da definire

Le prossime task verranno assegnate dalla correlatrice. Aggiornare questa sezione e il resto del file ad ogni nuova task.
