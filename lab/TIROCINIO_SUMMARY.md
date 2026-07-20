═══════════════════════════════════════
# TIROCINIO MAGISTRALE — Carlo Invernizzi
### "Profili evolutivi multilivello di reti complesse tramite temporal network motif"
═══════════════════════════════════════

> Documento di sintesi completo, generato per permettere a chiunque di ricostruire l'intero percorso del tirocinio senza dover rileggere codice, notebook o PDF originali. Ultimo aggiornamento dei dati: Task 5 (2026-07-xx), stato repository al 2026-07-20.

---

## CONTESTO GENERALE

### Obiettivo del tirocinio
Studiare l'evoluzione temporale di reti complesse (social network, messaggistica) a **più livelli di granularità** — locale (singolo nodo), mesoscopico (gruppi/pattern ricorrenti) e globale (intera rete) — usando due famiglie di strumenti complementari:

1. **Temporal network motif** (TMC/MTM, Liu & Sarıyüce, KDD 2023): contano pattern topologico-temporali ricorrenti in una finestra scorrevole e ne apprendono le probabilità di transizione, per generare reti sintetiche realistiche.
2. **Graph Evolution Rules** (EvoMine/GERM, Galdeman et al., IEEE DSAA 2023 / BDMA 2025): minano regole predittive "precondizione → postcondizione" con supporto statisticamente significativo, per descrivere come piccoli sottografi si trasformano nel tempo.

Il filo conduttore di tutto il lavoro (Task 3-5) è costruire un **ponte metodologico tra MTM ed EvoMine**: uno strumento misura probabilità condizionali locali su sequenze consecutive di archi, l'altro misura frequenza assoluta di trasformazioni precondizione→postcondizione su bucket temporali discreti. Capire dove questi due mondi si sovrappongono (e dove no, e perché) è la domanda di ricerca che attraversa i Task 3, 4 e 5.

