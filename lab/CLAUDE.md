# Tirocinio — Profili evolutivi multilivello di reti complesse
# tramite temporal network motif

## Stato dei task

### Task 1 — KDD23-MTM ✅ COMPLETATO
- Traduzione C++ → Python di MTM e TMC
- Null models: Time Shuffle, Edge Shuffle, Watts-Strogatz
- Visualizzazione grafica dei motif
- Test su facebook_wall (3000 eventi, L_MAX=4, 232 motif)
- Test su CollegeMsg (5000 eventi, L_MAX=3, 66 motif)
- Report: lab/task_1/report_task1.md

### Task 2 — EvoMine ✅ COMPLETATO
- Studio paper: Galdeman et al. IEEE DSAA 2023
- Run EvoMine via Docker (Linux ELF su macOS ARM64)
- Test su CollegeMsg (support=20000, 14 regole)
- Test su Facebook Wall (support=500, 27 regole)
- Report: lab/task_2/report_task2.md

### Task 3 — Comparazione MTM vs EvoMine ✅ COMPLETATO (in attesa conferma granularità da correlatrice)
- [x] Sample dataset (CollegeMsg 5000 eventi) → output_data/collegemsg_sample_5000.txt
- [x] MTM con consecutive=YES → transition matrix, **tre versioni**:
  - originale: L_MAX=4, DELTA=86400s (6×12, 53/72 celle osservate)
  - riallineata: L_MAX=3, DELTA=1800s/30min (6×12, 53/72 celle osservate)
  - riallineata: L_MAX=3, DELTA=3600s/1h (6×12, 54/72 celle osservate)
- [x] Analisi granularità temporale (inter-event time) → mediana IET=47s,
  candidate finali: 30min (1800s) e 1h (3600s)
- [x] Preprocessing GER per 5 granularità candidate (30min/1h/6h/12h/1d)
- [x] EvoMine con projection=event, support=1 eseguito per **entrambe** le
  granularità candidate (30min: 197s, 62 regole; 1h: 97s, 62 regole)
- [x] Codifica pattern EvoMine con stessa codifica MTM (encode_motif) per
  entrambe le granularità, confrontata con le rispettive matrici MTM L_MAX=3
- [ ] **Decisione finale tra 30min e 1h** — da concordare con la correlatrice
  (entrambe le granularità danno risultati qualitativamente equivalenti:
  stesso numero di regole, stesse regole comparabili, stesse regole trovate
  in MTM — cambia solo il tempo di calcolo e la scala dei supporti assoluti)

### Task 4 — Scalabilità TMC + Confronto MTM↔EvoMine + Evoluzione strutturale ✅ COMPLETATO
- Benchmark TMC su CollegeMsg al variare di N (500/1000/2000/3000/5000),
  DELTA=86400s, L_MAX=3, consecutive=YES → tempi tutti <0.1s
- Benchmark TMC al variare di DELTA (1800/3600/21600/43200/86400s), N=5000
  → n_motifs distinti costante (60) a tutte le granularità, cresce solo
  n_instances; tempi dell'ordine di decine di ms (rumore di misura non
  trascurabile a queste scale)
- Raccomandazione: N=5000 (intero campione Task 3, tempo trascurabile),
  DELTA=1800s (30 min) — coerente con analisi IET del Task 3
- Run EvoMine (support=1) confrontato per entrambe le granularità candidate:
  30min (197s) vs 1h (97s) — tempo quasi raddoppiato ma sostenibile
- Confronto MTM↔EvoMine con codifica canonica comune (encode_motif), per
  entrambe le granularità: 62 regole GER codificate, 12 strutturalmente
  comparabili con MTM (2 archi body + 1 arco head), 10/12 trovate anche in
  MTM, risultato identico a entrambe le granularità
- Nuova sezione: evoluzione strutturale del dataset nel tempo (archi/nodi
  attivi per bucket, nodi nuovi cumulativi, novelty rate) per entrambe le
  granularità — evidenzia un burst di attività finale a bassa novità
- Report: lab/task_4/task4_scalability.ipynb, lab/task_4/task4_evomine_coding.ipynb,
  output_data/scalability_summary.txt, output_data/structural_evolution_summary.txt

