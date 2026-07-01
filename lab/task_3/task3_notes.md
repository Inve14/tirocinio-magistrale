# Task 3 — Note e Risultati Preliminari

## 1. Transition Matrix MTM

**Dataset:** CollegeMsg, 5000 eventi (primi 5000 per timestamp)
**Parametri:** L_MAX=4, DELTA=86400s (1 giorno), consecutive=YES

**Dimensioni matrice:** 6 prefissi × 12 terzi-eventi
**Celle osservate:** 53 / 72 (73.6%)

### Transizioni più frequenti

| Prefisso (2 eventi) | Terzo evento | Probabilità | Count |
|---------------------|--------------|-------------|-------|
| `0102` | `03` | 0.4066 | 196 |
| `0101` | `01` | 0.3894 | 81 |
| `0110` | `01` | 0.3669 | 51 |
| `0120` | `23` | 0.3152 | 58 |
| `0121` | `23` | 0.2872 | 85 |

**Prefisso più comune:** `0102`
**Prefissi con transizione dominante (P>0.5):** 0 / 6

### Osservazioni MTM

- La matrice di transizione è **sparsa**: la maggior parte delle
  combinazioni prefisso→prossimo-evento non sono osservate.
- I prefissi a 3 nodi (codici come `0102`, `0110`, `0112`) presentano
  la più alta varietà di transizioni possibili.
- I prefissi a 2 nodi (es. `0101` = ripetizione arco) hanno transizioni
  più concentrate (alta probabilità su pochi pattern).
- `consecutive=YES` in MTM è il comportamento nativo: ogni prefisso viene
  consumato quando si estende, evitando il doppio conteggio.

---

## 2. Distribuzione Temporale e Granularità per EvoMine

**Statistiche inter-event time (IET) — 4,970 delta_t > 0:**

| Statistica | Valore (secondi) | Equivalente |
|------------|------------------|-------------|
| Media      | 270s | 0.1h |
| P25        | 18s | 0.0h |
| Mediana    | 47s | 0.0h |
| P75        | 123s | 0.0h |
| P95        | 629s | 0.2h |

**Granularità raccomandata per EvoMine:** sub-oraria (es. 15min = 900s o 30min = 1800s)

**Motivazione:**
- La mediana degli IET (47s = 0.0h) suggerisce
  che gli eventi si concentrano in finestre temporali relativamente strette.
- Con granularità sub-oraria (es. 15min = 900s o 30min = 1800s), il 99% degli IET
  ricade entro 1 bucket, mantenendo le sequenze temporali significative.
- La distribuzione IET appare con coda pesante (heavy tail),
  tipica delle reti sociali: pochi grandi gap temporali, molti eventi ravvicinati.

---

## 3. Osservazioni Preliminari per il Confronto MTM vs EvoMine

| Aspetto | MTM (Task 3) | EvoMine (prossimo step) |
|---------|-------------|------------------------|
| Dataset | CollegeMsg 5000 eventi | CollegeMsg 5000 eventi |
| Temporalità | Timestamp continui, DELTA=86400s | Bucket discreti (suggerito: sub-oraria, 1800s) |
| Output chiave | Matrice di transizione P(next\|prefix) | Regole GER: precondizione→postcondizione |
| Tipo pattern | Sequenze consecutive di archi | Sottografi che evolvono nel tempo |
| Consecutive | YES (nativo MTM) | N/A (EvoMine usa bucket) |
| Direzione archi | Diretti | Diretti (flag -d) |

### Prossimi passi

1. Eseguire EvoMine su CollegeMsg 5000 eventi (su Linux/Docker)
   con granularità temporale: sub-oraria (es. 15min = 900s o 30min = 1800s)
2. Confrontare i pattern trovati: MTM cattura sequenze temporali
   mentre EvoMine cattura regole evolutive di sottografi
3. Verificare se i motif ad alto enrichment (Task 1) corrispondono
   alle GER ad alto support trovate da EvoMine
