# Discorso di discussione — ~14 minuti

> Traccia per la discussione orale. Ogni sezione corrisponde a una slide.
> I tempi indicati sono di riferimento; il totale è di circa 13–14 minuti.
> Le note tra parentesi quadre `[...]` sono indicazioni di regia, non da leggere.

---

## Slide 1 — Titolo *(~30 s)*

Buongiorno a tutti. Mi chiamo Matteo Sardi e vi presento il mio lavoro di tesi, dal titolo *"Generazione Sintetica e Segmentazione di Immagini ECG: Pipeline per la Digitalizzazione del Tracciato"*, svolto sotto la supervisione del professor Matteo Sereno e del dottor Daniele Baccega, che ringrazio.

[Pausa, passa alla slide dell'indice.]

---

## Slide 2 — Struttura della presentazione *(~20 s)*

Il lavoro si articola in due stadi: la generazione sintetica di immagini ECG annotate e la loro segmentazione automatica. Vi guiderò attraverso entrambi, partendo dal problema clinico che li motiva.

---

## Slide 3 — Sezione: Contesto e Obiettivo *(transizione)*

Partiamo dal contesto.

---

## Slide 4 — Il problema: archivi clinici ancora su carta *(~1 min)*

L'elettrocardiogramma è l'esame che registra l'attività elettrica del cuore; lo standard clinico è quello a 12 derivazioni, che vedete qui su carta millimetrata.

Il progetto nasce da un'esigenza concreta: il dottor Baccega lo ha avviato dopo essersi confrontato con il personale dell'ospedale Molinette di Torino, dove il problema si presenta ogni giorno. Nonostante la digitalizzazione della sanità, una quantità enorme di dati diagnostici storici esiste ancora **soltanto su carta**, o come scansioni. Per analizzarli su larga scala serve convertirli automaticamente in segnali numerici.

L'ostacolo è che addestrare un modello di Computer Vision robusto richiede grandi quantità di immagini annotate — e dataset con la variabilità e le imperfezioni dei documenti ospedalieri reali, in pratica, non esistono. È questo che il lavoro affronta.

---

## Slide 5 — Lavori correlati: un gap aperto *(~50 s)*

La digitalizzazione degli ECG cartacei è già stata affrontata in letteratura, ma riutilizzare gli approcci esistenti si è rivelato difficile: diversi metodi hanno codice **non funzionante** o non riproducibile; altri sono **instabili** sui tracciati rumorosi o richiedono un processo manuale; altri ancora sono a pagamento o non rilasciano il codice.

Esiste un toolbox open-source per la generazione sintetica, *ECG-Image-Kit*, ma non produce immagini abbastanza degradate da simulare le condizioni reali. Il mio contributo nasce per colmare questo doppio vuoto: una pipeline di generazione più controllata **e** un segmentatore addestrato proprio su immagini con artefatti realistici.

---

## Slide 6 — Obiettivo: una pipeline a due stadi *(~55 s)*

L'idea centrale è una pipeline a due stadi.

Il **primo stadio** parte da segnali ECG digitali esistenti e li trasforma in immagini realistiche e degradate. Il punto chiave è che, generando io l'immagine, conosco esattamente dove passa il tracciato: ottengo "gratis" una **ground truth**, cioè una maschera del segnale, senza alcuna annotazione manuale.

Il **secondo stadio** usa queste coppie immagine–maschera per addestrare una rete a fare il percorso inverso: isolare il tracciato da griglia, sfondo e artefatti, anche su documenti degradati.

Una precisazione di onestà: la conversione finale dalla maschera al segnale numerico non fa parte di questo lavoro — è lo sviluppo futuro naturale. Qui mi fermo alla separazione del tracciato.

---

## Slide 7 — Sezione: Stadio 1 *(transizione)*

Vediamo il primo stadio, la generazione.

---

## Slide 8 — Dal segnale all'immagine: rendering calibrato *(~55 s)*

Il primo passo è trasformare il segnale numerico in immagine. Lo faccio con un motore di rendering calibrato sulle convenzioni della carta millimetrata clinica: sull'asse orizzontale 5 millimetri valgono 0,2 secondi, su quello verticale 5 millimetri valgono mezzo millivolt.

Questa calibrazione **millimetrica** non è estetica: garantisce una corrispondenza precisa tra pixel e grandezze fisiche, condizione necessaria perché in futuro si possa ricostruire il segnale reale.

Il sistema supporta i quattro layout della pratica clinica e l'ho costruito estendendo *ECG-Image-Kit*, a cui ho aggiunto, tra le altre cose, il supporto a un secondo dataset in formato HDF5, una configurazione esterna in YAML e un rendering interamente in memoria.

---

## Slide 9 — Il domain gap: artefatti realistici e contributi originali *(~1 min 15 s)*

Qui arriviamo al cuore del primo stadio. Un modello addestrato su immagini **pulite** crolla appena lo si mette davanti a documenti reali, che accumulano difetti durante tutto il loro ciclo di vita — stampa, manipolazione, conservazione, scansione. Questo divario si chiama *domain gap*.

Per ridurlo ho costruito una pipeline che inietta tre classi di artefatti realistici: il **testo manoscritto** delle annotazioni mediche, generato con una rete neurale; le **pieghe e le rugosità** della carta; e una serie di **degradazioni fotometriche e geometriche** — rotazioni, sfocatura, rumore, compressione JPEG, variazioni di luminosità. A destra ne vedete alcuni esempi e, in basso, la configurazione completa con tutti gli artefatti combinati: un'immagine che assomiglia a un documento davvero scansionato.

Tra i contributi originali rispetto al repository di partenza ne segnalo due: il **dropout differenziale**, che degrada sfondo e tracciato con probabilità diverse — utile per simulare carta rovinata ma inchiostro ancora leggibile — e la **generazione automatica della maschera**, che vediamo ora.

---

## Slide 10 — Coppia immagine / maschera: ground truth gratuita *(~45 s)*

Questo è il prodotto chiave del primo stadio. A sinistra l'immagine sintetica degradata, input del modello; a destra la maschera binaria, con i soli tracciati in bianco.

Il punto è che entrambe subiscono **le stesse trasformazioni geometriche**: quando ruoto o sposto il tracciato nell'immagine, lo ruoto e lo sposto identicamente nella maschera. La corrispondenza resta **pixel per pixel** ed è esattamente la ground truth che serve per la segmentazione — senza che nessuno l'abbia annotata a mano.

---

## Slide 11 — Risultati Stadio 1: parallelizzazione *(~50 s)*

Generare un dataset così è costoso, ma il carico è *embarrassingly parallel*: ogni ECG è indipendente dagli altri.

Ho introdotto la parallelizzazione con `ProcessPoolExecutor`, che usa processi separati per aggirare il *Global Interpreter Lock* di Python e sfruttare tutti i core. Lo speedup cresce fino a circa **3 volte con 8 worker** — il numero di core fisici della macchina di test. Oltre, le prestazioni calano: il carico è **CPU-intensive**, non I/O-bound, quindi l'hyperthreading non aiuta — i thread logici aggiuntivi competono per le stesse unità di calcolo invece di riempire tempi morti. Per questo ho limitato automaticamente i worker ai core fisici.

---

## Slide 12 — Sezione: Stadio 2 *(transizione)*

Passiamo al secondo stadio: il modello di segmentazione.

---

## Slide 13 — Architettura: U-Net + attenzione scSE *(~1 min 10 s)*

Il modello è una **U-Net**, l'architettura di riferimento per la segmentazione biomedicale. Ha la forma di una "U": un **encoder** che comprime progressivamente l'immagine — passando dal "dove sono i pixel" al "cosa contiene la regione" — e un **decoder** simmetrico che risale alla risoluzione di partenza e assegna a ogni pixel una probabilità: tracciato o sfondo.

Comprimendo, però, si perde la posizione esatta dei dettagli fini, e i tracciati sono spessi pochi pixel. Lo risolvono le **skip connection**, le frecce grigie nello schema: collegano ogni livello dell'encoder al corrispondente del decoder, reiniettando il dettaglio ad alta risoluzione.

L'encoder è una **ResNet-50 pre-addestrata su ImageNet**, così riuso filtri generici di bordi e texture già appresi, ed è intercambiabile. La vera novità è nel **decoder, con attenzione scSE**: un filtro appreso che ricalibra le feature su due assi — *quali* caratteristiche contano e *dove* contano. Serve contro un problema preciso: la griglia millimetrata somiglia al tracciato — anch'essa fatta di linee sottili — e genera falsi positivi; l'attenzione impara a sopprimere la griglia e a tenere il tracciato. [Se chiedono il meccanismo, Appendice B.]

---

## Slide 14 — Funzione di perdita composita *(~1 min 10 s)*

La funzione di perdita somma tre componenti, ciascuna pensata per un fallimento diverso e complementare.

La **Focal Loss** affronta lo sbilanciamento estremo: i pixel di tracciato sono meno del 5%, e una loss standard imparerebbe semplicemente a dire "tutto sfondo". La Focal pesa ogni pixel in base alla difficoltà: i pixel facili contribuiscono quasi nulla al gradiente, quelli difficili lo guidano.

La **Dice Loss** lavora sulla forma complessiva: misura la sovrapposizione tra maschera predetta e ground truth ed è insensibile allo sbilanciamento. Le due sono complementari: la Focal porta la rete sui pixel giusti, la Dice garantisce che la forma globale sia corretta.

La terza, la **clDice**, è la più interessante. Una maschera può avere ottime Focal e Dice ed essere comunque inutilizzabile: basta una piccola lacuna che spezza il tracciato, e le prime due — che ragionano pixel per pixel — non la "vedono". La clDice confronta invece gli **scheletri** delle due maschere: se la predizione ha un buco, lo scheletro si interrompe e la penalità sale. Penalizza cioè le interruzioni *topologiche*. Per questo il suo peso cresce stage dopo stage: più gli artefatti sono pesanti, più conta difendere la continuità del tracciato. [Soft-skeleton differenziabile: Appendice C.]

---

## Slide 15 — Curriculum learning *(~1 min 10 s)*

E qui entra l'ultima idea chiave: il **curriculum learning**. Invece di mostrare subito alla rete le immagini più difficili, la addestro in tre stage di difficoltà crescente — prima immagini pulite, poi degradazioni moderate, infine i campioni più complessi — senza mai reinizializzare il modello.

L'intuizione è pedagogica: partendo dagli esempi peggiori il gradiente è dominato dal rumore, prima ancora che il modello abbia una rappresentazione stabile del tracciato, e rischia di restare bloccato in minimi locali difficili. Facendogli imparare prima la geometria di base su immagini pulite, e raffinandola poi, la generalizzazione migliora nettamente.

A ogni passaggio di stage muovo **quattro leve coordinate**: scongelo progressivamente l'encoder; riduco il learning rate di un ordine di grandezza per volta; aumento il dropout, per regolarizzare di più man mano che il compito si fa difficile; e alzo il peso della clDice, per difendere la continuità proprio quando gli artefatti minacciano di spezzare i tracciati. Il grafico a destra mostra la diagnostica dello stage più difficile.

---

## Slide 16 — Risultati Stadio 2: effetto del curriculum *(~50 s)*

Questi sono i risultati sul test set sintetico. Per ogni immagine vedete tre colonne: l'input, la mappa di confidenza del modello e la maschera finale.

Il confronto è tra il primo e l'ultimo stage del curriculum: allo stage iniziale le maschere sono rumorose e includono pezzi di griglia come falsi positivi; allo stage finale sono molto più pulite e i tracciati più completi. È la conferma visiva che l'addestramento progressivo funziona.

---

## Slide 17 — Estensione: detection head multi-task *(~55 s)*

Come estensione ho aggiunto una *detection head* che, oltre a segmentare, localizza i 12 lead: per ciascuno dice se è presente e ne stima la regione.

Il meccanismo è una **cross-attention per lead**: ogni derivazione ha una *query* appresa che la confronta con tutte le posizioni della feature map e ne estrae un riassunto visivo della propria zona — in pratica ogni lead impara a *cercarsi da solo* nell'immagine. I riquadri sono **oriented bounding box**: predico anche l'angolo, così il box segue le inclinazioni introdotte dalle augmentation invece di includere molto sfondo vuoto. Per non disturbare la segmentazione, questa testa è addestrata in uno **stage finale dedicato, con encoder e decoder congelati**. [Dettagli su codifica dell'angolo e loss in Appendice E.]

---

## Slide 18 — Sezione: Conclusioni *(transizione)*

Arrivo alle conclusioni.

---

## Slide 19 — Conclusioni *(~55 s)*

In sintesi, il lavoro ha prodotto due contributi. Da un lato una pipeline di **generazione sintetica** estesa, con artefatti realistici, ground truth automatica, parallelizzazione e supporto multi-dataset. Dall'altro un modello di **segmentazione** U-Net con attenzione scSE, perdita composita e curriculum learning, più una testa di detection per la localizzazione dei lead.

Gli sviluppi futuri sono due: completare la digitalizzazione convertendo la maschera in segnale numerico, e validare il sistema in modo esteso su archivi clinici reali.

---

## Slide 20 — Grazie *(~10 s)*

Vi ringrazio per l'attenzione e resto a disposizione per le vostre domande.

---

### Note di regia

- **Tempo totale stimato:** ~13–14 minuti a ritmo regolare. Prova il discorso **una volta con il cronometro**: se superi i 14 minuti, i tagli più sicuri sono la slide 17 (detection, riassumibile in due frasi) e la coda tecnica delle slide 14 e 15.
- Rallenta sulle slide **14 (loss)** e **15 (curriculum)**: sono quelle su cui è più probabile ricevere domande.
- Non leggere a memoria i nomi dei parametri/funzioni: bastano i concetti.
- Le frasi tra `[...]` rimandano all'Appendice e **non vanno lette**.

---

# Appendice — Risposte alle domande tecniche attese

> Materiale di preparazione per il question time, **non** da leggere durante l'esposizione.
> Ogni voce è una possibile domanda della commissione con una risposta sintetica ma precisa.
> Le risposte sono allineate al Capitolo 5 della tesi.

---

## A) Architettura U-Net, encoder e decoder

**D — Che cos'è una U-Net e perché è adatta a questo problema?**
È una rete encoder–decoder a forma di "U". L'**encoder** (percorso di contrazione) riduce progressivamente la risoluzione spaziale tramite pooling ed estrae feature sempre più astratte ma povere di dettaglio spaziale; il **decoder** (percorso di espansione) risale alla risoluzione originale tramite up-convolution, producendo una predizione *per pixel*. È l'architettura di riferimento per la segmentazione biomedicale perché restituisce una maschera densa della stessa dimensione dell'input, esattamente ciò che serve per separare il tracciato pixel per pixel.

**D — A cosa servono le skip connection?**
Collegano direttamente le feature ad alta risoluzione dell'encoder ai corrispondenti livelli del decoder. Senza di esse il decoder dovrebbe ricostruire i dettagli fini solo da feature molto compresse, perdendo precisione sui bordi. Per i tracciati ECG — strutture spesse pochi pixel — questo è cruciale: le skip connection reiniettano l'informazione di localizzazione fine che il pooling aveva scartato.

**D — Perché un encoder ResNet-50 pre-addestrato su ImageNet? Non è un dominio diverso?**
Sì, ImageNet sono foto naturali, ma i **filtri dei primi livelli** (bordi, texture, gradienti) sono generici e trasferibili a qualsiasi immagine, ECG inclusi. Partire da pesi pre-addestrati anziché casuali accelera la convergenza e migliora la generalizzazione, soprattutto con dataset relativamente piccoli. L'encoder è comunque **intercambiabile** (`--encoder`): ResNet-50 ed EfficientNet-B5 offrono buon compromesso precisione/VRAM, mentre backbone transformer come MiT-B4 modellano dipendenze a lungo raggio al costo di più memoria.

**D — Perché normalizzate con media/std di ImageNet?**
Perché i pesi pre-addestrati dell'encoder sono stati appresi su immagini normalizzate con quei valori esatti ($\mu=[0.485,0.456,0.406]$, $\sigma=[0.229,0.224,0.225]$). Presentare immagini con una distribuzione diversa degraderebbe le feature estratte, vanificando il vantaggio del pre-addestramento.

---

## B) Attenzione scSE

**D — Cos'è l'attenzione scSE?**
scSE (*concurrent Spatial and Channel Squeeze & Excitation*) è un blocco di attenzione inserito nel decoder che **ricalibra** le feature map prima della predizione finale, filtrandole su due assi indipendenti che vengono poi sommati elemento per elemento:
- **cSE — gate di canale ("quali" feature contano):** un global average pooling comprime ogni canale in uno scalare (quanto quel tipo di feature è presente globalmente); due layer fully-connected + sigmoid producono un peso in $[0,1]$ per canale, che moltiplica il canale. Addestrato end-to-end, impara a **scalare verso il basso i canali che rispondono a linee sottili e periodiche** — la griglia — non predittive del tracciato.
- **sSE — gate spaziale ("dove" guardare):** una convoluzione $1\times1$ comprime tutti i canali in uno scalare per pixel → una mappa di rilevanza spaziale in $[0,1]$ che moltiplica la feature map posizione per posizione. Attenua le zone di puro sfondo/griglia e mantiene quelle con tracciato reale.

**D — Perché esattamente questo riduce i falsi positivi sulla griglia?**
Un falso positivo qui è un **pixel di griglia predetto come tracciato**. La griglia inganna perché condivide la firma a basso livello del tracciato (linee sottili, orientamento misto orizzontale/verticale): gli stessi filtri convoluzionali si attivano su entrambi, e queste attivazioni "spingono" i pixel di griglia sopra la soglia di 0,5. Il problema peggiora in due casi: (a) sugli ECG in **bianco e nero**, dove sparisce il segnale cromatico rosso-griglia/nero-tracciato; (b) in corrispondenza delle **skip connection**, che reiniettano nel decoder il dettaglio ad alta risoluzione — ed è proprio lì che vive la griglia, che è un segnale ad alta frequenza.

scSE agisce come **discriminatore appreso**: una feature sopravvive solo se è *sia* un tipo rilevante (cSE) *sia* in una posizione rilevante (sSE). Questo doppio filtro corrisponde quasi esattamente alla distinzione tracciato-vs-griglia: la griglia è ovunque nello spazio ma è un *tipo* di feature distinto → la cSE la sopprime globalmente; dove il tracciato è sovrapposto alla griglia, cSE e sSE insieme tengono il segnale del tracciato e smorzano la componente di griglia. Al momento della decisione le attivazioni di griglia risultano attenuate, meno pixel superano la soglia → meno falsi positivi. Inserire il blocco nel **decoder** è la scelta giusta perché ri-filtra esattamente il dettaglio ad alta risoluzione che le skip connection riportano indietro: conserva i bordi del tracciato e scarta la texture della griglia.

**D — È una garanzia?**
No: è una ricalibrazione *appresa*, quindi l'interpretazione "sopprime i canali della griglia" descrive il comportamento dopo l'addestramento, non una garanzia in forma chiusa. L'evidenza che funziona è empirica — il calo di falsi positivi da stage 0 a stage 2 e la differenza visibile nelle maschere — non una dimostrazione analitica.

---

## C) Funzione di perdita composita

**D — Perché tre loss invece di una? Cosa corregge ciascuna?**
$\mathcal{L} = w_{\text{focal}}\mathcal{L}_{\text{Focal}} + w_{\text{dice}}\mathcal{L}_{\text{Dice}} + w_{\text{cl}}\mathcal{L}_{\text{clDice}}$. Ognuna affronta un fallimento diverso e complementare:
- **Focal Loss** → lo **sbilanciamento estremo** (i pixel di tracciato sono < 5%). Modula il peso per difficoltà: $\mathcal{L}_{\text{Focal}}=-\frac{1}{N}\sum_i \alpha_t(1-p_t)^\gamma\log(p_t)$, con $\gamma=2.0$ (penalizza i pixel facili) e $\alpha=0.8$ sui pixel di tracciato. Senza di essa la BCE imparerebbe a dire "tutto sfondo".
- **Dice Loss** → la **forma globale**: $1-\frac{2\sum p_i g_i}{\sum p_i + \sum g_i}$. Misura la sovrapposizione maschera/ground truth ed è intrinsecamente insensibile allo sbilanciamento (si normalizza sui positivi, non sul totale).
- **clDice (centerline Dice)** → la **continuità topologica**. Focal e Dice ottimizzano la correttezza pixel-wise ma non penalizzano una piccola lacuna che spezza il tracciato e rende il segnale inutilizzabile. La clDice confronta gli **scheletri** delle due maschere: una lacuna interrompe lo scheletro e produce penalità alta.

**D — La scheletrizzazione non è non-differenziabile? Come la usate in backprop?**
Sì, la scheletrizzazione morfologica classica non è derivabile. La clDice usa un **soft skeleton**: sostituisce le erosioni/aperture discrete con operazioni di **pooling differenziabili** (min/max-pooling) e una ReLU per azzerare i residui negativi. Lo scheletro è costruito iterativamente accumulando i residui di apertura su versioni progressivamente erose; l'implementazione usa **15 iterazioni**, così la perdita è sensibile anche a interruzioni lontane dagli estremi. La formula finale è $1-\frac{2 T_{\text{prec}} T_{\text{sens}}}{T_{\text{prec}}+T_{\text{sens}}}$, dove $T_{\text{prec}}$ verifica che lo scheletro predetto sia nel posto giusto e $T_{\text{sens}}$ che nessuna parte del tracciato reale sia persa. Su ground truth vuota il termine viene saltato per evitare gradienti instabili.

**D — Come scegliete i pesi delle tre loss?**
Focal e Dice hanno **peso costante**; il peso della **clDice cresce stage dopo stage** del curriculum, perché con artefatti più pesanti aumenta il rischio di interruzioni nei tracciati ed è lì che la continuità topologica va difesa di più.

**D — Nel grafico la Dice domina le altre: è un problema?**
No. La Dice ha scala naturale in $[0,1]$ quindi è quantitativamente più grande, ma Focal e clDice — pur di valore assoluto inferiore — pesano in modo proporzionalmente importante sul **gradiente** complessivo grazie ai loro coefficienti. Il valore assoluto di una loss non equivale al suo contributo all'aggiornamento dei pesi.

---

## D) Curriculum learning

**D — In cosa consiste il curriculum e perché aiuta?**
Tre stage di difficoltà crescente — `stage0_easy` (pulite) → `stage1_medium` → `stage2_hard` — **senza mai reinizializzare** il modello. L'intuizione è pedagogica: partendo dagli esempi peggiori il gradiente è dominato dal rumore e il modello rischia minimi locali difficili. Imparando prima la geometria di base su immagini pulite e raffinandola poi, la generalizzazione migliora nettamente.

**D — Cosa cambia concretamente tra uno stage e l'altro?**
Quattro leve si muovono insieme: (1) **learning rate** decrescente ($10^{-4}\to10^{-5}\to10^{-6}$, con cosine annealing dentro ogni stage); (2) **unfreezing progressivo** dell'encoder (stage0 congelato → stage1 scongela `layer3`/`layer4` → stage2 tutto scongelato); (3) **dropout** crescente, $p=0.05\times(\text{stage\_idx}+1)$; (4) **peso clDice** crescente. Nota: stage1 e 2 usano la **stessa** intensità di augmentation, così la progressione deriva dall'unfreezing, non da un cambio di distribuzione dei dati.

**D — Perché tenete il BatchNorm in modalità eval anche quando scongelate l'encoder?**
Le statistiche dei layer BatchNorm sono già calibrate su ImageNet. Aggiornarle su batch piccoli di ECG (distribuzione molto diversa) produrrebbe stime rumorose che corrompono le feature dell'encoder. Congelandole ai valori pre-addestrati l'encoder resta coerente anche mentre i suoi pesi vengono aggiornati.

**D — Perché la val loss è più bassa della train loss? Non è anomalo?**
È atteso con dropout e augmentation. Il **dropout** è attivo solo in training (PyTorch lo disabilita in `eval()`), riducendo la capacità effettiva durante l'addestramento. Inoltre l'**augmentation di training è casuale a ogni epoch** (immagini sempre nuove), mentre quella di **validation è fissa per campione**: il set di validazione è quindi mediamente più stabile. Il gap si chiude man mano che il cosine annealing abbassa il learning rate.

---

## E) Detection head multi-task (estensione)

**D — Come fa la testa di detection a localizzare i 12 lead?**
Con una **cross-attention per lead**: ogni derivazione ha una *query appresa* che, via prodotto scalare con i descrittori di cella della feature map, calcola pesi di attenzione (softmax) e ne estrae un **vettore di contesto** come media pesata. I punteggi sono divisi per $\sqrt{d}$ (con $d=256$) per evitare la saturazione del softmax e gradienti nulli. Ogni vettore passa poi a un MLP per-lead che produce 13 logit: 1 di presenza + 6 per l'OBB del tracciato + 6 per l'OBB dell'etichetta.

**D — Perché oriented bounding box e non assi-allineati?**
Le augmentation ruotano i tracciati: un box allineato agli assi (AABB, 4 parametri) attorno a un tracciato inclinato include molto sfondo → errore di localizzazione sistematico. L'**OBB** aggiunge l'angolo $\theta$ e racchiude fedelmente regioni inclinate.

**D — Perché codificate l'angolo come $\sin 2\theta/\cos 2\theta$?**
Per la simmetria a $180°$ del rettangolo: con la codifica diretta $\theta$ e $\theta+180°$ sarebbero due target validi per lo stesso box → segnale ambiguo. Usando $2\theta$ i due angoli equivalenti collassano nello stesso punto $(\sin,\cos)$, rendendo il target univoco. Inoltre si predicono **6 parametri** $[c_x,c_y,w,h,\sin2\theta,\cos2\theta]$ invece di 8 coordinate libere, garantendo *per costruzione* un rettangolo valido.

**D — Perché GIoU per le coordinate e MSE per l'angolo?**
La **GIoU** ottimizza direttamente la sovrapposizione e, grazie al termine del contenitore minimo, dà gradiente utile **anche quando i box sono disgiunti** (dove la IoU pura avrebbe gradiente nullo). La **MSE** sugli angolari va bene perché i target sono in $[-1,1]$ (niente gradienti esplosivi) ed è simmetrica rispetto allo zero, coerente con la natura circolare dell'angolo. I termini geometrici/angolari sono calcolati **solo sui lead presenti** (mascherati sulla ground truth di presenza).

**D — La detection non degrada la segmentazione?**
No: durante i tre stage di curriculum la `DetectionHead` esiste ma **non riceve gradiente**. Viene addestrata in un **quarto stage dedicato** (`det_finetune`) con encoder e decoder **congelati**, ottimizzatore e scheduler reinizializzati; parte da pesi casuali e converge con LR $10^{-4}$.

---

## F) Generazione, dati e parallelizzazione

**D — Da dove viene la ground truth? Non c'è annotazione manuale?**
La maschera è generata **insieme** all'immagine: poiché disegno io il tracciato, so esattamente dove passa. Le **stesse trasformazioni geometriche** (rotazione, shift) sono applicate in modo sincronizzato a immagine e maschera, garantendo corrispondenza **pixel per pixel**. Nessuna annotazione umana.

**D — Perché interpolazione NEAREST per le maschere e bilineare per le immagini?**
La bilineare media i pixel e produrrebbe valori intermedi (0.3, 0.7) ai bordi della maschera, introducendo artefatti dopo la binarizzazione. La **NEAREST** copia il valore del pixel più vicino, preservando la natura binaria della maschera.

**D — Lo speedup si ferma a ~3× con 8 worker: perché non scala oltre?**
Il carico è *embarrassingly parallel* ma **CPU-bound**. `ProcessPoolExecutor` usa processi separati per aggirare il **GIL** di Python e usare i core reali. Il massimo è a 8 worker = numero di **core fisici**. Oltre, l'hyperthreading non aiuta: i thread logici darebbero vantaggio solo riempiendo tempi morti (es. attese I/O), ma qui ogni processo tiene le unità di calcolo occupate quasi al 100% → i thread aggiuntivi competono per le stesse risorse invece di riempire vuoti. Per questo limito automaticamente i worker ai core fisici.

---

## G) Valutazione e onestà del lavoro

**D — Avete validato su ECG reali?**
I risultati mostrati sono sul **test set sintetico** (split fisso, seed 42, identico tra gli stage). La validazione estesa su archivi clinici reali è dichiarata come **sviluppo futuro**, insieme alla conversione finale maschera → segnale numerico, che **non** fa parte di questo lavoro: qui mi fermo alla separazione del tracciato.

**D — Come leggete le visualizzazioni dei risultati (3 colonne)?**
Input, **heatmap di confidenza** (output dopo sigmoid, colormap jet: rosso ≈ 1 tracciato, blu ≈ 0 sfondo; il giallo/verde attorno a 0.5 segnala incertezza, tipicamente sulla griglia) e **maschera binaria** (heatmap sogliata a 0.5). Il confronto stage0 vs stage2 mostra meno falsi positivi di griglia e tracciati più continui allo stage finale — conferma visiva che il curriculum funziona.