### Task 5 — Confronto MTM↔EvoMine con codifica canonica GERANIO ✅ COMPLETATO
- Dataset: CollegeMsg 5000 eventi, granularità 1h (support=1), stessi input del
  Task 3/4 (`collegemsg_sample_5000.txt`, `collegemsg_evomine_1h_s1_raw.txt`)
- Nuova codifica canonica condivisa MTM/EvoMine: rinomina nodi per ordine di
  prima apparizione, ordina archi per `(t_logico, src, dst)`, stringa
  `src_dst_t-src_dst_t-...` (schema confermato dalla correlatrice, diverso
  dalla digit-code `encode_motif` usata nei Task 3/4)
- **Attenzione — asimmetria di clock logico tra MTM ed EvoMine** (per
  costruzione, definita esplicitamente nel prompt del Task 5): MTM usa un
  orologio sequenziale (`t=0,1,2,...`, una posizione per arco), EvoMine un
  orologio binario body/head (`t=0` per tutti gli archi di precondizione,
  `t=1` per tutti quelli di postcondizione, a prescindere da quanti sono).
  Le due convenzioni coincidono solo quando la precondizione EvoMine ha
  esattamente 1 arco body + 1 arco head. Risultato: sull'unione dei codici
  lunghi (Step 1d) solo 3/148 stringhe sono condivise tra le due fonti, e le
  matrici di transizione finali (Step 3 vs 4) non hanno celle in comune
  (0 celle comuni, 54 esclusive MTM, 8 esclusive EvoMine) — non è un errore
  di implementazione ma conseguenza diretta delle definizioni richieste
- Cutoff regole EvoMine: gomito primario a rank 14 (tutte body-only, nessuna
  transizione utile) → scelto invece il gomito secondario a rank 23
  (N=23, dentro il range 15-30 indicato come fallback), che include 8 vere
  transizioni precondizione→postcondizione
- Transition matrix EvoMine (8 celle, top-23 regole): frequenza_pre cercata
  tra le regole body-only sulle 62 totali; 1 cella su 8 con precondizione
  mai osservata come pattern body-only autonomo (`0_1_0-2_3_0`, archi
  sconnessi) → probabilità non calcolabile (NaN), non ricomputata dal
  dataset grezzo (fuori scopo)
- Transition matrix MTM in codifica GERANIO: stessa forma 6×12 e stessi
  valori di probabilità della matrice Task 3 (es. `0102→03` = 0.375,
  verificato), solo rietichettata con i codici corti p1..pN — probabilità
  calcolata escludendo lo stato di non-estensione 'S' dal denominatore,
  come in Task 3 dopo la correzione della correlatrice
