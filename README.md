# Passero.AccountingDocumentAI
L’AI per la gestione dei fatti contabili a partire dai documenti 
 L’obiettivo è avere una piattaforma che:

    acquisisce documenti
    li classifica
    estrae i dati
    consulta knowledge base e storico
    propone movimenti contabili e/o di magazzino
    sottopone a revisione i casi dubbi
    scrive nell’ERP solo quando consentito

1) Architettura applicativa concreta

1.1 Vista logica dei componenti

Frontend / UI

    Document Review UI
    Queue Monitor UI
    Knowledge Base Admin UI
    Rules Admin UI
    Mapping Admin UI
    Audit & Explain UI

Backend applicativo

    Document Intake Service
    Document Processing Orchestrator
    OCR / Extraction Service
    Document Classification Service
    Field Extraction Service
    RAG / Knowledge Retrieval Service
    Accounting Rules Engine
    Inventory Rules Engine
    Proposal Composer Service
    Validation Service
    Human Review Workflow Service
    ERP Integration Service
    Feedback Learning Service

Data layer

    Operational SQL DB
    Vector DB
    Object Storage
    Queue / Bus
    Observability / Audit Store

1.2 Componenti

A. Document Intake Service

Responsabilità:

    ricezione documenti
    import da cartelle, email, API, ERP, PEC, FTP, upload UI
    hashing
    deduplica preliminare
    creazione record iniziale

Input:

    PDF
    immagini
    XML
    ZIP
    email con allegati

Output:

    DocumentEnvelope
    file salvato su object storage
    evento su coda document.received

B. Document Processing Orchestrator

È il coordinatore della pipeline.

Responsabilità:

    sequenziare gli step
    rilanciare in caso di errore
    mantenere stato
    parallelizzare dove possibile
    decidere fallback

Esempio step:

    detect format
    parse / OCR
    classify
    extract
    validate
    retrieve knowledge
    compose proposals
    confidence scoring
    route review/autopost

C. OCR / Extraction Service

Responsabilità:

    estrazione testo da PDF nativi
    OCR da scansioni
    layout analysis
    identificazione tabelle
    separazione pagine/sezioni
    rilevazione lingua
    ricostruzione blocchi header/body/footer

Output:

    testo integrale
    testo per pagina
    blocchi layout
    righe tabellari
    confidence OCR

Questo modulo deve essere separato dal LLM.

D. Document Classification Service

Responsabilità:

    classificazione del documento per tipo
    classificazione per natura contabile
    classificazione per impatto magazzino
    classificazione per workflow

Output esempio:

    tipo: FATTURA_ACQUISTO
    natura: MERCE
    impatto_magazzino: CARICO
    workflow: REVIEW_REQUIRED
    confidence: 0.87

Qui puoi usare:

    regole deterministiche
    modello ML
    Qwen 3.5 come classificatore strutturato
    ensemble finale

E. Field Extraction Service

Responsabilità:

    estrarre campi strutturati
    estrarre righe documento
    normalizzare date, importi, aliquote, quantità
    proporre match anagrafici

Campi tipici:

    fornitore
    cliente
    PIVA / CF
    numero documento
    data documento
    imponibile
    IVA
    totale
    valuta
    righe
    articoli
    quantità
    UM
    riferimenti ordine/DDT

Qui Qwen 3.5 va usato con output JSON rigidissimo.

F. RAG / Knowledge Retrieval Service

Questo non deve fare parsing del documento. Deve supportare le decisioni.

Fonti interrogate:

    piano dei conti
    causali contabili
    causali di magazzino
    tabelle IVA
    anagrafiche fornitori/clienti
    anagrafiche articoli
    policy contabili
    manuali aziendali
    esempi storici validati

Funzioni:

    retrieval semantico
    retrieval keyword
    reranking
    restituzione fonti
    explainable context

G. Accounting Rules Engine

È il cuore affidabile del sistema contabile.

Responsabilità:

    prendere classificazione + campi + contesto RAG
    applicare regole verificabili
    produrre proposta contabile

Esempi:

    fattura acquisto merce -> fornitore + conto acquisto merci + IVA a credito
    fattura servizi -> fornitore + costo servizi + IVA
    cespite -> fornitore + conto cespite + eventuale workflow cespiti

