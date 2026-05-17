# Task 2 — EvoMine / GERANIO: Note e Risultati

**Autore:** Carlo Invernizzi  
**Data:** 2026-05-11  
**Tema:** Mining di Graph Evolution Rules in reti temporali con EvoMine (toolkit GERANIO)  
**Paper:** Galdeman, Zignani, Gaito — *Unfolding temporal networks through statistically significant graph evolution rules*, IEEE DSAA 2023

---

## 1. Cos'è EvoMine

EvoMine mina **Graph Evolution Rules (GERs)** da grafi temporali. Una GER è una regola della forma:

```
Precondizione (body, t=0)  →  Postcondizione (head, t>0)
```

- La **precondizione** è un sottoinsieme di archi già presenti al tempo t=0.
- La **postcondizione** è l'insieme di nuovi archi che appaiono in un momento successivo (t=1, 2, …).

Esempio intuitivo: "Se A→B e B→C esistono, con alta probabilità apparirà A→C."

Il supporto è misurato con **MIB (Minimum Image-Based) support**, lo standard del frequent subgraph mining, per evitare l'overcounting di match sovrapposti.

### Due algoritmi nel toolkit GERANIO
| Algoritmo | Tipo di grafo | Note |
|-----------|--------------|-------|
| **GERM** | Non diretto | Più semplice, output precomputed disponibile nella repo |
| **EvoMine** | Diretto, con rimozione archi, con label | Estende GERM, supporta directed graphs |

---

## 2. Come funziona l'algoritmo (pipeline step-by-step)

### Step 1 — Preparazione input
Convertire la lista di archi (src, dst, timestamp) nel formato GER:
```
t # 0
v <node_id> 0       ← un nodo per riga (label sempre 0 per grafi non etichettati)
...
e <src> <dst> <ts>  ← un arco per riga, ordinati per timestamp crescente
```
La funzione `file_converters.from_txtfile_to_gerinput()` gestisce questa conversione automaticamente. I node ID devono essere interi contigui a partire da 0, i self-loop vanno rimossi.

### Step 2 — Esecuzione del binario
```bash
cd EvoMine/
# EvoMine (directed)
./algorithms/evomine -s <support> -e <max_edges> -T <projection> -t -d -f <input_file>

# GERM (undirected)
./algorithms/germ <support> <input_file> <max_edges>
```
Output: file `.out.evomine.<PROJECTION>.<support>.<max_edges>` (EvoMine) o `.out.<support>.<max_edges>.REL` (GERM).

### Step 3 — Parsing dell'output
Ogni blocco `t # <id> <support>` nel file di output definisce una GER. Gli archi con timestamp 0 sono la precondizione, quelli con timestamp >0 la postcondizione.
```python
info_list, patterns, support_patterns, mapping = f.from_ger_output(output_file)
pattern_list, support = f.obtain_pattern_list(patterns, support_patterns, algorithm='germ')
```

### Step 4 — Mapping canonico
Assegna ID consistenti a pattern isomorfi attraverso diversi run/dataset:
```python
new_pattern_list, new_support = gm.mapping_pattern_ids(
    algo='germ', patterns=pattern_list, support=support,
    general_mapping=dict(), mapped_patterns_path='...', directed=False
)
```

### Step 5 — Profilo evolutivo
Converte i conteggi in frequenze relative → vettore "profilo" del dataset:
```python
profile = a.get_profiles(support_file=new_support, all_patterns_id=all_ids)
```

### Step 6 — Visualizzazione
```python
d.draw_rule('germ', edge_list, w_box=4)           # singola regola
d.draw_several_patterns('germ', edge_lists, columns=5)  # gallery
a.plot_profile(profile)                           # profilo evolutivo
a.plot_heatmap(profiles, ger_ids)                 # confronto tra dataset
a.t_span_plot(pattern_list)                       # distribuzione t-span
```

---

## 3. Parametri principali

| Flag CLI | Tipo | Descrizione |
|----------|------|-------------|
| `-s` | int | **Minimum support**: soglia di frequenza. Più alta → meno regole, più frequenti. Es: 5000 per DBLP (~230k archi), 1000 per CollegeMsg (~60k archi) |
| `-e` | int | **Max edges in postcondition**: complessità massima della testa della regola. Tipicamente 3 o 4 |
| `-T` | str | **Tipo di supporto**: `full` = MIB completo; `neigh` = MIB di vicinato; `event` = event-based |
| `-d` | flag | Modalità **grafo diretto** |
| `-u` | flag | Modalità **grafo non diretto** |
| `-r` | flag | Includi eventi di **rimozione archi** (non solo inserimento) |
| `-c` | flag | Grafo con **label sugli archi** |
| `-C` | flag | Grafo con **label sui nodi** |
| `-t` | flag | Modalità **temporale** (obbligatorio per grafi temporali) |
| `-f` | path | File di input in formato GER |

