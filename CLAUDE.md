# CLAUDE.md — Don Basilico Riepilogo Giornaliero
> Documento tecnico-operativo per Claude Code. Aggiornato al 19/05/2026.

---

## 1. STACK TECNICO

| Layer | Tecnologia |
|-------|-----------|
| Frontend | HTML5 puro + Vanilla JS (no framework) |
| Styling | CSS inline + classi custom nel `<style>` |
| Database | Supabase (PostgreSQL) — progetto `don-basilico-crm` |
| Hosting | Vercel (deploy automatico da GitHub) |
| Repository | `https://github.com/simonefloriano79-spec/don-basilico-riepilogo` |
| PWA | manifest.json + Service Worker (sw.js) |
| Auth | PIN numerico a 4 cifre verificato lato server (tabella `sedi_pin`) |

### Librerie esterne (CDN)
- Nessuna — tutto vanilla JS
- Supabase JS SDK **rimosso** (sostituito con fetch REST API dirette)

### Comunicazione con Supabase
Tutte le query usano la REST API di Supabase direttamente via `fetch()`:
```js
// SELECT
fetch(`${SUPA_URL}/rest/v1/riepilogo_daily?select=*&...`, {
  headers: { 'apikey': SUPA_KEY, 'Authorization': 'Bearer '+SUPA_KEY }
})
// INSERT
fetch(`${SUPA_URL}/rest/v1/riepilogo_daily`, {
  method: 'POST',
  headers: { 'apikey': SUPA_KEY, 'Authorization': 'Bearer '+SUPA_KEY, 'Content-Type': 'application/json', 'Prefer': 'return=minimal' },
  body: JSON.stringify(row)
})
```
**NON usare il client Supabase JS** — causava errori di schema. Usare sempre fetch dirette.

---

## 2. FUNZIONALITÀ IMPLEMENTATE (PRODUZIONE)

### 🔐 Autenticazione PIN
- Schermata di selezione sede (6 PDV + Admin)
- Inserimento PIN da tastierino numerico
- Verifica PIN lato server (tabella `public.sedi_pin`) — i PIN **NON** sono nel codice sorgente
- Sessione in-memory (nessun cookie/localStorage)

### 📊 Tab Stats
- Selezione periodo: 7gg / Mese / 3 mesi / Anno / Tutto / Custom (date picker)
- KPI: Incasso Totale, Media/Giorno, Incasso Pizzeria, Incasso Just Eat, Pizze Totali, Valore Medio Pizza, B/N Medio, Spese Totali
- Grafico SVG incasso nel tempo
- Grafico SVG pizze nel tempo (Pizzeria vs Just Eat)
- Dettaglio pizze (maxi, baby, consegne, comande PV)
- Spese per categoria con barre proporzionali (include descrizioni "Altro: xxx")
- Confronto sedi con barre colorate (solo Admin)
- Grafico confronto sedi nel tempo (solo Admin)
- Ultimi inserimenti con B/N colorato (verde <30%, rosso >=30%)

### 📋 Tab Storico
- **Vista Lista**: tutte le giornate paginate (10/pagina), espandibili con dettaglio
- **Vista Calendario**: giorni colorati (verde=inserito, rosso=mancante, grigio=futuro)
- Navigazione mese per mese nel calendario
- Riepilogo mensile (giorni inseriti, mancanti, incasso mese)
- Filtro per sede (solo Admin) tramite menu a tendina
- Click su giornata → carica dati nel form per modifica
- Click su giorno rosso nel calendario → apre form con data preimpostata

### 📝 Tab Inserisci
**Sezione SEDE** (solo Admin): menu a tendina per selezionare la sede
- Al cambio sede: carica automaticamente i dati esistenti per quella sede+data (o svuota se nessun dato)

**Sezione DATA**: date picker

**Sezione PERSONALE**:
- FC (Fondo Cassa) — numerico decimale, riferimento cassetto, NON è una spesa
- Ragazzi — importo pagato ai ragazzi, SI somma alle spese totali

**Sezione POS**:
- POS Domicilio, POS Just Eat, POS Pizzeria, Annullati — tutti decimali
- Nota: POS Domicilio può avere più valori (un POS per ragazzo) — sommare manualmente

