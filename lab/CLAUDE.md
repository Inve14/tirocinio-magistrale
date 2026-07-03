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

### Task 3 — Comparazione MTM vs EvoMine 🔄 IN CORSO
- [x] Sample dataset (CollegeMsg 5000 eventi) → output_data/collegemsg_sample_5000.txt
- [x] MTM con consecutive=YES → transition matrix (6×12, L_MAX=4, DELTA=86400s)
- [x] Analisi granularità temporale (inter-event time) → mediana IET=47s, suggerito 1800s
- [ ] Preprocessing temporale (⚠️ sentire correlatrice)
- [ ] EvoMine con projection=event, support=1
- [ ] Codifica pattern EvoMine con stessa codifica MTM

## Dataset usati
- CollegeMsg: lab/task_1/task_1_SNAP/CollegeMsg.txt
- Facebook Wall: lab/task_1/task_1_facebook/ia-facebook-wall-wosn-dir.edges

## Struttura cartelle
```
lab/
├── CLAUDE.md          ← questo file
├── task_1/            ← MTM/TMC
├── task_2/            ← EvoMine
└── task_3/            ← Comparazione (in corso)
    ├── task3_comparison.ipynb
    ├── task3_notes.md
    ├── output_png/
    │   ├── mtm_transition_matrix.png
    │   └── inter_event_time_dist.png
    └── output_data/
        ├── collegemsg_sample_5000.txt
        ├── mtm_transition_matrix.csv
        └── inter_event_stats.txt
```

## Note tecniche importanti
- EvoMine gira solo su Linux x86-64 → usare Docker
- Timestamp facebook_wall hanno problema span=0 nel campione
- EvoMine richiede discretizzazione timestamp (non Unix epoch)
- MTM consecutive=YES per essere comparabile con EvoMine
- Sentire la correlatrice prima di scegliere granularità temporale
  per il preprocessing del Task 3

## Risultati chiave Task 3 (aggiornati al 2026-06-17)
- MTM transition matrix: top transizione `0102 → 03` (P=0.41) — star verso nodo nuovo
- IET mediana=47s, P95=629s → distribuzione con heavy tail (burst di messaggi)
- Granularità suggerita per EvoMine: sub-oraria (1800s = 30 min)
  - Produce ~747 bucket su 15.6 giorni, ~7 eventi/bucket
  - 99% degli IET ricade entro 1 bucket