Per GERM la CLI è più semplice: `./algorithms/germ <support> <input_file> <max_edges>`

---

## 4. Come eseguire il codice

### Blocco critico: compatibilità binari
I file `evomine` e `germ` sono **Linux x86-64 ELF** — non girano su macOS ARM64 (Apple Silicon). Restituiscono `exec format error (exit code 126)`.

### Soluzione Docker (macOS)
```bash
docker pull ubuntu:22.04

# CollegeMsg — directed, support=1000, max 3 archi in postcondition
docker run --rm \
  -v /Users/inve14/Desktop/UnInve/Tirocinio/lab/task_2/EvoMine:/EvoMine \
  ubuntu:22.04 \
  /EvoMine/algorithms/evomine -s 1000 -e 3 -T full -t -d \
  -f /EvoMine/input-files/collegemsg_ger.txt

# Facebook Wall (campione 10k) — directed, support=500
docker run --rm \
  -v /Users/inve14/Desktop/UnInve/Tirocinio/lab/task_2/EvoMine:/EvoMine \
  ubuntu:22.04 \
  /EvoMine/algorithms/evomine -s 500 -e 3 -T full -t -d \
  -f /EvoMine/input-files/facebook_ger.txt
```
Dopo il run, copiare l'output `.out.evomine.*` in `task_2/output_data/` e rieseguire le sezioni 10-11 del notebook.

### File già convertiti in formato GER
| File | Nodi | Archi | Note |
|------|------|-------|------|
| `EvoMine/input-files/collegemsg_ger.txt` | 1,899 | 59,835 | Dataset completo |
| `EvoMine/input-files/facebook_ger.txt` | 3,757 | 10,000 | Campione dei primi 10k archi |

---

## 5. Dataset — Risultati osservati

### Statistiche di base (Sezione 3 del notebook)

| Dataset | Archi | Nodi unici | Range temporale (epoch) | Span (giorni) |
|---------|-------|-----------|------------------------|---------------|
| **CollegeMsg** | 59,835 | 1,899 | 1082040961 – 1098777142 | **193.7** |
| **Facebook Wall** | 855,542 | 45,813 | 1097725406 – 1232598691 | **1561.0** |

### Distribuzione temporale (grafico `temporal_dist_both.png`)
- **CollegeMsg**: attività concentrata nei primi 50 giorni (picco ~6,200 archi/bin), poi calo netto. Secondo picco minore attorno ai giorni 75-125. Tipico di un semestre universitario.
- **Facebook Wall**: crescita monotona sull'intero arco di ~4.3 anni, con accelerazione forte nell'ultimo anno (picco finale ~60,000 archi/bin). Classico pattern di adozione social.

---

## 6. Risultati DBLP 2003-05 (output precomputed GERM, Sezione 6-8)

L'unico output già disponibile nella repo è quello GERM sul dataset di co-authorship DBLP 2003-2005 (support=5000, max_edges=3).

### Statistiche aggregate
| Metrica | Valore |
|---------|--------|
| Regole totali trovate | **20** |
| Support minimo | 5,054 (regola 19) |
| Support massimo | 109,044 (regola 0) |
| Support mediano | 10,297 |
| Peso totale (somma support) | 529,964 |

### Top 5 regole per supporto
| Rank | Pattern ID | Support | Archi totali |
|------|-----------|---------|-------------|
| 1 | 0 | 109,044 | 1 |
| 2 | 1 | 83,197 | 2 |
| 3 | 2 | 76,462 | 3 |
| 4 | 3 | 68,876 | 3 |
| 5 | 4 | 56,292 | 3 |

Le top 5 regole pesano ~74% del supporto totale.

### Distribuzione per dimensione del pattern
| Archi nel pattern | Numero di regole |
|------------------|-----------------|
| 1 | 1 |
| 2 | 3 |
| 3 | 16 |

16 delle 20 regole hanno 3 archi totali (corpo + testa).

### Profilo evolutivo (grafico `dblp_evolutionary_profile.png`)
Decadimento a legge di potenza: le prime 5 regole dominano il profilo, poi c'è un salto brusco (~0.107 → ~0.028) dalla regola 4 alla 5, con coda lunga e piatta dalle regole 5-19.

### Analisi T-Span (grafico `dblp_tspan.png`)
Il **t-span** è la distanza temporale massima tra due archi di una regola.

| T-span | Regole | Frequenza | Interpretazione |
|--------|--------|-----------|----------------|
| 0 | 5 (regole 0-4) | ~25% | Pattern statici: tutti gli archi al t=0 (solo precondizione, nessuna postcondizione esplicita) |
| 1 | 8 | ~40% | Evoluzione immediata: il corpo viene completato al passo successivo |
| 2 | 7 | ~35% | Evoluzione in 2 passi |