**Sezione PRODUZIONE**:
- Pizze Maxi, Pizze Baby
- Consegne Just Eat, Consegne Pizzeria
- Pizze Consegnate (= Pizze Pizzeria nel DB), Pizze Just Eat
- Comande PV, Num Pizze PV
- QUI (Pizze in Sala)
- **TOTALE PIZZE GIORNALIERO** — campo auto-calcolato, non editabile
  - Formula: `Pizze JE + Pizze Consegnate + Num Pizze PV + Pizze Baby + (Pizze Maxi × 2) + QUI`

**Sezione CASSA**:
- Importo Verde, Punto Pizza, Punto Just Eat
- Incasso, Incasso Just Eat
- Chiusura Fiscale
- Banconote, Monete, PAC (Porto a Casa)
- **VALORE MEDIO PIZZA** — auto-calcolato: `(Incasso + Incasso JE) / Totale Pizze Giornaliero`
- **B/N** — auto-calcolato: `|Incasso - Chiusura Fiscale| / Chiusura Fiscale × 100`
  - Verde se < 30%, Rosso se >= 30%

**Sezione SPESE** (dinamica):
- Pulsante "+ Aggiungi Spesa"
- Menu a tendina: Natural, Tutto Carta, Bevande, Verdura, Fritti, Salsiccia, Stipendi, Contado, Mozzarella, Altro
- Se si seleziona "Altro": appare campo testo descrizione libera
- TOTALE SPESE = somma voci spese + Ragazzi
- Spese salvate come JSONB in `spese_dettaglio` (con campo `desc` per Altro)

**Sezione NOTE**: campo testo libero

**Blocco 48 ore**: i gestori non possono modificare record salvati da più di 48 ore. Solo Admin può modificare qualsiasi record.

### 🛒 Tab Ordini
- 6 fornitori: Natural, Tutto Carta, Bevande (Vicentini), Verdure, Fritti, Salsiccia
- Prodotti per fornitore con unità di misura
- Quantità con pulsanti − / +
- Campo Note per fornitore
- Riepilogo ordine multi-fornitore
- Bottone WhatsApp per ogni fornitore → apre WA con messaggio preformattato
  - Messaggio: `"Ciao, ordine per [SEDE]:\n- Prodotto: quantità um\n\n📝 Note: ...\n\nGrazie!"`
- Numeri WhatsApp associati ai fornitori (attualmente numeri di test)

### ⚙️ Tab Admin (solo Admin)
- Stato inserimenti OGGI per ogni sede (✅ Inserito / ❌ Mancante)
- Griglia ultimi 7 giorni × tutte le sedi (click su ✓ per aprire il record)
- Ultimi 20 inserimenti (click per modificare)
- Sezione PIN Sedi:
  - Admin: vede tutti i PIN (nascosti, rivelabili con "👁 Mostra PIN")
  - Gestore: vede solo il proprio PIN

### 📲 PWA
- Installabile su Android via pulsante "📲 Installa App"
- Installabile su iPhone via banner con istruzioni manuali (appare dopo 2s)
- Icone 192×192 e 512×512 (logo Don Basilico)
- Service Worker per funzionamento offline base

---

## 3. STRUTTURA DEL PROGETTO

```
don-basilico-riepilogo/
├── index.html          # TUTTA l'app (HTML + CSS + JS in un unico file)
├── manifest.json       # PWA manifest
├── sw.js               # Service Worker
├── icon-192.png        # Icona PWA 192×192
├── icon-512.png        # Icona PWA 512×512
├── vercel.json         # Config Vercel (rewrite tutto → index.html)
└── CLAUDE.md           # Questo file
```

**IMPORTANTE**: tutta la logica è in `index.html`. Non ci sono componenti separati, build steps, o package.json. È un file HTML statico puro.

---

## 4. DATABASE SUPABASE

### Progetto
- **Project ID**: `wgvuhjfszfzmzlhvizph`
- **URL**: `https://wgvuhjfszfzmzlhvizph.supabase.co`
- **Schema usato**: `public` (NON `riepiloghi` — quello era il vecchio schema, abbandonato)

### Tabelle