### Supervisori
- **Relatore**: docente universitario Matteo Zignani
- **Correlatrice**: **Alessia Galdeman** (autrice del paper EvoMine/GER, ora ricercatrice post-dottorato all'IT University di Copenaghen; il toolkit GERANIO usato nei Task 2-5 è della sua repository). Le sue indicazioni hanno guidato correzioni metodologiche esplicite in più punti (vedi Task 3/4 "correzioni" e Task 5 "clock sequenziale vs binario").

### Dataset usati

**CollegeMsg** (`lab/task_1/task_1_SNAP/CollegeMsg.txt`, Stanford SNAP)
- Messaggi privati testuali tra studenti di un campus universitario (UC Irvine).
- Dataset completo: 59.798 (in altre note: 59.835) eventi, 1.899 nodi, 13.838 archi statici, span 193 giorni (2004-04-15 → 2004-10-26).
- Campione usato in quasi tutti i task successivi al Task 1: **primi 5000 eventi ordinati per timestamp** → 530 nodi, 2020 coppie src→dst uniche, periodo di soli **15.6 giorni** (2004-04-15 14:56:01 → 2004-05-01 04:08:19).
- IET (inter-event time) mediano: 47s → attività molto "a burst", tipica di una chat istantanea.

**Facebook Wall** (`lab/task_1/task_1_facebook/ia-facebook-wall-wosn-dir.edges`)
- Post pubblici su bacheca Facebook (regione New Orleans).
- Dataset completo: 264.004 (in altre note: 855.542) eventi, 45.813 nodi, 183.412 archi statici.
- Campione Task 1: 3.000 eventi, 2.797 nodi. Campione Task 2 (per EvoMine): primi 10.000 archi, 3.757 nodi, bucket settimanali.
- **Problema noto**: nel campione usato per Task 1, i timestamp risultano già "collassati" (span rilevato = 0 giorni), quindi l'analisi temporale su questo campione specifico non è affidabile — il TMC resta comunque valido perché usa solo differenze tra timestamp, non i valori assoluti.

**Differenze strutturali chiave tra i due dataset** (emerse fin dal Task 1):
| Aspetto | CollegeMsg | Facebook Wall |
|---|---|---|
| Tipo di interazione | Messaggio privato (dialogico) | Post su bacheca pubblica (broadcast) |
| Pattern dominante | Varietà: fan-in, fan-out, reciprocità, multi-edge | Hub puro (un mittente verso molti) |
| Enrichment massimo | 19.55× | 2.86× |
| Profilo EvoMine | Piatto/uniforme (nessuna regola dominante) | Legge di potenza (una regola pesa ~26%) |

Questa dicotomia (rete "dialogica densa e uniforme" vs rete "broadcast scale-free e concentrata") si ripresenta identica sia nell'analisi a motivi (Task 1) sia nell'analisi a regole evolutive (Task 2), confermando che le due tecniche colgono lo stesso fenomeno da angolazioni diverse.

### Background teorico studiato (da `documentazione/`)

**Paper 1 — "Using Motif Transitions for Temporal Graph Generation"** (Liu & Sarıyüce, KDD 2023) — riferimento per **Task 1**.
- Introduce il **Motif Transition Model (MTM)**: modella l'arrivo di nuovi eventi come processo stocastico di transizione tra motivi temporali, invece di contare esplicitamente i motivi (approccio precedente, costoso perché il conteggio scala esponenzialmente con la dimensione del motivo).
- Definisce formalmente il **motivo temporale** M^l = (V_j, E_j): sottografo debolmente connesso con l eventi, dove ogni evento condivide almeno un nodo con uno degli eventi precedenti e i timestamp sono strettamente crescenti.
- **Notazione a cifre**: un motivo a l eventi è rappresentato da 2l cifre, una coppia per arco; i nodi sono numerati in ordine di prima apparizione (0, 1, 2, ...). Esempio: `0110` = due eventi opposti tra due nodi (arco 0→1 poi 1→0).
- Spettro dei motivi: 6 tipi per M2, 60 tipi per M3, 888 tipi per M4.
- **Processo di transizione dei motivi**: T(M^l_i → M^l+1_j) — un motivo transita al successivo se arriva un nuovo evento adiacente entro la finestra δ, prima che scada il timeout o si raggiunga l_max.
- Cinque proprietà calcolate dal grafo di input: distribuzione di grado degli "eventi freddi" (K_CE), timestamp degli eventi freddi (T_CE), probabilità di transizione (P), tassi di transizione come processo di Poisson (Λ), numero medio di archi per processo di transizione (μ).
- L'algoritmo MTM (Algoritmo 1/2 del paper) è **O(|E|·|CE|·l_max/δ)** — non conta mai esplicitamente i motivi, il che lo rende fino a **391× più veloce** di TASBM e **84× più veloce** di STM sui dataset più grandi, mantenendo comunque errori (MSRE) fino a 10× inferiori nel preservare gli spettri di motivi temporali.
- Parametri chiave: `L_MAX` (max eventi per motivo), `DELTA`/`δ` (finestra temporale di transizione), `consecutive` (se un prefisso viene "consumato" o meno dopo l'estensione).

**Paper 2 — "Graph Evolution Rules Meet Communities"** (Galdeman, Zignani, Gaito — BDMA 2025, estensione del paper DSAA 2023 su cui si basa EvoMine) — riferimento per **Task 2-5**.
- Formalizza la **Graph Evolution Rule (GER)**: coppia di grafi r = (G_pre, G_post), dove G_pre (precondizione/body, t=0) si trasforma in G_post (postcondizione/head, t>0).
- **EvoMine** si basa sull'algoritmo **gSpan** (frequent subgraph mining tramite codice minimo DFS) applicato a una rappresentazione chiamata **grafo dell'unione** (union graph): una sequenza di grafi diretti G^T = (G1,...,GT) viene "appiattita" in un unico grafo dove le etichette di nodi/archi codificano l'intera storia temporale (es. l'etichetta `212` su un nodo significa: etichetta 2 al tempo 1, etichetta 1 al tempo 2, etichetta 2 al tempo 3).
- Tre vincoli topologici per una regola EvoMine valida: (1) i nodi di precondizione e postcondizione coincidono (nessuna aggiunta di nodi), (2) deve esserci almeno un cambiamento reale (E_pre ≠ E_post), (3) il grafo unione della regola deve essere connesso (la regola cattura un processo *localizzato*).
- **MIB support** (Minimum Image-Based, supporto basato su embedding) vs **supporto basato su eventi** (event-based): quest'ultimo — usato in tutto il lavoro di tirocinio — conta gli eventi di cambiamento (grafi-evento, costruiti dal vicinato del nodo coinvolto in un nuovo arco) in cui la regola compare, evitando l'overcounting di match sovrapposti.
- **Profilo GER**: rappresentazione vettoriale che riflette la distribuzione di supporto su tutte le regole trovate — permette di confrontare dataset diversi come vettori (analogo dell'enrichment/motif-spectrum di MTM).
- **Metodo euristico di scelta del supporto**: bilancia la "copertura" (percentuale di supporto totale catturata) contro il "numero di regole perse" aumentando la soglia — usato nel paper su reti (Sarafu, DBLP, Enron, Stack Overflow) e concettualmente ripreso nel Task 5 per la scelta del cutoff.
- Estensione mesoscopica (comunità via algoritmo di **Leiden**): dimostra che reti come Sarafu ed Enron hanno comunità con profili GER eterogenei (ogni comunità un proprio meccanismo evolutivo), mentre DBLP e Stack Overflow hanno comunità più omogenee; comunità vicine tendono a evolvere in modo simile. Questa parte "a comunità" del paper non è stata replicata nel tirocinio (che si è fermato al confronto globale MTM↔EvoMine), ma fornisce il framework teorico di riferimento per eventuali sviluppi futuri.

### Concetti chiave (glossario esteso in fondo al documento)
`temporal motif`, `GER`, `MTM`, `EvoMine`, `canonical coding`/`GERANIO`, `enrichment score`, `null model`, `MIB support`, `antimonotonicità` — vedi sezione **GLOSSARIO TECNICO** per le definizioni puntuali.

---

## TASK 1 — Temporal Motif Counting + MTM

### Obiettivo
Rispondere alla domanda: *quali pattern di interazione (motif) si ripetono nella rete, quanto sono frequenti, e sono statisticamente significativi rispetto a reti casuali?* Tradurre fedelmente in Python la repository C++ di riferimento (`erdemUB/KDD23-MTM`) e validarla su due dataset reali.

### Background studiato
Paper KDD 2023 (Liu & Sarıyüce) — vedi sezione CONTESTO GENERALE. Concetti applicati: motivo temporale, codifica a cifre, processo di transizione, null model per la significatività statistica.

### Dati usati e modifiche
- **facebook_wall**: campione 3.000 eventi, 2.797 nodi, `L_MAX=4`, `DELTA=86400s`.
- **CollegeMsg**: campione 5.000 eventi, 530 nodi, `L_MAX=3`, `DELTA=86400s` (L_MAX ridotto rispetto a facebook per contenere il costo computazionale, che scala esponenzialmente con L_MAX).
- Timestamp facebook_wall: rilevato bug/anomalia nel campione (span=0 giorni) — l'analisi topologica resta valida, quella temporale no.

### Funzioni implementate (con spiegazione di cosa fa ognuna)
Tradotte 1:1 dal C++ originale, in due notebook (`task1_facebook.ipynb`, `task1_snap.ipynb`):

| Funzione | Input | Output | Cosa fa |
|---|---|---|---|
| `create_events(filename)` | path del file edge-list `(u, v, t)` | `(events, V)`: lista di eventi ordinati per timestamp, set dei nodi | Legge il dataset, rimuove self-loop e duplicati esatti, ordina per tempo |
| `encode_motif(instance)` | lista di eventi `(t,(u,v))` che formano un'istanza di motivo | stringa di cifre (es. `"0102"`) | Rinomina i nodi in ordine di prima apparizione e concatena le coppie (src,dst); due istanze hanno lo stesso codice sse isomorfe (stessa topologia e stesso ordine temporale) |
| `get_nodes(events)` | lista di eventi | set di nodi coinvolti | Utility per `count_mtp`/`count_instance` |
| `count_instance(e, imap, keys, N_event, d_c, consecutive)` | nuovo evento `e`, instance map `imap`, set di chiavi attive, `L_MAX`, `DELTA`, flag consecutive | set di chiavi aggiornato (modifica `imap` in place) | TMC: estende ogni prefisso attivo che condivide un nodo con `e` ed è entro `delta`; se `consecutive=YES` il prefisso "consumato" viene rimosso dal set attivo (evita doppio conteggio) |
| `run_tmc(input_file, max_event, max_memory, consecutive, output_file, verbose)` | path input, `L_MAX`, `DELTA`, flag consecutive | dizionario `codice_motivo → conteggio` | Pipeline TMC completa: legge eventi, applica `count_instance` a ogni evento, aggrega i conteggi finali |
| `count_mtp(e, MC, prefixes, N_event, d_c, initial_E)` | nuovo evento, dizionario conteggi motivi `MC`, set prefissi attivi, `L_MAX`, `DELTA`, lista eventi iniziali | set di prefissi aggiornato | Per MTM: come `count_instance` ma tiene traccia anche dei tempi di transizione (Δt) per ogni transizione osservata, necessari per il tasso di Poisson |
| `count_transition(MC, initial_size)` | dizionario motivi→(count,tempi), numero eventi iniziali | dizionario `TR`: `prefisso → lista di (estensione, count)` incluso stato `'S'` (stop) | Costruisce la tabella di transizione P(M_{l+1}\|M_l), aggiungendo lo stato terminale S per i prefissi che non si estendono più |
| `randomize_ie(initial_E, V)` | lista eventi iniziali, set nodi | lista eventi iniziali randomizzati | Riassegna casualmente i nodi degli eventi iniziali preservando la degree sequence (config model), per la fase di generazione |
| `generate_graph(TR, IE, V, Max_event, K, graph_size, new_ratio)` | tabella di transizione, eventi iniziali randomizzati, nodi, `L_MAX`, numero di grafi da generare `K`, dimensione target, rapporto nuovi archi | lista di eventi sintetici | Per ogni evento iniziale, segue le transizioni con probabilità `P` campionando via `random.choices`, campiona Δt da distribuzione esponenziale (tasso λ), decide se riusare un arco esistente o crearne uno nuovo (Eq. 1 del paper) |
| `count_size(events)` | lista eventi | numero di archi unici (non orientati) | Utility per calibrare `generate_graph` |
| `degree_dist(events)` | lista eventi | lista ordinata `(grado, #nodi)` | Calcola la distribuzione di grado (in+out) per il confronto originale vs sintetico |
| `time_shuffle(events, seed)` | eventi, seed | eventi con timestamp rimescolati | Null model 1: preserva la topologia, distrugge la correlazione temporale |
| `edge_shuffle(events, V, seed)` | eventi, nodi, seed | eventi con endpoint rimescolati (config model) | Null model 2: preserva timestamp e degree sequence, distrugge la topologia (chi parla con chi) |
| `strogatz_shuffle(events, V, k_neighbors, p_rewire, seed)` | eventi, nodi, parametri Watts-Strogatz | eventi completamente sintetici su topologia ring-lattice + rewiring | Null model 3: preserva solo # nodi e distribuzione temporale campionata con reinserimento |
| `parse_motif_code(code)` | stringa codice | lista di archi `(t, u, v)` | Decodifica per la visualizzazione grafica |
| `motif_description(code)`, `get_layout(nodes, edges)`, `draw_motif(...)`, `plot_motif_gallery(...)` | codice/lista di codici | figure matplotlib | Pipeline di visualizzazione delle gallery di motivi (layout a stella se c'è un hub, altrimenti spring layout; colore delle frecce = ordine temporale) |

### Parametri usati (con motivazione)
| Parametro | facebook_wall | CollegeMsg | Motivazione |
|---|---|---|---|
| `L_MAX` | 4 | 3 | 3 per CollegeMsg per contenere il costo esponenziale del TMC |
| `DELTA` | 86400s (1 giorno) | 86400s | Finestra naturale per reti sociali: interazioni correlate nella stessa giornata |
| `K` (grafi generati) | 1 | 1 | Sufficiente per confronto qualitativo |
| `consecutive` | NO | NO | Permette di contare tutte le istanze, non solo quelle che partono da un evento iniziale distinto |
| `N_SAMPLE` | 3.000 | 5.000 | Bilancia tempo di calcolo e copertura |
| `k_neighbors` / `p_rewire` (Watts-Strogatz) | 4 / 0.1 | 4 / 0.1 | Standard per ring lattice + 10% di rewiring |

### Output prodotti (file e grafici)
- `task_1_facebook/output_data/`: campione (`facebook_sample_3000.txt`), motivi con conteggio (`facebook_wall_motifs_3000_4_86400.txt`, 232 tipi), grafo sintetico MTM (`facebook_wall_generated_3000_4_86400.txt`, 2976 eventi), 3 null model.
- `task_1_SNAP/output_data/`: campione (`collegemsg_sample_5000.txt`), motivi (`CollegeMsg_motifs_5000_3_86400.txt`, 66 tipi), grafo sintetico MTM (5419 eventi), 3 null model.
- 8 grafici per dataset: top-20 motivi, distribuzione per lunghezza, enrichment, confronto null model, degree distribution orig-vs-MTM, distribuzione temporale, gallery top-12, gallery top-8 enriched.

### Risultati con numeri esatti

**facebook_wall**: 264.004 eventi totali / 45.813 nodi / 183.412 archi statici nel dataset completo; campione 3.000 eventi, 2.797 nodi. TMC trova **232 tipi di motivo**. Il motivo dominante `01020304` (hub puro a 4 archi, fan-out) ha **3.678.598** occorrenze — circa 10× il secondo classificato (`010203`, 341.911). Enrichment massimo **2.86×** per l'hub puro; 13 motivi su 20 hanno enrichment > 1. Motivi con **reciprocità** (`01101213`, `01021013`) hanno enrichment 2.57× e 2.41×, indicando che le conversazioni bilaterali seguite da espansione sono un pattern reale e non casuale. Tempo TMC: ~437s. Grafo MTM generato: 2.976 eventi (rapporto eventi=0.992, archi=0.999, nodi=1.000 — qualità eccellente).

**CollegeMsg**: 59.798 eventi totali / 1.899 nodi / 13.838 archi statici nel dataset completo (span 193 giorni); campione 5.000 eventi, 530 nodi. TMC trova **66 tipi di motivo** (con L_MAX=3). Distribuzione molto più piatta di facebook: rapporto 1°/20° classificato ≈ 6:1 (contro ~500:1 di facebook). **Tutti i 30 motivi analizzati hanno enrichment > 1** (contro 13/20 di facebook), con massimo **19.55×** per il motivo `010121` (0→1 due volte + 2→1, cioè messaggi ripetuti allo stesso destinatario seguiti da un terzo mittente). Tempo TMC: ~244s. Grafo MTM generato: 5.419 eventi (rapporto eventi=1.084, archi=0.924, nodi=0.949 — qualità buona ma inferiore a facebook).

Tempi dei null model: facebook — Time Shuffle 429s, Edge Shuffle 708s (il più lento, per la maggiore varietà strutturale generata: 377 tipi di motivo vs 232 reali), Strogatz 5s; CollegeMsg — Time Shuffle 42s, Edge Shuffle 102s, Strogatz 4s.

### Osservazioni chiave
- Il fenomeno dei **multi-edge temporali** (stesso arco ripetuto entro la finestra δ) è altamente caratteristico di CollegeMsg (enrichment fino a 19.55×) e assente in facebook: riflette la messaggistica istantanea (raffiche di messaggi allo stesso utente) vs il posting singolo su bacheca.
- facebook_wall è dominato da **hub broadcast** (un mittente centrale verso molti destinatari); CollegeMsg mostra equilibrio tra fan-out, fan-in, reciprocità, catene — riflette rispettivamente comunicazione broadcast vs dialogica.
- Il modello MTM preserva la degree distribution quasi perfettamente per facebook (curve log-log sovrapposte); per CollegeMsg genera più nodi a basso grado e alcuni hub leggermente più grandi del reale.
- Il problema di timestamp collassati nel campione facebook_wall (span=0) non invalida il TMC (basato solo su differenze temporali) ma invalida l'analisi della distribuzione temporale su quel campione specifico.
- Questo task fornisce la base concettuale per la lettura multilivello della tesi: motivo locale (comportamento del nodo) → transizione tra motivi (livello mesoscopico) → enrichment/distribuzione (livello globale).

---

## TASK 2 — Graph Evolution Rules con EvoMine

### Obiettivo
Minare **Graph Evolution Rules** sugli stessi due dataset del Task 1, rispondendo alla domanda: *dato un certo sottografo osservato al tempo t, quali nuovi archi appariranno con alta probabilità in un momento successivo?*

### Background studiato
Paper Galdeman et al., IEEE DSAA 2023 (versione precedente all'estensione BDMA 2025 letta per intero nel contesto generale). Toolkit **GERANIO** (repository della correlatrice), che include due algoritmi:
- **GERM**: grafi non diretti, solo inserimento archi.
- **EvoMine**: estende GERM a grafi diretti, rimozione archi, grafi con label — usato in questo lavoro.

### Dati usati e modifiche
- **CollegeMsg**: dataset completo convertito in formato GER (1.899 nodi, 59.835 archi, bucket giornalieri, 194 bucket).
- **Facebook Wall**: campione dei primi 10.000 archi (3.757 nodi), bucket settimanali (82 bucket).
- **DBLP 2003-05**: output GERM precomputed già presente nella repo GERANIO, usato per validare la pipeline di analisi prima di eseguire EvoMine sui dataset propri.

### Pipeline implementata
```
Dataset grezzo (u,v,t)
  → discretizzazione timestamp in bucket (from_txtfile_to_gerinput)
  → formato GER: "t # 0 / v <id> 0 / e <src> <dst> <ts_bucket>"
  → EvoMine binary (via Docker, Linux ELF)
  → parsing output (from_ger_output, obtain_pattern_list)
  → mapping canonico cross-dataset (mapping_pattern_ids)
  → profilo evolutivo (get_profiles)
  → visualizzazione (draw_rule, draw_several_patterns, plot_profile, plot_heatmap, t_span_plot)
```

Funzioni chiave del toolkit GERANIO (`task_2/EvoMine/*.py`):
| Funzione | Input | Output | Cosa fa |
|---|---|---|---|
| `file_converters.from_txtfile_to_gerinput(...)` | edge list grezza | file in formato GER | Converte la lista di archi nel formato richiesto da EvoMine, discretizzando i timestamp in bucket interi progressivi |
| `file_converters.from_ger_output(output_file)` | file di output del binario | `(info_list, patterns, support_patterns, mapping)` | Parsa i blocchi `t # <id> <support>` del file grezzo prodotto da EvoMine |
| `file_converters.obtain_pattern_list(patterns, support_patterns, algorithm)` | pattern grezzi e supporti | `(pattern_list, support)` | Separa archi con timestamp 0 (precondizione) da quelli con timestamp >0 (postcondizione); qui è definito il mapping critico `mapping_ts = {3: 0, 1: 1}` (label arco 3 = body/t=0, label 1 = head/t>0) |
| `general_mapping.mapping_pattern_ids(algo, patterns, support, general_mapping, mapped_patterns_path, directed)` | pattern di un run, mapping generale esistente | pattern con ID coerenti tra run diversi | Assegna ID stabili ai pattern isomorfi, per confrontare dataset diversi |
| `analysis.get_profiles(support_file, all_patterns_id)` | supporti e ID totali | vettore "profilo" (frequenze relative) | Rappresentazione vettoriale della rete, analoga all'enrichment del Task 1 |
| `analysis.plot_profile / plot_heatmap / t_span_plot` | profili | grafici | Visualizzazione profilo singolo, confronto multi-dataset, distribuzione del t-span (distanza temporale massima tra archi di una regola) |

### Parametri usati (con motivazione)
| Parametro | Valore usato | Significato |
|---|---|---|
| `-s` (support) | 20.000 (CollegeMsg), 500 (Facebook) | Soglia minima di frequenza; valori diversi per compensare la diversa densità/dimensione dei due dataset |
| `-e` (max_edges) | 3 | Numero massimo di archi nella postcondizione |
| `-T full` | full | MIB support completo |
| `-t`, `-d` | flag attivi | Modalità temporale e grafo diretto |

### Criticità tecniche incontrate e soluzioni
1. **Incompatibilità binari**: `evomine`/`germ` sono Linux x86-64 ELF, non eseguibili nativamente su macOS ARM64 (Apple Silicon) → risolto con **Docker Desktop**, immagine `ubuntu:22.04`, montando la cartella del progetto come volume.
2. **Discretizzazione timestamp** (criticità più grave): EvoMine crea uno snapshot del grafo per ogni timestamp *unico*. Con Unix epoch grezzo (es. 1082040961) questo genera decine di migliaia di snapshot → out-of-memory (exit 137). **Soluzione**: convertire i timestamp in datetime, troncare al giorno (o alla settimana per Facebook), assegnare `t=0,1,2,...` progressivo per ciascuna data distinta — così il numero di snapshot eguaglia il numero di *giorni* distinti, non di secondi.
3. **Soglia di supporto per CollegeMsg**: con support=1000 o 5000 il mining si bloccava per ore su singoli pattern candidati (enumerazione di sottografi è NP-hard nel caso peggiore). Support=20000 ha permesso risultati in pochi secondi, al costo di trovare solo le regole più frequenti.

### Output prodotti
- `output_data/college_evomine_20000_3.txt`, `facebook_evomine_500_3.txt` (output grezzi EvoMine).
- `output_png/`: `CollegeMsg_profile.png`, `Facebook_profile.png`, `comparison_heatmap.png`, più le figure di validazione su DBLP (`dblp_all_patterns_gallery.png`, `dblp_evolutionary_profile.png`, `dblp_pattern_support_dist.png`, `dblp_tspan.png`, `example_rule_germ.png`, `temporal_dist_both.png`).

### Risultati con numeri esatti

**Validazione su DBLP 2003-05** (output precomputed GERM, support=5000, max_edges=3): **20 regole** totali, supporto tra 5.054 e 109.044 (mediana 10.297, somma totale 529.964). Le top-5 regole pesano ~74% del supporto totale. 16/20 regole hanno 3 archi totali (corpo+testa). Distribuzione t-span: 5 regole con t-span=0 (pattern statici, solo precondizione — es. singolo arco, catena a 2, triangolo, catena a 3, stella), 8 con t-span=1 (evoluzione immediata), 7 con t-span=2.

**CollegeMsg** (support=20000, max_edges=3, bucket giornalieri): **14 regole trovate**, supporto tra 147.998 e 214.115 (mediana 181.792, somma 2.535.031). Profilo **piatto**: frequenza relativa tra 0.058 e 0.085, nessuna regola dominante.

**Facebook Wall** (support=500, max_edges=3, bucket settimanali, campione 10.000 archi): **27 regole trovate**, supporto tra 510 e 43.467 (mediana 4.225, somma 167.433). Profilo a **legge di potenza**: la regola 0 pesa da sola ~26% del totale; la regola più frequente ha supporto ~85× maggiore dell'ultima.

Nell'heatmap comparativa, CollegeMsg mostra intensità quasi uniforme (~0.07 per colonna 0-13, poi zero perché ha solo 14 regole), Facebook mostra concentrazione estrema (colonne 0-1 in blu scuro, poi quasi zero — coda lunga scale-free).

### Osservazioni chiave
- EvoMine mina regole **causali** (precondizione→postcondizione), non solo co-occorrenze come TMC/MTM — le due tecniche rispondono a domande diverse ma complementari: "quali pattern esistono ed sono significativi?" (MTM) vs "dato un pattern, cosa accade dopo?" (EvoMine).
- I due dataset hanno dinamiche evolutive opposte, coerentemente col Task 1: CollegeMsg uniforme, Facebook concentrata/scale-free.
- Limitazione pratica: soglie di supporto troppo basse su hardware Docker/macOS portano a run di ore; questo limite tecnico è stato uno dei driver della scelta dei parametri nei Task successivi (in particolare la scelta di ridurre il campione a 5000 eventi anziché usare il dataset completo).

---

## TASK 3 — Allineamento MTM ↔ EvoMine (parte 1)

### Obiettivo
Rendere MTM ed EvoMine confrontabili sullo **stesso campione** (CollegeMsg, 5000 eventi): stessa granularità temporale, stesso concetto di "prefisso/precondizione → estensione/postcondizione".

### Dati usati e modifiche
Campione `collegemsg_sample_5000.txt`: primi 5000 eventi ordinati per timestamp del dataset CollegeMsg completo → 530 nodi, periodo di **15.6 giorni** (durata esatta: 1.343.538s), dal 2004-04-15 14:56:01 al 2004-05-01 04:08:19.

### Funzioni implementate
Stesse funzioni MTM del Task 1 (`create_events`, `encode_motif`, `count_mtp`, `count_transition`), riusate con `consecutive=YES` (comportamento nativo MTM: ogni prefisso viene consumato quando si estende) e applicate sia alla costruzione della transition matrix sia all'analisi IET. Nuova analisi: distribuzione oraria e inter-event time (IET), con calcolo di CCDF empirica in scala log-log.

### Parametri usati
- MTM: `L_MAX=3` (allineato ai benchmark di scalabilità Task 4 e alla run originale su CollegeMsg del Task 1 — vedi nota sotto), `DELTA` testato sia a 1800s (30 min) sia a 3600s (1h), `consecutive=YES`, `N=5000`.
- **Nota metodologica importante**: la prima versione della matrice usava `L_MAX=4`; è stata poi ricalcolata con `L_MAX=3` per allinearla ai parametri usati altrove nel progetto — le due versioni sono entrambe documentate (vedi risultati sotto).

### Output prodotti
- `output_data/`: `mtm_transition_matrix.csv` (originale L_MAX=4/86400s), `mtm_transition_matrix_L3_30min.csv`, `mtm_transition_matrix_L3_1h.csv` (riallineate), `inter_event_stats.txt`, `granularity_summary.txt`, 5 file di preprocessing GER (`collegemsg_ger_{30min,1h,6h,12h,1d}.txt`), 2 output grezzi EvoMine (`collegemsg_evomine_{30min,1h}_s1_raw.txt`), `evomine_30min_timing.txt`.
- `output_png/`: 3 heatmap della transition matrix (originale, L3-30min, L3-1h), `inter_event_time_dist.png`, `collegemsg_hourly_distribution.png`, `collegemsg_iet_distribution.png`.
- `report_task3_completo.pdf/docx`.

### Risultati con numeri esatti

**Transition matrix MTM — tutte e tre le versioni** (dimensione sempre 6 prefissi × 12 terzi-eventi; probabilità condizionata sul prefisso: `P(terzo evento|prefisso) = count(prefisso→terzo evento) / count(prefisso→qualsiasi evento)`):

| Versione | L_MAX | DELTA | Celle osservate | Top transizione | P |
|---|---|---|---|---|---|
| Originale | 4 | 86400s (1g) | 53/72 (73.6%) | `0102→03` | 0.4066 (count 196) |
| Riallineata 30min | 3 | 1800s | 53/72 (73.6%) | `0101→01` | 0.4467 (count 109) |
| Riallineata 1h | 3 | 3600s | 54/72 (75.0%) | `0101→01` | 0.4508 (count 110) |

Top-5 transizioni originali (L_MAX=4): `0102→03` P=0.4066(196), `0101→01` P=0.3894(81), `0110→01` P=0.3669(51), `0120→23` P=0.3152(58), `0121→23` P=0.2872(85).
Top-5 a 30min (L_MAX=3): `0101→01` P=0.4467(109), `0102→03` P=0.4269(225), `0110→01` P=0.4118(63), `0120→23` P=0.2727(42), `0121→23` P=0.2314(59).
Top-5 a 1h (L_MAX=3): `0101→01` P=0.4508(110), `0110→01` P=0.3910(61), `0102→03` P=0.3752(212), `0120→23` P=0.2841(50), `0121→23` P=0.2310(64).

Confronto tra versioni: stessa forma (6×12) e stesso insieme di prefissi dominanti in tutte e tre. Differenza numerica media assoluta rispetto all'originale: **0.021** (max 0.088) per 30min, **0.018** (max 0.061) per 1h, su 53 celle confrontabili. **Unica differenza qualitativa**: con L_MAX=4 la transizione dominante è `0102→03` (star verso nodo nuovo, broadcast); con L_MAX=3 diventa `0101→01` (ripetizione dello stesso arco, conversazione ripetuta tra la stessa coppia) — `0102→03` resta comunque tra le prime 2-3.

**Analisi IET completa** (4.970 campioni con delta_t>0, esclusi 29 con delta_t=0):
| Statistica | Valore |
|---|---|
| Media | 270.3s |
| Std dev | 4132.9s |
| P25 | 18.0s |
| Mediana | 47.0s |
| P75 | 123.0s |
| P95 | 629.1s |
| P99 | 2122.7s |
| Max | 258.552s (2.99 giorni) |

Distribuzione oraria: 375 ore nel periodo, media 13.3 eventi/ora, mediana 2.0, max 151; 159/375 ore (42.4%) senza eventi — attività molto discontinua.

**Tabella granularità candidate**:
| Granularità | Secondi | % IET entro 1 bucket | # bucket totali |
|---|---|---|---|
| 30 minuti | 1800 | 98.6% | 747 |
| 1 ora | 3600 | 99.4% | 374 |
| 6 ore | 21600 | 99.9% | 63 |
| 12 ore | 43200 | 100.0% | 32 |
| 1 giorno | 86400 | 100.0% | 16 |

Preprocessing GER per tutte e 5 le granularità (530 nodi in tutti i casi): a 30min si hanno 747 bucket totali (388 non vuoti, 359 vuoti, media 12.89 eventi/bucket non vuoto, max 97); a 1h 374 bucket (216 non vuoti, media 23.15, max 157); a 6h 63 bucket (48 non vuoti, media 104.17, max 444); a 12h 32 bucket (26 non vuoti, media 192.31, max 793); a 1 giorno 16 bucket (14 non vuoti, media 357.14, max 1006).

Granularità candidate finali selezionate per EvoMine: **30 minuti** e **1 ora** — entrambe testate in parallelo nel Task 4, poiché la scelta finale è demandata alla correlatrice.

### Osservazioni chiave
- La mediana IET di 47s indica un comportamento fortemente "a burst" (messaggistica istantanea): giustifica una granularità sub-oraria per EvoMine, altrimenti sequenze temporalmente vicine finirebbero schiacciate/separate in modo non realistico.
- La CCDF dell'IET si azzera già ben prima della soglia di 1 ora — coerente con l'heavy-tail osservato anche nel Task 1 (enrichment dei motivi multi-edge).
- Il cambio L_MAX=4→3 ha un effetto misurabile ma non strutturale: la matrice resta sparsa, la forma resta identica, cambia solo quale transizione risulta "in testa".

---

## TASK 4 — Scalabilità e Confronto MTM ↔ EvoMine

### Obiettivo
Misurare i tempi di esecuzione di TMC/MTM al variare di dimensione del campione (N) e granularità (DELTA); eseguire EvoMine con `support=1` (nessun filtro di frequenza, tutte le regole possibili); confrontare MTM ed EvoMine con una codifica canonica comune; analizzare l'evoluzione strutturale del dataset nel tempo.

### Dati usati
Stesso campione CollegeMsg 5000 eventi (530 nodi, 2020 coppie src→dst uniche) del Task 3, con sotto-campioni aggiuntivi per il benchmark di scalabilità (`sample_{500,1000,2000,3000,5000}.txt`).

### Funzioni implementate
Riuso delle funzioni MTM/TMC del Task 1 per i benchmark, più:
- Wrapper di timing attorno a `run_tmc` per variare N e DELTA sistematicamente.
- `encode_motif` riapplicata alle regole EvoMine (non solo ai motivi MTM) per produrre una codifica comune tra i due strumenti.
- Nuove funzioni per l'analisi di evoluzione strutturale: conteggio archi/nodi attivi per bucket, nodi nuovi cumulativi, calcolo del "novelty rate" (frazione di archi per bucket che coinvolge almeno un nodo mai visto prima).

### Parametri usati (con motivazione)
- Benchmark N: `{500,1000,2000,3000,5000}`, `DELTA=86400s`, `L_MAX=3`, `consecutive=YES`.
- Benchmark DELTA: `{1800,3600,21600,43200,86400}s`, `N=5000`, `L_MAX=3`.
- EvoMine: `support=1`, `max_edges=3`, `-T full`, granularità **30min e 1h in parallelo** (non ancora deciso quale sia definitiva).
- Raccomandazione finale scelta: `N=5000` (intero campione Task 3, tempo trascurabile), `DELTA=1800s` (30 min, coerente con l'analisi IET del Task 3).

### Output prodotti
- `output_data/`: `tmc_scalability.csv`, `tmc_delta_scalability.csv`, `scalability_summary.txt`, `execution_times.csv`, `evomine_{30min,1h}_coded_rules.csv` (+ storico `evomine_coded_rules.csv`), `mtm_evomine_{30min,1h}_comparison.csv` (+ storico), `structural_evolution_{30min,1h}.csv`, `structural_evolution_summary.txt`.
- `output_png/`: `execution_times.png`, `tmc_scalability.png`, `tmc_delta_scalability.png`, `evomine_granularity_comparison.png`, `structural_evolution_{30min,1h}.png`.
- `report_task4_completo.pdf/docx`.

### Risultati con numeri esatti

**Tabella tempi completa Task 1-4**:
| Task | Algoritmo | Dataset | N eventi | Parametri | Tempo (s) |
|---|---|---|---|---|---|
| 1 | TMC | facebook_wall | 3000 | L_MAX=4, δ=86400s | 437 |
| 1 | TMC | CollegeMsg | 5000 | L_MAX=3, δ=86400s | 244 |
| 1 | TMC null (Time Shuffle) | facebook_wall | 3000 | L_MAX=4 | 429 |
| 1 | TMC null (Edge Shuffle) | facebook_wall | 3000 | L_MAX=4 | 708 |
| 1 | TMC null (Strogatz) | facebook_wall | 3000 | L_MAX=4 | 5 |
| 1 | TMC null (Time Shuffle) | CollegeMsg | 5000 | L_MAX=3 | 42 |
| 1 | TMC null (Edge Shuffle) | CollegeMsg | 5000 | L_MAX=3 | 102 |
| 1 | TMC null (Strogatz) | CollegeMsg | 5000 | L_MAX=3 | 4 |
| 2 | EvoMine | CollegeMsg | 59835 | s=20000, e=3, T=full | pochi secondi |
| 3 | TMC benchmark | CollegeMsg | 5000 | L_MAX=3, δ=86400s | 0.071 |
| 4 | EvoMine | CollegeMsg | 5000 | s=1, e=3, T=full, δ=3600s (1h) | 97 |
| 4 | EvoMine | CollegeMsg | 5000 | s=1, e=3, T=full, δ=1800s (30min) | 197 |

**Benchmark TMC per N** (DELTA=86400s, L_MAX=3): N=500→0.004066s (57 motivi, 929 istanze); N=1000→0.007975s (59, 1917); N=2000→0.033599s (59, 3872); N=3000→0.031803s (59, 5830); N=5000→0.060946s (60, 9771). Scaling superlineare ma sempre trascurabile (<0.1s).

**Benchmark TMC per DELTA** (N=5000, L_MAX=3): 1800s→0.027102s (60 motivi, 7519 istanze); 3600s→0.048604s (60, 8112); 21600s→0.048285s (60, 9241); 43200s→0.070354s (60, 9593); 86400s→0.060983s (60, 9771). **I motivi distinti restano costanti a 60 per qualsiasi DELTA** — cambia solo il numero di istanze.

**Run EvoMine entrambe le granularità**: 30min (dataset `collegemsg_ger_30min.txt`, 747 bucket, 388 non vuoti) → **197s**, **62 regole GER trovate**; 1h (dataset `collegemsg_ger_1h.txt`, 374 bucket, 216 non vuoti) → **97s** (+103% per la versione 30min), **62 regole GER trovate** — stesso numero esatto a entrambe le granularità.

**Confronto MTM↔EvoMine** (codifica canonica `encode_motif` comune, confrontato contro le matrici MTM L_MAX=3 riallineate del Task 3): **62 regole codificate** per granularità, **12 strutturalmente comparabili con MTM** (esattamente 2 archi body + 1 arco head), **10 su 12 trovate anche in MTM** — risultato identico a entrambe le granularità. Le 52 regole rimanenti sono per lo più senza precondizione (body vuoto).

Le 10 regole comparabili (supporto e probabilità MTM affiancati per le due granularità):
| Codice regola | Supp. 30min | Prob. MTM 30min | Supp. 1h | Prob. MTM 1h |
|---|---|---|---|---|
| 0112→03 | 1764 | 0.0513 | 1466 | 0.0632 |
| 0102→03 | 1622 | 0.4269 | 1341 | 0.3752 |
| 0112→30 | 1609 | 0.0385 | 1359 | 0.0316 |
| 0110→20 | 1463 | 0.1895 | 1224 | 0.1667 |
| 0102→30 | 1397 | 0.0285 | 1178 | 0.0372 |
| 0110→02 | 852 | 0.0458 | 708 | 0.0385 |
| 0120→10 | 397 | 0.1364 | 326 | 0.1250 |
| 0102→10 | 390 | 0.0266 | 318 | 0.0248 |
| 0121→10 | 388 | 0.0863 | 318 | 0.0722 |
| 0112→10 | 365 | 0.1026 | 297 | 0.1053 |

Osservazione: `0102→03` è la transizione **dominante in MTM** (P=0.43 a 30min, P=0.38 a 1h) ma ha supporto EvoMine solo **moderato** (1622/1341): le regole a supporto assoluto più alto in EvoMine sono tutte body-only (es. `01→`, supporto >30.000), strutturalmente non comparabili con la matrice MTM. Conferma diretta della complementarità: MTM misura probabilità condizionale normalizzata sul prefisso, EvoMine misura frequenza assoluta grezza.

**Evoluzione strutturale del dataset** (530 nodi unici, 2020 coppie src→dst uniche — identiche a entrambe le granularità, come atteso, essendo proprietà del dataset non della discretizzazione):
- 30min (747 bucket): 50% dei nodi unici raggiunto al bucket 510 (68% del periodo); 90% al bucket 705 (94% del periodo); picco di attività al bucket 707 (97 archi, ovvero al 94.6% del periodo osservato); novelty rate medio **62.1%**.
- 1h (374 bucket): 50% dei nodi al bucket 255 (68% del periodo); 90% al bucket 352 (94% del periodo); picco al bucket 353 (157 archi, 94.4% del periodo); novelty rate medio **67.1%**.
- La rete raggiunge il 50% dei nodi unici molto presto in termini di *eventi* (evento 1315/5000 = 26% degli eventi) ma solo al 68% del *tempo* trascorso: i primi giorni hanno pochi eventi ma già alta copertura di nodi, seguiti da un lungo periodo a bassa attività.
- Il novelty rate parte altissimo nei primi bucket attivi (93-100%) e scende fino a ~48% negli ultimi bucket attivi: il burst finale riattiva prevalentemente coppie di nodi già note, non introduce molte connessioni nuove.

### Osservazioni chiave
- TMC scala benissimo fino a N=5000 (max 71-76ms) — nessun collo di bottiglia, si può usare l'intero campione.
- Il numero di motivi distinti è **indipendente da DELTA** — solo il numero di istanze cresce; questo suggerisce che la scelta della finestra temporale per MTM non è critica per la topologia osservata, solo per la sua frequenza.
- EvoMine con support=1 è sostenibile (97-197s) su un singolo run a N=5000, ma quasi raddoppia passando da 1h a 30min per il maggior numero di bucket da processare.
- La dinamica "crescita rapida iniziale + burst finale a bassa novità" rafforza la scelta di una granularità fine (30min/1h): con bucket più larghi (es. 1 giorno) il burst finale e il periodo di stasi verrebbero schiacciati nello stesso bucket, perdendo il segnale.

### Correzioni apportate (rispetto alla versione iniziale)
Tre disallineamenti identificati e corretti nell'iterazione del 2026-07-08:
1. **Granularità EvoMine non allineata alla raccomandazione IET** (EvoMine testato solo a 1h mentre l'analisi IET del Task 3 indicava 30min come opzione preferibile) → risolto testando **sistematicamente entrambe** le granularità in ogni fase successiva.
2. **L_MAX disallineato tra MTM (L_MAX=4) e i benchmark di scalabilità TMC (L_MAX=3)** → la transition matrix MTM è stata ricalcolata con L_MAX=3 (Task 3, Sezione 3) per essere comparabile.
3. **Assenza di un'analisi di evoluzione strutturale del dataset** → aggiunta come nuova sezione (Sezione 7 del report Task 4).
Inoltre: il confronto MTM↔EvoMine è stato rifatto contro le matrici L_MAX=3 riallineate (non più contro la versione originale L_MAX=4/86400s), per confrontare grandezze coerenti tra loro (fix documentato come "Fix 2" nel report).

---

## TASK 5 — Forma canonica GERANIO e Transition Matrix EvoMine

### Obiettivo
Ripetere il confronto MTM↔EvoMine del Task 3/4 usando una **codifica canonica condivisa** diversa da `encode_motif` (chiamata "GERANIO", confermata dalla correlatrice): rinomina dei nodi per ordine di prima apparizione + ordinamento degli archi per `(t_logico, src, dst)` in una stringa `src_dst_t-src_dst_t-...`, applicata identicamente sia ai motivi MTM sia alle regole EvoMine, per verificare quante celle delle rispettive transition matrix coincidono esattamente.

### Background studiato
Stesso paper EvoMine (Galdeman et al.) del Task 2; in aggiunta, lettura mirata di `task_2/EvoMine/file_converters.py` per verificare la convenzione `mapping_ts = {3: 0, 1: 1}` (arco con label 3 = precondizione/body a t=0, label 1 = postcondizione/head a t=1) usata nel parsing delle regole EvoMine grezze.

### Dati usati
CollegeMsg 5000 eventi, granularità **1 ora** (support=1) — stessi input dei Task 3/4: `task_3/output_data/collegemsg_sample_5000.txt` (per MTM) e `task_3/output_data/collegemsg_evomine_1h_s1_raw.txt` (per EvoMine, 62 regole grezze).

### Funzioni implementate (con spiegazione dettagliata)

**1. Canonical coding schema semplice** — `canonical_form(edges_with_t)`:
- Input: lista di tuple `(u, v, t)` (archi con etichetta temporale, non necessariamente ordinati).
- Output: lista di tuple canoniche `(u', v', t)` con nodi rinominati progressivamente `"0","1","2",...` in ordine di prima apparizione, ordinate per `(t, src, dst)`.
- `tuples_to_str(tuples)` converte il risultato in stringa leggibile `u_v_t-u_v_t-...`.
- Esempio verificato: MTM `"0102"` → canonical `"0_1_0-0_2_1"`.

**2. Mapping codici lunghi → p1, p2, ... — Step 1d** (`union`, dizionario `short_code`):
- Unisce **ogni** stringa canonica prodotta in qualunque punto della pipeline: pattern MTM completi, regole EvoMine complete, e i "pezzi" riga/colonna (precondizione/postcondizione separate) necessari per le matrici di transizione.
- Assegna codici corti in ordine di **peso combinato decrescente** (somma dei count MTM + supporti EvoMine in cui la stringa compare).
- Risultato: **148 codici lunghi totali** nell'unione → 76 esclusivi EvoMine, 69 esclusivi MTM, **solo 3 condivisi** tra le due fonti (`0_1_0-0_2_1`→p18, `0_1_0-2_0_1`→p21, `0_1_0-1_0_1`→p36).
- Salvato in `output_data/canonical_mapping.csv`.

**3. Transition matrix EvoMine** (Step 2-3):
- `parse_evomine_rules(path)`: legge il file grezzo EvoMine, estrae blocchi `supp=N` + archi `src(role)--label[w]-->dst(role)`. Output: lista di `(supporto, lista_archi_con_label)` — **62 regole totali**.
- `rule_to_body_head_edges(edges)`: separa gli archi con label 3 (body, t=0) da quelli con label 1 (head, t=1) — verificato che la correzione dell'ordine è necessaria perché nel file grezzo gli archi di head possono essere stampati *prima* di quelli di body.
- `evomine_row_col(edges)`: applica `canonical_form` alla regola completa (body+head insieme, per preservare la numerazione condivisa dei nodi), poi separa riga (precondizione, t=0) e colonna (postcondizione, t=1).
- Selezione del cutoff (distribuzione del supporto su 62 regole: min 12, max 32.571, mediana 401, P75 1544, P90 15.714): il gomito più marcato è al rank 14 (rapporto 2.60, supporto 5045→1941), ma tutte le prime 14 regole sono **body-only** (nessuna transizione utile); scelto invece il **gomito secondario a rank 23** (rapporto ≈1.66, supporto 1178→708), che include le prime 8 vere transizioni precondizione→postcondizione oltre alle 14 body-only e alla regola head-only `→01`.
- Per ciascuna delle top-23 regole con body E head non vuoti (8 regole): `frequenza_pre` = supporto della regola body-only con la stessa precondizione (cercata tra **tutte** le 62 regole, non solo le top-23); `probabilità = frequenza_pre_post / frequenza_pre`.
- Output: matrice pivot 4 righe × 4 colonne (`output_data/evomine_transition_matrix.csv`), con **1 cella su 8 pari a NaN** (precondizione `0_1_0-2_3_0`, archi sconnessi, mai osservata come pattern body-only autonomo tra le 62 regole minate).

**4. Transition matrix MTM con codifica GERANIO** (Step 4):
- Stesso calcolo di probabilità del Task 3 (`P = count(prefisso→terzo evento) / count(prefisso→qualsiasi evento)`, escludendo lo stato `S` di non-estensione dal denominatore — correzione della correlatrice già applicata nel Task 3), ma righe/colonne rietichettate con i codici corti `p1...pN` invece della stringa `encode_motif`.
- Output: matrice 6×12 (`output_data/mtm_transition_matrix_geranio.csv`), verificata numericamente identica al Task 3: `0102→03` (codici `p18→p73`) = **0.3752212389380531**, coincidente con il valore Task 3 a 1h (0.3752).

### Parametri usati
`L_MAX=3`, `DELTA=3600s` (1h), `consecutive=YES`, `N=5000` — identici al Task 3/4 versione "1 ora".

### Output prodotti
- `output_data/`: `canonical_mapping.csv` (148 righe), `evomine_top_rules.csv` (23 righe), `evomine_transition_matrix.csv` (4×4), `mtm_transition_matrix_geranio.csv` (6×12).
- `output_png/`: `support_distribution.png` (curva del supporto con marker sui due gomiti, rank 14 e rank 23), `evomine_transition_matrix.png`, `mtm_transition_matrix_geranio.png`, `comparison_heatmaps.png` (heatmap affiancate stessa scala colore).

### Risultati con numeri esatti

- **Motivi MTM distinti (≥2 archi)**: 60 (coerente con il Task 4, 60 motivi distinti indipendenti da DELTA); **6 prefissi a 2 archi** (righe candidate transition matrix); **54 transizioni** prefisso→terzo arco.
- **Regole EvoMine**: 62 totali parsate; **34 con body E head non vuoti** (vere transizioni) su tutte le 62; tra le **top-23** per supporto, solo **8** sono vere transizioni (le prime 14 sono body-only, poi 1 head-only `→01`, poi le 8 transizioni).
- **Distribuzione supporto EvoMine** (tutte le 62 regole): min 12, max 32.571, mediana 401.0, P50 401, P75 1544, P90 15.714, P95 19.131. Soglie: 61 regole con supporto >20, 60 con >50, 57 con >100, 27 con >500, 23 con >1000, 14 con >5000, 13 con >10000.
- **Cutoff scelto**: N=23 (motivazione: gomito primario a rank 14 non utilizzabile perché tutte body-only; gomito secondario a rank 23 dentro il range 15-30 indicato come fallback, con 8 transizioni utilizzabili).
- **Celle in comune tra le due matrici finali**: **0** — 54 celle esclusive MTM, 8 esclusive EvoMine. Correlazione **non calcolabile** (serve almeno 1 cella in comune).

### Punto aperto (clock sequenziale vs binario)
Causa della non-sovrapposizione, per costruzione (definita esplicitamente nel prompt del Task 5), non un errore di implementazione:
- **MTM** usa un **orologio logico sequenziale**: ogni arco del motivo ha una posizione temporale distinta `t=0,1,2,...` (una per arco).
- **EvoMine** usa un **orologio binario body/head**: tutti gli archi di precondizione sono a `t=0` e tutti quelli di postcondizione a `t=1`, indipendentemente da quanti archi compongono ciascun gruppo.
- Le due convenzioni **coincidono solo quando la precondizione EvoMine ha esattamente 1 arco body + 1 arco head** — in quel caso specifico la forma completa della regola EvoMine (t=0,1) combacia esattamente con la forma di un prefisso MTM a 2 archi (t=0,1). Questo spiega perché, sull'unione dei 148 codici lunghi (Step 1d), solo 3 stringhe risultano condivise, e perché le matrici di transizione finali non hanno alcuna cella in comune (le colonne MTM sono "terzo arco isolato a t=2", le colonne EvoMine sono "postcondizione a t=1" — stringhe diverse anche a parità di struttura src/dst).
- Questo è il **punto esplicitamente lasciato aperto**: se si desidera un confronto cella-per-cella più significativo, servirebbe ridefinire uno dei due clock logici (es. rendere il clock MTM binario come EvoMine, oppure normalizzare diversamente le colonne), decisione che spetta alla correlatrice.

### Osservazioni chiave
- Il lavoro del Task 5 conferma sperimentalmente, con numeri esatti, l'ipotesi qualitativa già emersa nei Task 3-4 ("MTM e EvoMine sono complementari, non sovrapponibili 1:1"): con la codifica GERANIO la sovrapposizione strutturale è **quantificabile e pari a zero celle**, un risultato più netto (e più onesto) rispetto al confronto precedente basato su `encode_motif`, che aveva trovato 10/12 regole in comune proprio perché la codifica precedente non distingueva i due clock logici.
- La cella NaN nella matrice EvoMine (`0_1_0-2_3_0`, archi sconnessi tra loro) evidenzia un limite pratico del top-N cutoff: alcune precondizioni compaiono solo *dentro* regole più grandi, mai come pattern autonomo tra le regole minate — capire se questo sia un limite del support threshold o una proprietà strutturale del dataset resta un'ulteriore domanda aperta.
- La verifica di continuità numerica (`0102→03` = 0.375221 identico tra Task 3 e Task 5) conferma che la sola rietichettatura con codici GERANIO non altera in alcun modo i valori di probabilità sottostanti — cambia solo la notazione.

---

## STATO ATTUALE E PROSSIMI PASSI

### Cosa è completato
- **Task 1**: ✅ completo — traduzione Python di MTM/TMC, null model, visualizzazione, validato su facebook_wall e CollegeMsg.
- **Task 2**: ✅ completo — EvoMine via Docker, validato su DBLP precomputed + run propri su CollegeMsg e Facebook Wall.
- **Task 3**: ✅ completo nei contenuti tecnici (transition matrix MTM a 3 granularità, analisi IET, preprocessing GER a 5 granularità, run EvoMine a 30min e 1h) — **in attesa di conferma finale sulla granularità** da parte della correlatrice.
- **Task 4**: ✅ completo — scalabilità TMC (per N e per DELTA), run EvoMine support=1 a entrambe le granularità, confronto MTM↔EvoMine con codifica `encode_motif`, evoluzione strutturale del dataset. Tre disallineamenti metodologici identificati e corretti in questa iterazione (2026-07-08).
- **Task 5**: ✅ completo — nuova codifica canonica GERANIO condivisa, cutoff del supporto EvoMine motivato (N=23), transition matrix EvoMine (4×4, 1 NaN) e MTM (6×12, verificata identica al Task 3) in codici corti, confronto diretto (0 celle in comune) con spiegazione strutturale del perché.

### Cosa è in attesa di feedback da Alessia (correlatrice)
**Come trattare l'asimmetria di clock logico MTM/EvoMine emersa nel Task 5**: se ridefinire uno dei due clock per rendere le matrici di transizione confrontabile cella-per-cella, o se accettare che i due strumenti restino intrinsecamente non sovrapponibili a questo livello di granularità e concentrarsi su un confronto solo qualitativo/per profilo (come nel Task 2).

### Decisioni ancora aperte
- Se estendere la transition matrix MTM a lunghezze diverse dal prefisso 2→3 (es. 1+1, 3+1) per un confronto più ampio con EvoMine, come suggerito nelle conclusioni del report Task 4.
- Se applicare la parte "a comunità" del paper BDMA 2025 (Leiden + profili GER mesoscopici) ai dataset del tirocinio, che finora non è stata replicata.
- Se la cella NaN della matrice EvoMine del Task 5 (precondizione mai osservata come pattern autonomo) vada ricomputata dal dataset grezzo o lasciata come limite noto.

---

## PARAMETRI DI RIFERIMENTO

Tabella riassuntiva di tutti i parametri usati nei vari task, con valori e motivazione.

| Parametro | Task | Valore/i usati | Motivazione |
|---|---|---|---|
| `L_MAX` (MTM) | 1 | 4 (facebook), 3 (CollegeMsg) | 3 per CollegeMsg per limitare il costo esponenziale del TMC |
| `L_MAX` (MTM) | 3-5 | 3 (dopo revisione, era 4) | Allineato ai benchmark di scalabilità TMC del Task 4 e alla run CollegeMsg del Task 1 |
| `DELTA` (MTM/TMC) | 1 | 86400s (1 giorno) | Finestra naturale per reti sociali: correlazioni nella stessa giornata |
| `DELTA` (MTM/TMC) | 3-5 | 1800s (30min) e 3600s (1h), entrambe testate | Sub-oraria, coerente con mediana IET=47s (Task 3) |
| `K` (grafi sintetici generati) | 1 | 1 | Sufficiente per confronto qualitativo |
| `consecutive` | 1 | NO | Conta tutte le istanze, non solo quelle da un evento iniziale distinto |
| `consecutive` | 3-5 | YES | Comportamento nativo MTM, comparabile con la logica body→head di EvoMine |
| `N_SAMPLE` | 1 | 3000 (facebook), 5000 (CollegeMsg) | Bilancio tempo/copertura |
| `N_SAMPLE` | 3-5 | 5000 (CollegeMsg) | Tempo TMC trascurabile fino a questa dimensione (Task 4) |
| `k_neighbors` / `p_rewire` (Watts-Strogatz) | 1 | 4 / 0.1 | Standard ring lattice + 10% rewiring |
| `-s` support (EvoMine) | 2 | 20000 (CollegeMsg), 500 (Facebook) | Evitare run di ore (NP-hard su support basso) |
| `-s` support (EvoMine) | 3-4 | 1 | Nessun filtro: tutte le regole GER possibili, per un confronto esaustivo con MTM |
| `-e` max_edges (EvoMine) | 2-5 | 3 | Standard nel lavoro della correlatrice, limita il costo di gSpan |
| `-T` (EvoMine) | 2-5 | full | MIB support completo |
| Cutoff supporto EvoMine (top-N) | 5 | N=23 | Gomito secondario, include le prime 8 vere transizioni body→head (il gomito primario a rank 14 sarebbe tutto body-only) |

---

## GLOSSARIO TECNICO

- **TMC (Temporal Motif Counting)**: algoritmo che conta le istanze di ogni tipo di motivo temporale in una rete, scorrendo gli eventi ed estendendo i prefissi attivi entro una finestra `delta`.
- **MTM (Motif Transition Model)**: modello che apprende le probabilità/tassi di transizione tra motivi temporali successivi e li usa per generare reti sintetiche, senza contare esplicitamente i motivi (più efficiente del TMC puro).
- **GER (Graph Evolution Rule)**: regola "precondizione (body, t=0) → postcondizione (head, t>0)" che descrive come un sottografo frequente si trasforma nel tempo.
- **EvoMine**: algoritmo di mining delle GER per grafi diretti, basato su gSpan e sul concetto di grafo dell'unione (union graph); supporta rimozione archi e rietichettatura (a differenza del più semplice GERM).
- **GERANIO**: nome del toolkit/repository della correlatrice che implementa EvoMine/GERM; nel Task 5 è anche il nome informale dato allo schema di codifica canonica condivisa MTM/EvoMine (`src_dst_t-...`).
- **Canonical coding**: rinomina dei nodi di un pattern in ordine di prima apparizione (0,1,2,...), per rendere confrontabili istanze isomorfe con etichette di nodo diverse. Nel tirocinio esistono due varianti: `encode_motif` (stringa a cifre concatenate, Task 1/3/4) e la codifica GERANIO (stringa `u_v_t` con trattino, Task 5).
- **Enrichment score**: rapporto tra il conteggio di un motivo nella rete reale e la media dei conteggi nei null model; enrichment>1 = motivo statisticamente sovra-rappresentato (significativo).
- **Null model**: rete randomizzata usata come termine di paragone per stabilire la significatività statistica di un pattern osservato. Tre varianti usate nel Task 1: Time Shuffle (rimescola i timestamp), Edge Shuffle/configuration model (rimescola gli endpoint preservando la degree sequence), Watts-Strogatz (topologia small-world sintetica completamente diversa).
- **MIB support (Minimum Image-Based support)**: misura di supporto per il frequent subgraph mining che evita l'overcounting di embedding sovrapposti dello stesso pattern; alternativa concettuale al più semplice "supporto basato su eventi" (conta i grafi-evento in cui compare il pattern), quest'ultimo usato nel lavoro di tirocinio.
- **Antimonotonicità**: proprietà usata dagli algoritmi di frequent subgraph mining (gSpan) per potare l'albero di ricerca: se un pattern non è frequente, nessuna sua estensione può esserlo, quindi si evita di esplorarla.
- **Temporal motif**: sottografo temporale connesso con l eventi ordinati cronologicamente, dove ogni evento condivide almeno un nodo con uno degli eventi precedenti (definizione formale nel paper KDD 2023, Sezione 3.1).
- **Inter-event time (IET)**: intervallo di tempo tra un evento e il successivo (in genere calcolato sull'intera sequenza ordinata di eventi di un dataset); una IET mediana bassa con coda pesante indica attività "a burst".
- **CCDF (Complementary Cumulative Distribution Function)**: P(X>x), usata per visualizzare distribuzioni heavy-tail come l'IET in scala log-log.
- **L_MAX**: numero massimo di eventi (archi) che un motivo temporale può avere; parametro di MTM/TMC.
- **DELTA (δ)**: finestra temporale massima entro cui un nuovo evento può estendere un prefisso attivo di un motivo/processo di transizione.
- **consecutive**: flag di TMC/MTM; se YES, un prefisso viene "consumato" (rimosso dai prefissi attivi) non appena si estende, evitando il doppio conteggio della stessa sequenza di eventi in più motivi sovrapposti.
- **Support (EvoMine)**: soglia minima di frequenza (basata su eventi o su MIB) sotto la quale una regola GER non viene riportata in output.
- **Transition matrix**: tabella P(evento successivo | prefisso/precondizione osservato), il cuore sia del modello MTM sia del confronto MTM↔EvoMine nei Task 3-5.
- **Novelty rate**: frazione di archi in un bucket temporale che coinvolgono almeno un nodo mai osservato prima in quel bucket — usato nell'analisi di evoluzione strutturale del Task 4.
- **Heavy-tail**: distribuzione con coda pesante (pochi valori molto grandi, molti valori piccoli), tipica degli IET nelle reti sociali/di messaggistica (osservata sia nel Task 1 sull'enrichment sia nel Task 3 sull'IET).
- **Bucket temporale**: intervallo discreto di tempo (es. 30 minuti, 1 ora, 1 giorno) usato per discretizzare i timestamp continui, necessario per EvoMine (che richiede uno snapshot per ogni timestamp discreto, non Unix epoch continuo).
- **Body / Head**: sinonimi di precondizione/postcondizione in una GER; nel formato grezzo di EvoMine l'etichetta d'arco **3 = body (t=0)** e **1 = head (t>0)** — convenzione controintuitiva verificata esplicitamente in `file_converters.py` (`mapping_ts = {3: 0, 1: 1}`).
- **Prefisso**: nella terminologia MTM, la sequenza di eventi (motivo parziale, tipicamente a 2 archi nei Task 3-5) che precede l'estensione al terzo evento; concettualmente analogo alla precondizione di una GER, ma con un clock logico sequenziale (non binario) — differenza chiave discussa nel Task 5.