- **Chiarimento della correlatrice sulle 0 celle in comune**: le 0 celle
  condivise tra le transition matrix finali sono attese per costruzione
  (MTM ed EvoMine tracciano l'evoluzione della rete in modo diverso), non
  un errore. Usare la stessa struttura di visualizzazione (transition
  matrix, stesso formato heatmap) per entrambi gli algoritmi serve al
  confronto qualitativo affiancato, non a garantire sovrapposizione
  numerica — assi, codici e valori restano specifici di ciascun algoritmo.
  Il confronto realmente informativo è sui pattern individuali della
  codifica canonica condivisa: dei 148 codici totali, 3 pattern a 2 archi
  sono condivisi tra MTM ed EvoMine (`p18`=`0_1_0-0_2_1`,
  `p21`=`0_1_0-2_0_1`, `p36`=`0_1_0-1_0_1`). Le rispettive frequenze
  relative (count/supporto del pattern sul totale dei pattern trovati da
  ciascun algoritmo: 3202 su tutti i 60 motif MTM, 256498 sulle 62 regole
  EvoMine) differiscono di 30-40× in valore assoluto (es. `p18`: 20.08% in
  MTM contro 0.61% in EvoMine) ma conservano lo stesso ordinamento
  qualitativo in entrambe le fonti (`p18 > p21 > p36`) — indicazione di
  complementarità tra i due algoritmi più che di sovrapponibilità diretta.
  Dettagli in `task_5/output_data/shared_patterns_comparison.csv` e
  sezione "8. Discussion" del report.
- Report: lab/task_5/task5_comparison.ipynb,
  lab/task_5/report_task5_completo.docx / .pdf

## Dataset usati
- CollegeMsg: lab/task_1/task_1_SNAP/CollegeMsg.txt
- Facebook Wall: lab/task_1/task_1_facebook/ia-facebook-wall-wosn-dir.edges

## Struttura cartelle
```
lab/
├── CLAUDE.md          ← questo file
├── task_1/            ← MTM/TMC
├── task_2/            ← EvoMine
├── task_3/            ← Comparazione MTM vs EvoMine ✅
│   ├── task3_comparison.ipynb
│   ├── task3_granularity_analysis.ipynb
│   ├── task3_notes.md
│   ├── report_task3_completo.docx / .pdf
│   ├── output_png/
│   │   ├── mtm_transition_matrix.png                  (L_MAX=4, DELTA=86400s — originale)
│   │   ├── mtm_transition_matrix_L3_30min.png          (L_MAX=3, DELTA=1800s — riallineata)
│   │   ├── mtm_transition_matrix_L3_1h.png             (L_MAX=3, DELTA=3600s — riallineata)
│   │   ├── inter_event_time_dist.png
│   │   ├── collegemsg_hourly_distribution.png
│   │   └── collegemsg_iet_distribution.png
│   └── output_data/
│       ├── collegemsg_sample_5000.txt
│       ├── mtm_transition_matrix.csv                   (originale)
│       ├── mtm_transition_matrix_L3_30min.csv          (riallineata)
│       ├── mtm_transition_matrix_L3_1h.csv             (riallineata)
│       ├── inter_event_stats.txt
│       ├── granularity_summary.txt
│       ├── collegemsg_ger_{30min,1h,6h,12h,1d}.txt     (preprocessing GER, 5 granularità)
│       ├── collegemsg_evomine_30min_s1_raw.txt         (output grezzo EvoMine, 30min)
│       ├── collegemsg_evomine_1h_s1_raw.txt            (output grezzo EvoMine, 1h)
│       └── evomine_30min_timing.txt
└── task_4/             ← Scalabilità + Confronto + Evoluzione strutturale ✅
    ├── task4_scalability.ipynb
    ├── task4_evomine_coding.ipynb
    ├── report_task4_completo.docx / .pdf
    ├── output_png/
    │   ├── execution_times.png
    │   ├── tmc_scalability.png
    │   ├── tmc_delta_scalability.png
    │   ├── evomine_granularity_comparison.png          (30min vs 1h)
    │   ├── structural_evolution_30min.png
    │   └── structural_evolution_1h.png
    └── output_data/
        ├── tmc_scalability.csv
        ├── tmc_delta_scalability.csv
        ├── scalability_summary.txt
        ├── execution_times.csv
        ├── evomine_coded_rules.csv                     (1h, storico — confrontato con matrice L_MAX=4)
        ├── evomine_1h_coded_rules.csv                  (1h — confrontato con matrice L_MAX=3/1h)
        ├── evomine_30min_coded_rules.csv               (30min — confrontato con matrice L_MAX=3/30min)
        ├── mtm_evomine_comparison.csv                  (1h, storico, vs matrice originale L_MAX=4)
        ├── mtm_evomine_1h_comparison.csv                (1h, vs matrice riallineata L_MAX=3)
        ├── mtm_evomine_30min_comparison.csv             (30min, vs matrice riallineata L_MAX=3)
        ├── structural_evolution_30min.csv / _1h.csv
        └── structural_evolution_summary.txt
└── task_5/             ← Confronto MTM↔EvoMine, codifica canonica GERANIO ✅
    ├── task5_comparison.ipynb
    ├── report_task5_completo.docx / .pdf
    ├── output_png/
    │   ├── support_distribution.png
    │   ├── evomine_transition_matrix.png
    │   ├── mtm_transition_matrix_geranio.png
    │   ├── comparison_heatmaps.png
    │   └── shared_patterns_frequency.png    (freq. relativa dei 3 pattern condivisi, MTM vs EvoMine)
    └── output_data/
        ├── canonical_mapping.csv        (unione codici lunghi MTM+EvoMine → p1..pN, fonte)
        ├── evomine_top_rules.csv        (top 23 regole per supporto)
        ├── evomine_transition_matrix.csv
        ├── mtm_transition_matrix_geranio.csv
        └── shared_patterns_comparison.csv   (i 3 pattern condivisi: ruolo/count/freq.rel. in MTM vs EvoMine)
```

## Note tecniche importanti
- EvoMine gira solo su Linux x86-64 → usare Docker
- Timestamp facebook_wall hanno problema span=0 nel campione
- EvoMine richiede discretizzazione timestamp (non Unix epoch)
- MTM consecutive=YES per essere comparabile con EvoMine
- Nel formato grezzo di EvoMine l'etichetta d'arco **3 = precondizione
  (body, t=0)** e **1 = postcondizione (head, t>0)** — l'opposto
  dell'intuizione naturale, verificato in task_2/EvoMine/file_converters.py
  (mapping_ts = {3: 0, 1: 1})
- I report .docx sono esportati da Pages: le immagini incorporate finiscono
  con estensione `.undefined` invece di `.png`/`.jpg`, il che rompe la
  lettura con python-docx finché non si aggiungono gli `Override` mancanti
  in `[Content_Types].xml` (image/png) prima di riaprire il file — il
  problema non si ripresenta nei file poi risalvati da python-docx
- Granularità EvoMine e L_MAX/DELTA di MTM vanno sempre tenuti allineati tra
  loro per rendere il confronto MTM↔EvoMine valido — la mancata allineatura
  (EvoMine a 1h mentre la raccomandazione IET indicava 30min; MTM a L_MAX=4
  mentre i benchmark TMC usavano L_MAX=3) è stata identificata e corretta
  nell'iterazione del 2026-07-08 (vedi sotto)
- La decisione finale tra granularità 30min e 1h resta aperta: sentire la
  correlatrice — nel frattempo tutte le analisi (MTM, EvoMine, confronto,
  evoluzione strutturale) vengono prodotte per **entrambe**

## Risultati chiave Task 3 (aggiornati al 2026-07-08)
- MTM transition matrix (L_MAX=4/86400s, originale): top transizione
  `0102 → 03` (P=0.41) — star verso nodo nuovo
- MTM transition matrix riallineate a L_MAX=3: la transizione dominante
  diventa `0101 → 01` (ripetizione dello stesso arco, P≈0.45 sia a 30min
  che a 1h); `0102 → 03` resta comunque tra le prime (P=0.43 a 30min,
  P=0.38 a 1h). Differenza numerica media rispetto all'originale: ~0.02
  (max ~0.09) — nessuna differenza strutturale significativa
- IET mediana=47s, P95=629s, P99=2123s → distribuzione con heavy tail
  (burst di messaggi)
- Granularità candidate finali per EvoMine: **30 minuti (1800s)** — 98.6%
  IET entro 1 bucket, 747 bucket — e **1 ora (3600s)** — 99.4% IET entro
  1 bucket, 374 bucket. Entrambe testate in parallelo nel Task 4.

## Risultati chiave Task 4 (aggiornati al 2026-07-08)
- EvoMine support=1: 62 regole GER trovate a entrambe le granularità;
  30min impiega 197s, 1h impiega 97s (+103%)
- Confronto MTM↔EvoMine: 12/62 regole strutturalmente comparabili (2 archi
  body + 1 arco head), 10/12 trovate anche in MTM — identico a entrambe le
  granularità; le regole esclusive di EvoMine sono quasi tutte pattern
  senza precondizione (body vuoto)
- Evoluzione strutturale: 530 nodi unici, 2020 coppie src→dst uniche
  (indipendenti dalla granularità); il 50% dei nodi arriva già al 26%
  degli eventi (evento 1315/5000) ma solo al 68% del tempo trascorso;
  picco di attività nell'ultimo ~5% del periodo osservato; novelty rate
  medio 62–67%, in calo da ~95% nei primi bucket a ~48% negli ultimi —
  segno di un burst finale che riattiva prevalentemente nodi già noti