#### `public.riepilogo_daily`
| Colonna | Tipo | Note |
|---------|------|------|
| id | uuid | PK, auto |
| pizzeria | text | Nome completo es. "Don Basilico - Chieti" |
| date | date | UNIQUE con pizzeria |
| fc | numeric | Fondo Cassa (NON spesa) |
| ragazzi | numeric | Pagamento ragazzi (È spesa) |
| pos_domicilio | numeric | |
| pos_just_eat | numeric | |
| pos_pizzeria | numeric | |
| annullati | numeric | |
| consegne_just_eat | integer | |
| consegne_pizzeria | integer | |
| pizze_pizzeria | integer | = "Pizze Consegnate" nel form |
| pizze_just_eat | integer | |
| pizze_maxi | integer | |
| pizze_baby | integer | |
| pizze_piatto | integer | = TOTALE PIZZE GIORNALIERO (calcolato) |
| pizze_sala | integer | = QUI (pizze in sala) |
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
| spese | numeric | Totale spese (inclusi ragazzi) |
| spese_dettaglio | jsonb | Array [{cat, val, desc?}] |
| note | text | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

**UNIQUE constraint**: `(pizzeria, date)`
**RLS**: DISABILITATO (app interna)

#### `public.sedi_pin`
| Colonna | Tipo | Note |
|---------|------|------|
| id | text | PK (es. "pescara_centro") |
| pin | text | 4 cifre |

**RLS**: DISABILITATO
**GRANT**: SELECT TO anon

### Nomi Sedi nel DB (campo `pizzeria`)
```
"Don Basilico - Pescara Centro"
"Don Basilico - Centro Marconi"
"Don Basilico - Montesilvano"
"Don Basilico - Chieti"
"Don Basilico - Università"
"Don Basilico - Pescara Nord"
```

---

## 5. VARIABILI / COSTANTI NEL CODICE

Non ci sono variabili d'ambiente (app statica). Le costanti sono hardcoded in `index.html`:

```js
const SUPA_URL  = 'https://wgvuhjfszfzmzlhvizph.supabase.co';
const SUPA_KEY  = 'eyJhbGciOiJIUzI1NiI...'; // anon key pubblica
```

La `anon key` è pubblica per design (app client-side). La sicurezza è garantita da:
1. RLS disabilitato ma accesso solo da chi conosce i PIN
2. PIN verificati server-side (non nel sorgente)

---

## 6. FLUSSO OPERATIVO QUOTIDIANO

### Gestore (ogni sera)
1. Apre `https://don-basilico-riepilogo.vercel.app`
2. Seleziona la propria sede → inserisce PIN
3. Va su **Inserisci**
4. Compila tutti i campi del form
5. Aggiunge le spese del giorno con "+ Aggiungi Spesa"
6. Preme **Salva Riepilogo**
7. Può modificare entro 48 ore dal salvataggio

### Admin (Simone, ogni giorno)
1. Accede con PIN `5943`
2. Tab **Admin** → controlla chi ha inserito oggi (✅/❌)
3. Tab **Storico** → filtra per sede → verifica dati
4. Se serve modificare un vecchio record → click sul record → modifica → salva
5. Tab **Stats** → analisi performance

### Ordini (quando necessario)
1. Qualsiasi sede accede con il proprio PIN
2. Tab **Ordini** → seleziona fornitore → inserisce quantità → aggiunge note
3. Preme **📱 WhatsApp [Fornitore]** → si apre WA con messaggio preformattato
4. Invia il messaggio al rappresentante

---

## 7. INTEGRAZIONI

| Servizio | Uso | Note |
|----------|-----|------|
| Supabase | Database PostgreSQL | Progetto `don-basilico-crm`, schema `public` |
| Vercel | Hosting + deploy | Auto-deploy da push su `main` |
| GitHub | Repository | `simonefloriano79-spec/don-basilico-riepilogo` |
| WhatsApp | Ordini fornitori | Via link `wa.me/NUMERO?text=MESSAGGIO` |

**NON integrato** (ancora):
- Google Sheets
- Email/notifiche automatiche
- WhatsApp Business API (a pagamento)

---

## 8. TODO / FUNZIONALITÀ MANCANTI

### Alta priorità
- [ ] Numeri WhatsApp reali dei fornitori (ora ci sono numeri di test)
- [ ] Export PDF/Excel del riepilogo mensile per sede
- [ ] Notifica automatica se un gestore non inserisce entro le 23:00
- [ ] Aggiornare `updated_at` automaticamente via trigger DB

