# Don Basilico — Riepilogo Giornaliero
**Versione:** 1.0 — **Ultimo aggiornamento:** 19/05/2026
**URL produzione:** https://don-basilico-riepilogo.vercel.app
**Repository:** https://github.com/simonefloriano79-spec/don-basilico-riepilogo

---

## Stack

| Layer | Tecnologia |
|-------|-----------|
| Frontend | HTML5 + Vanilla JS — file unico `index.html` |
| CSS | Inline nel `<style>`, nessun framework |
| Database | Supabase (PostgreSQL) — progetto `don-basilico-crm` |
| Hosting | Vercel — deploy automatico su push a `main` |
| Auth | PIN a 4 cifre verificato via query REST su `public.sedi_pin` |
| PWA | `manifest.json` + `sw.js` (Service Worker) |

Il client Supabase JS è stato rimosso. Tutta la comunicazione con il DB avviene tramite `fetch()` dirette alle REST API di Supabase.

---

## Struttura del progetto

```
don-basilico-riepilogo/
├── index.html        # Intera applicazione (HTML + CSS + JS)
├── manifest.json     # Configurazione PWA
├── sw.js             # Service Worker
├── icon-192.png      # Icona PWA
├── icon-512.png      # Icona PWA
├── vercel.json       # Rewrite: tutto → index.html
└── CLAUDE.md         # Questo file
```

Nessun build step. Nessun `package.json`. Nessun bundler. File HTML statico.

---

## Database

**Project ID:** `wgvuhjfszfzmzlhvizph`
**URL:** `https://wgvuhjfszfzmzlhvizph.supabase.co`
**Schema:** `public` — lo schema `riepiloghi` è abbandonato, non usarlo.

### Tabella `public.riepilogo_daily`

| Colonna | Tipo | Note |
|---------|------|------|
| id | uuid | PK auto-generato |
| pizzeria | text | Nome completo sede (es. "Don Basilico - Chieti") |
| date | date | UNIQUE con pizzeria |
| fc | numeric | Fondo Cassa — riferimento cassetto, **non è una spesa** |
| ragazzi | numeric | Compenso ragazzi — **è una spesa**, si somma al totale |
| pos_domicilio | numeric | |
| pos_just_eat | numeric | |
| pos_pizzeria | numeric | |
| annullati | numeric | |
| consegne_just_eat | integer | |
| consegne_pizzeria | integer | |
| pizze_pizzeria | integer | Nel form: "Pizze Consegnate" |
| pizze_just_eat | integer | |
| pizze_maxi | integer | |
| pizze_baby | integer | |
| pizze_piatto | integer | Totale pizze giornaliero (campo calcolato) |
| pizze_sala | integer | Nel form: "QUI (Pizze in Sala)" |
| comande_pv | integer | |
| num_pizze_pv | integer | |
| incasso | numeric | Incasso pizzeria |
| incasso_just_eat | numeric | |
| chiusura_fiscale | numeric | |
| importo_verde | numeric | |
| punto_pizza | numeric | |
| punto_just_eat | numeric | |
| banconote | numeric | |
| monete | numeric | |
| pac | numeric | Porto a Casa |
| spese | numeric | Totale spese (include ragazzi) |
| spese_dettaglio | jsonb | Array `[{cat, val, desc?}]` |
| note | text | |
| created_at | timestamptz | |
| updated_at | timestamptz | Non aggiornato automaticamente — manca il trigger |

RLS disabilitato. Sicurezza gestita via PIN.

### Tabella `public.sedi_pin`

| Colonna | Tipo | Note |
|---------|------|------|
| id | text | PK (es. "pescara_centro") |
| pin | text | 4 cifre |

RLS disabilitato. GRANT SELECT TO anon.
I PIN non sono nel codice sorgente — la verifica avviene tramite query su questa tabella.

### Nomi sedi (campo `pizzeria`)

```
Don Basilico - Pescara Centro
Don Basilico - Centro Marconi
Don Basilico - Montesilvano
Don Basilico - Chieti
Don Basilico - Università
Don Basilico - Pescara Nord
```

---

## Autenticazione

PIN a 4 cifre. Flusso:
1. L'utente seleziona la sede e inserisce il PIN
2. L'app interroga `sedi_pin` via REST: `id=eq.{sede}&pin=eq.{pin}`
3. Risposta non vuota → accesso consentito
4. Sessione in-memory — nessun token, nessun cookie

Admin: `id = 'admin'` in `sedi_pin`.

---

## Funzionalità

### Tab Stats
Selezione periodo: 7 giorni / Mese / 3 mesi / Anno / Tutto / Custom.

KPI: Incasso Totale, Media/Giorno, Incasso Pizzeria, Incasso Just Eat, Pizze Totali, Valore Medio Pizza, B/N Medio, Spese Totali.

Grafici SVG (nessuna libreria): incasso nel tempo, pizze nel tempo (Pizzeria vs JE), confronto sedi con barre, andamento sedi nel tempo.

Spese per categoria — la voce "Altro" mostra la descrizione libera inserita dal gestore (es. "Altro: Penny").

Confronto sedi e grafici multipli visibili solo da Admin.

### Tab Storico
**Vista Lista:** paginata (10 per pagina), ogni riga espandibile con dettaglio completo. Click su una riga → carica il record nel form. Filtro per sede (solo Admin).

**Vista Calendario:** giorni colorati — verde (inserito), rosso (mancante), grigio (futuro). Click su verde → apre il record. Click su rosso → apre il form con data preimpostata. Navigazione mensile.

### Tab Inserisci

**Sede** (solo Admin): al cambio sede, il form carica i dati esistenti per quella sede + data oppure si azzera.

**Personale:** FC (Fondo Cassa), Ragazzi.