Non deve essere affidato interamente all’LLM.

H. Inventory Rules Engine

Responsabilità:

    decidere se generare carico/scarico
    scegliere causale movimento
    scegliere magazzino
    associare articolo
    validare quantità e UM

Esempi:

    DDT ingresso merce -> carico
    nota credito con reso -> scarico / rettifica
    fattura servizi -> nessun movimento magazzino

I. Proposal Composer Service

Unifica i risultati.

Produce:

    proposta contabile
    proposta IVA
    proposta magazzino
    motivazione
    elenco anomalie
    score finale

È il modulo che aggrega:

    extraction
    retrieval
    regole
    scoring

J. Validation Service

Controlli finali prima della review o del posting.

Verifiche:

    quadratura totale
    coerenza imponibile/IVA/totale
    duplicato documento
    fornitore noto
    conto ammesso
    articolo noto
    quantità coerenti
    causale ammessa
    date coerenti
    periodo contabile aperto

K. Human Review Workflow Service

Se lo score è basso o medio:

    crea task di revisione
    assegna operatore
    conserva originale e proposta
    registra correzioni
    salva motivo correzione

Questa parte è fondamentale per il miglioramento progressivo.

L. ERP Integration Service

Responsabilità:

    scrittura su ERP
    chiamata API ERP
    generazione prima nota
    generazione movimenti magazzino
    esito integrazione
    rollback / retry controllato

Mai chiamare ERP direttamente dal prompt LLM.

M. Feedback Learning Service

Responsabilità:

    raccogliere correzioni umane
    aggiornare mapping
    arricchire esempi few-shot
    migliorare casi storici per RAG
    aggiornare statistiche confidenza

1.3 Architettura fisica

Frontend

    Wisej.NET

Backend

    ASP.NET Core come piattaforma principale
    modular monolith ben separato

Pipeline asincrona

    RabbitMQ oppure
    Azure Service Bus / Kafka

DB operativo

    SQL Server 2025

oppure

    PostgreSQL se vuoi maggiore flessibilità su vector/JSON

Vector DB

    pgvector

oppure

    Milvus

Object storage

    file system gestito

LLM serving

    Qwen 3.5 self-hosted via vLLM oppure endpoint compatibile
    separa il servizio LLM dal resto

Observability

    logs strutturati
    tracing
    dashboard tempi pipeline
    audit funzionale

1.4 Flusso end-to-end

Scenario: fattura acquisto merce

    Upload PDF
    Intake salva file e genera DocumentEnvelope
    OCR/parser estrae testo e righe
    Classification dice FATTURA_ACQUISTO, natura MERCE
    Extraction prende numero, data, fornitore, totale, righe
    Retrieval consulta: fornitore piano conti causali articoli casi storici simili
    Accounting Rules Engine propone scrittura
    Inventory Rules Engine propone carico magazzino
    Validation verifica quadrature e anagrafiche
    Confidence scoring
    Se alto score -> autopost o review light
    Se medio/basso -> review umana
    ERP Integration scrive i movimenti
    Audit e feedback vengono salvati

 

1.5 Uso di Qwen 3.5

 

1. Classificazione strutturata

Output JSON vincolato:

    tipo documento
    natura
    impatto
    score
    spiegazione breve

2. Estrazione su documenti non standard

Quando parser/OCR/regole non bastano.

3. Mapping semantico descrizione -> categoria

Esempio:

    “servizio di manutenzione annuale”
    “acquisto minuteria metallica”
    “trasporto conto terzi”

4. Explainability per l’operatore

Spiegazione leggibile del perché della proposta.

Non lo userei come generatore finale autonomo di scritture definitive.

1.6 Regole di decisione operative

Autopost

Consentito solo se:

    documento riconosciuto
    quadratura OK
    anagrafica matchata
    causale univoca
    conto univoco
    nessuna anomalia bloccante
    score superiore soglia

Review obbligatoria

Se:

    OCR scarso
    documento ambiguo
    fornitore sconosciuto
    più conti plausibili
    righe articolo non matchate
    importi incoerenti
    violazioni regole