### Media priorità
- [ ] Aggiunta prodotti mancanti per fornitore Salsiccia
- [ ] Storico ordini fornitori salvato nel DB
- [ ] Report settimanale automatico via email
- [ ] Grafici più avanzati (bar chart mensile, trend)

### Bassa priorità
- [ ] Gestione turni personale
- [ ] Food cost per prodotto
- [ ] Inventario materie prime
- [ ] Notifiche push (richiede backend)

---

## 9. NOTE OPERATIVE — COSE DA NON ROMPERE

### ⚠️ Regole critiche

1. **NON usare il client Supabase JS SDK** — causava `Invalid schema: riepiloghi`. Usare sempre `fetch()` dirette con headers `apikey` e `Authorization`.

2. **NON mettere PIN nel codice sorgente** — i PIN sono nella tabella `sedi_pin`. La verifica avviene via query REST.

3. **NON usare template literals annidati** nel JS — causano `SyntaxError: Unexpected string`. Usare concatenazione di stringhe con `+` o `JSON.stringify()` per valori variabili negli attributi HTML.

4. **Lo schema è `public`** — NON `riepiloghi`. La tabella si chiama `riepilogo_daily`, NON `daily`.

5. **Upsert = DELETE + INSERT** — l'upsert nativo di Supabase via REST non funziona correttamente. Il pattern corretto è:
   ```js
   // Prima DELETE
   fetch(`${SUPA_URL}/rest/v1/riepilogo_daily?pizzeria=eq.X&date=eq.Y`, {method:'DELETE', headers})
   // Poi INSERT
   fetch(`${SUPA_URL}/rest/v1/riepilogo_daily`, {method:'POST', headers, body: JSON.stringify(row)})
   ```

6. **Decimali**: usare sempre `parseNum()` (funzione custom che converte virgola→punto) invece di `parseFloat()` per i campi numerici degli utenti.

7. **calcTotale()** deve essere chiamato con `setTimeout(..., 300)` dopo il caricamento dei dati — i campi del DOM devono essere popolati prima del calcolo.

8. **Blocco 48h**: basato su `created_at` del record, NON sulla `date` del riepilogo.

### 🔧 Come modificare i PIN
```sql
UPDATE public.sedi_pin SET pin = 'NUOVO_PIN' WHERE id = 'nome_sede';
-- es: UPDATE public.sedi_pin SET pin = '1234' WHERE id = 'chieti';
```

### 🔧 Come aggiungere una sede
1. Aggiungere in `SEDI` array in `index.html`
2. Aggiungere in `DB_NAME` object
3. Aggiungere in `public.sedi_pin`
4. Aggiungere nei FORNITORI se necessario

### 🔧 Come aggiungere un campo al form
1. Aggiungere colonna in Supabase: `ALTER TABLE public.riepilogo_daily ADD COLUMN IF NOT EXISTS nome_campo tipo DEFAULT 0;`
2. Aggiungere input HTML in `renderForm()`
3. Aggiungere il valore in `saveForm()` nell'oggetto `row`
4. Aggiungere in `loadForEdit()` per il caricamento
5. Se campo calcolato: aggiungere in `calcTotale()` o `updateSpeseTotal()`

### 🔧 Deploy
Ogni `git push origin main` triggera automaticamente il deploy su Vercel.
L'app è live su `https://don-basilico-riepilogo.vercel.app`.

---

## 10. ARCHITETTURA DECISIONI (ADR)

| Decisione | Motivo |
|-----------|--------|
| HTML puro (no React/Vue) | Massima compatibilità mobile, zero build step, deploy istantaneo |
| Supabase anon key nel client | App interna, sicurezza gestita via PIN server-side |
| PIN nel DB (non nel codice) | Sicurezza: non visibili in F12/view-source |
| DELETE+INSERT invece di UPSERT | Il UPSERT REST di Supabase con conflict resolution non funzionava |
| Schema `public` (non `riepiloghi`) | Lo schema custom non era esposto via REST API |
| Tutto in index.html | Semplicità massima, nessun bundler, modificabile direttamente |
| Vercel framework: "Other" | File HTML statico, non Next.js |