**POS:** POS Domicilio, POS Just Eat, POS Pizzeria, Annullati. Tutti decimali. Se più ragazzi escono con POS separati, la somma va fatta manualmente prima di inserire.

**Produzione:** Pizze Maxi, Pizze Baby, Consegne Just Eat, Consegne Pizzeria, Pizze Consegnate, Pizze Just Eat, Comande PV, Num Pizze PV, QUI (Pizze in Sala).
Totale Pizze Giornaliero — campo non editabile, calcolato in tempo reale:
```
Pizze JE + Pizze Consegnate + Num Pizze PV + Pizze Baby + (Pizze Maxi × 2) + QUI
```

**Cassa:** Importo Verde, Punto Pizza, Punto Just Eat, Incasso, Incasso Just Eat, Chiusura Fiscale, Banconote, Monete, PAC.
Valore Medio Pizza e B/N — campi non editabili, calcolati in tempo reale:
```
Valore Medio Pizza = (Incasso + Incasso JE) / Totale Pizze Giornaliero
B/N = |Incasso - Chiusura Fiscale| / Chiusura Fiscale × 100
```
B/N verde se < 30%, rosso se ≥ 30%.

**Spese:** sezione dinamica. Categorie: Natural, Tutto Carta, Bevande, Verdura, Fritti, Salsiccia, Stipendi, Contado, Mozzarella, Altro. Selezionando "Altro" compare un campo testo libero. Totale Spese = somma voci + Ragazzi. Salvato in `spese_dettaglio` come JSONB.

**Blocco 48 ore:** i gestori non possono modificare record il cui `created_at` è più vecchio di 48 ore. Admin non ha questo limite.

### Tab Ordini
Fornitori: Natural, Tutto Carta, Bevande (Vicentini), Verdure, Fritti, Salsiccia. Ogni fornitore ha prodotti con unità di misura. Quantità con − / +. Note per fornitore. Il pulsante WhatsApp apre `wa.me/{numero}?text={messaggio}` con il testo preformattato. Gli ordini non vengono salvati nel DB.

### Tab Admin (solo Admin)
- Stato inserimenti del giorno per ogni sede
- Griglia ultimi 7 giorni × tutte le sedi — click su cella per aprire il record
- Ultimi 20 inserimenti
- PIN sedi: Admin li vede tutti (nascosti, con pulsante "Mostra PIN"). Gestore vede solo il proprio.

---

## Costanti nel codice

```js
const SUPA_URL = 'https://wgvuhjfszfzmzlhvizph.supabase.co';
const SUPA_KEY = 'eyJhbGciOiJIUzI1NiI...'; // anon key pubblica
```

L'anon key è pubblica per design. La sicurezza dipende dal meccanismo PIN, non dalla segretezza della chiave.

---

## Vincoli tecnici critici

**1. Fetch dirette — non il client Supabase JS.**
Il client causava `Invalid schema: riepiloghi`. Usare `fetch()` con headers `apikey` e `Authorization: Bearer`.

**2. Nessun template literal annidato.**
Causano `SyntaxError: Unexpected string`. Usare concatenazione con `+` e `JSON.stringify()` per i valori variabili negli attributi HTML.

**3. Upsert = DELETE poi INSERT.**
L'upsert nativo REST non funzionava. Pattern corretto:
```js
// DELETE
fetch(`${SUPA_URL}/rest/v1/riepilogo_daily?pizzeria=eq.${encodeURIComponent(p)}&date=eq.${d}`, { method: 'DELETE', headers })
// INSERT
fetch(`${SUPA_URL}/rest/v1/riepilogo_daily`, { method: 'POST', headers: {...headers, 'Prefer':'return=minimal'}, body: JSON.stringify(row) })
```

**4. Parsing numerico.**
Usare `parseNum()` (custom) invece di `parseFloat()`. Converte virgola in punto per compatibilità con tastiere Android.

**5. Calcoli differiti.**
`calcTotale()` e `updateSpeseTotal()` vanno chiamati dentro `setTimeout(..., 300)` dopo il popolamento del form.

**6. Schema.**
Schema: `public`. Tabella: `riepilogo_daily`. Lo schema `riepiloghi` è abbandonato.

**7. Blocco 48h.**
Basato su `created_at` del record, non sulla `date` del riepilogo.

---

## Operazioni di manutenzione

**Modificare un PIN:**
```sql
UPDATE public.sedi_pin SET pin = 'XXXX' WHERE id = 'nome_sede';
```

**Aggiungere una sede:**
1. Voce nell'array `SEDI` in `index.html`
2. Voce nell'oggetto `DB_NAME`
3. Riga in `public.sedi_pin`

**Aggiungere un campo al form:**
1. `ALTER TABLE public.riepilogo_daily ADD COLUMN IF NOT EXISTS nome tipo DEFAULT 0;`
2. Input in `renderForm()`
3. Valore nell'oggetto `row` in `saveForm()`
4. Restore in `loadForEdit()` e `onSedeChange()`
5. Se influisce sui totali: aggiornare `calcTotale()` o `updateSpeseTotal()`

**Deploy:**
```bash
git add . && git commit -m "descrizione" && git push origin main
```
Vercel fa il deploy automaticamente. Framework: "Other".

---

## Backlog

**Alta priorità**
- Numeri WhatsApp reali dei fornitori (ora ci sono numeri di test)
- Trigger DB per `updated_at`
- Export PDF / Excel riepilogo mensile per sede

**Media priorità**
- Storico ordini fornitori nel DB
- Completare prodotti fornitore Salsiccia
- Report settimanale via email

**Bassa priorità**
- Notifiche push per inserimenti mancanti
- Food cost per prodotto
- Inventario materie prime