Le regole 0-4 hanno t-span=0: rappresentano sottografi statici frequenti (non vera evoluzione body→head).  
T-span per pattern: `{0:0, 1:0, 2:0, 3:0, 4:0, 5:1, 6:1, 7:1, 8:1, 9:1, 10:1, 11:2, 12:2, 13:2, 14:1, 15:2, 16:2, 17:1, 18:2, 19:2}`

### Gallery dei 20 pattern (`dblp_all_patterns_gallery.png`)
- **Archi blu** = precondizione (t=0)
- **Archi verdi** = postcondizione (t=1 o t=2)

Riga 1 (regole 0-4, t-span=0): singolo arco, catena 2, triangolo, catena 3, stella — tutto blu, nessuna postcondizione.  
Riga 2+ (regole 5-19): appaiono archi verdi, regole vere con corpo→testa.

---

## 7. Differenze chiave rispetto a MTM/TMC (Task 1)

| Dimensione | MTM / TMC (Task 1) | EvoMine / GERM (Task 2) |
|-----------|-------------------|------------------------|
| **Obiettivo** | Contare pattern temporali ricorrenti | Minare regole predittive: body→head |
| **Output** | Vettore di conteggi / enrichment score per motif | Lista di regole con supporto |
| **Causalità** | Osservazionale: co-occorrenza in finestra δ | Predittiva: precondizione → postcondizione |
| **Direzione** | Solo directed (MTM) | Entrambe (EvoMine=directed, GERM=undirected) |
| **Modello temporale** | Timestamp esatti, sliding window di dimensione δ | Timestamp discretizzati in bucket (prima/dopo) |
| **Struttura dei pattern** | Sottografi k-nodo a dimensione fissa | Variabile: body (t=0) + head (t>0) |
| **Misura di supporto** | Conteggio grezzo delle occorrenze | MIB (Minimum Image-Based) support |
| **Parametro chiave** | δ (finestra), k (dimensione motif) | support threshold, max archi in head |
| **Significatività** | Confronto con null models (shuffle) | Soglia di frequenza minima |
| **Confronto tra dataset** | Diretto: stesse dimensioni vettore | Richiede mapping per isomorfismo degli ID |
| **Caso d'uso** | Caratterizzare la dinamica della rete | Predire la formazione futura di archi |
| **Paper principale** | Paranjape et al. 2017 | Galdeman et al. IEEE DSAA 2023 |

---

## 8. Struttura della repository

```
EvoMine/
├── algorithms/
│   ├── evomine        ← ELF 64-bit Linux x86-64 (non gira su macOS ARM64)
│   └── germ           ← ELF 32-bit Linux i386 (non gira su macOS ARM64)
├── input-files/       ← file GER di input (inclusi i nostri dataset convertiti)
├── output-files/      ← output precomputed (dblp0305.5000.3.txt)
├── raw-datasets/      ← edge list grezze (dblp0305.txt)
├── processed-outputs/ ← pattern mappati (pickle)
├── imgs/              ← immagini per la pipeline
├── analysis.py        ← profili, heatmap, t-span
├── canonical.py       ← forme canoniche via igraph/bliss
├── drawing.py         ← visualizzazione pattern e regole
├── file_converters.py ← parsing input/output
├── general_mapping.py ← mapping ID tra dataset
├── running_algorithms.py ← wrapper Python attorno ai binari
├── geranio.ipynb      ← notebook di esempio della repo
└── requirements.txt   ← networkx==3.2.1, mycolorpy==1.5.1
```

---

## 9. Dipendenze installate in ~/venv

Già presenti: `networkx==3.2.1`, `matplotlib`, `numpy`, `pandas`  
Aggiunte: `seaborn`, `mycolorpy`, `igraph`, `scipy`, `tqdm`, `ipykernel`

---

## 10. Stato del lavoro e prossimi passi

| Componente | Stato |
|-----------|-------|
| Codice Python (parsing, mapping, visualizzazione) | Funzionante |
| Analisi output precomputed DBLP (20 regole) | Completata |
| Conversione dataset in formato GER | Completata |
| Esecuzione binario EvoMine | **BLOCCATA** — Linux ELF su macOS ARM64 |
| Analisi CollegeMsg e Facebook Wall | In attesa del binary run |
| Heatmap comparativa tra i due dataset | In attesa del binary run |

### Prossimi passi
1. Eseguire EvoMine con Docker (comandi nella Sezione 4) sui due dataset convertiti.
2. Copiare i file `.out.evomine.*` in `task_2/output_data/`.
3. Rieseguire le sezioni 10-11 del notebook per profili e heatmap comparativa.
4. Confrontare i profili evolutivi di CollegeMsg e Facebook Wall.
5. Esplorare il flag `-r` (rimozione archi) per catturare pattern di decadimento.
