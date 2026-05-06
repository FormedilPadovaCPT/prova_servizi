# Guida Integrazione: Notizie + Push Notifications
## Portale Formedil Padova – Area Sicurezza e Salute

---

## Cosa è stato modificato

| File | Modifiche |
|------|-----------|
| `index.html` | Card "Notizie" in evidenza (grigio/arancio), pagina notizie, JS notifiche |
| `sw.js` | Handler push, notificationclick, messaggio in-app |
| `GAS_Notizie_PushNotifiche.gs` | **NUOVO** — funzioni GAS per notizie e push |

---

## STEP 1 — Foglio Google Sheet "Notizie"

Aprire il foglio `1qo3Qh8qk76bYKMrHdz-rSapQvbFhNrM5QEOCwntkYTA` e creare due nuovi fogli:

### Foglio: `Notizie`

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| ID | DATA | TITOLO | TESTO | CATEGORIA | LINK | ATTIVA |
| 1 | 07/05/2025 | Prova notizia | Testo della notizia | avviso | (vuoto) | SI |

- **ID**: numero progressivo univoco
- **DATA**: data di pubblicazione (formato data)
- **TITOLO**: titolo breve
- **TESTO**: testo completo (supporta `\n` per a capo)
- **CATEGORIA**: `normativa` | `formazione` | `cantiere` | `avviso` | `generale`
- **LINK**: URL opzionale (es. a un PDF o pagina web)
- **ATTIVA**: scrivere `SI` per pubblicare, qualsiasi altro valore per nascondere

### Foglio: `Push_Subscribers`

Viene creato automaticamente dal GAS al primo iscritto.
Se si vuole crearlo manualmente:

| A | B | C |
|---|---|---|
| SUBSCRIPTION_JSON | ENDPOINT | REGISTRATA_IL |

---

## STEP 2 — Modificare il doGet() e doPost() nel GAS

Aprire `Code_Portale_Formedil.gs` e modificare le funzioni `doGet` e `doPost`:

### In `doGet(e)` — aggiungere PRIMA del return finale:

```javascript
// ── NOTIZIE ────────────────────────────────────────────
const action = e.parameter.action || '';
if (action === 'get_news') {
  return ContentService
    .createTextOutput(JSON.stringify(getNews()))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### In `doPost(e)` — aggiungere PRIMA della lettura del payload esistente:

```javascript
// ── PUSH SUBSCRIPTIONS ────────────────────────────────
const action = e.parameter.action || '';

if (action === 'sub_push') {
  const sub = JSON.parse(e.postData.contents);
  savePushSubscription(sub);
  return ContentService
    .createTextOutput(JSON.stringify({ ok: true }))
    .setMimeType(ContentService.MimeType.JSON);
}

if (action === 'unsub_push') {
  const { endpoint } = JSON.parse(e.postData.contents);
  removePushSubscription(endpoint);
  return ContentService
    .createTextOutput(JSON.stringify({ ok: true }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

---

## STEP 3 — Generare le chiavi VAPID

Le chiavi VAPID identificano il tuo server come mittente autorizzato.
**Si generano una sola volta e non si cambiano più** (cambiarle invalida tutte le subscription esistenti).

### Metodo online (più semplice):
1. Vai su **https://vapidkeys.com/**
2. Clicca "Generate VAPID Keys"
3. Copia `Public Key` e `Private Key`

### Metodo da terminale:
```bash
npx web-push generate-vapid-keys
```

### Dove inserire le chiavi:

**In `index.html`** (riga con `VAPID_PUBLIC_KEY`):
```javascript
const VAPID_PUBLIC_KEY = 'BHv2...la_tua_chiave_pubblica...'; // ~87 caratteri
```

**In `GAS_Notizie_PushNotifiche.gs`**:
```javascript
const CFG_PUSH = {
  VAPID_PUBLIC_KEY:  'BHv2...la_tua_chiave_pubblica...',
  VAPID_PRIVATE_KEY: 'abc1...la_tua_chiave_privata...',  // ~43 caratteri
  ...
};
```

---

## STEP 4 — Configurare l'invio Push (Cloudflare Workers)

Google Apps Script non può firmare le notifiche VAPID direttamente.
La soluzione raccomandata è un **Cloudflare Worker gratuito** (100.000 req/giorno).

### 4a. Creare il Worker

1. Registrarsi su https://cloudflare.com (gratuito)
2. Dashboard → Workers → Create Worker
3. Incollare questo codice:

```javascript
import webpush from 'web-push';

export default {
  async fetch(request, env) {
    if (request.method !== 'POST') return new Response('Method not allowed', { status: 405 });

    webpush.setVapidDetails(
      'mailto:cpt@formedilpadova.it',
      env.VAPID_PUBLIC_KEY,
      env.VAPID_PRIVATE_KEY
    );

    const { subscription, payload } = await request.json();
    try {
      await webpush.sendNotification(subscription, payload);
      return new Response(JSON.stringify({ ok: true }), { headers: { 'Content-Type': 'application/json' } });
    } catch (e) {
      return new Response(JSON.stringify({ error: e.message, statusCode: e.statusCode }), {
        status: e.statusCode || 500,
        headers: { 'Content-Type': 'application/json' }
      });
    }
  }
};
```

4. Nelle impostazioni del Worker → Variables → aggiungere:
   - `VAPID_PUBLIC_KEY` = la chiave pubblica
   - `VAPID_PRIVATE_KEY` = la chiave privata (Secret)

5. Copiare l'URL del Worker (es. `https://formedil-push.USERNAME.workers.dev`)

### 4b. Aggiornare il GAS

In `GAS_Notizie_PushNotifiche.gs`, nella funzione `sendWebPush`, decommentare l'OPZIONE A e inserire l'URL del Worker:

```javascript
const RELAY_URL = 'https://formedil-push.USERNAME.workers.dev/send';
const resp = UrlFetchApp.fetch(RELAY_URL, {
  method: 'post',
  contentType: 'application/json',
  payload: JSON.stringify({ subscription, payload: payloadJson }),
  muteHttpExceptions: true,
});
return resp.getResponseCode();
```

---

## STEP 5 — Ridistribuire il GAS

Dopo aver modificato il codice GAS:
1. GAS Editor → Distribuisci → Gestisci distribuzioni
2. Clic sull'icona modifica (matita)
3. Versione → "Nuova versione"
4. Distribuisci

> **Nota:** Se l'URL del GAS non cambia (di solito non cambia), non è necessario aggiornare `GAS_URL` in `index.html`.

---

## STEP 6 — Caricare i file su GitHub

Sostituire i file modificati:

```
/servizi/
  ├── index.html          ← versione aggiornata (index_notizie.html)
  └── PWA/
      └── sw.js           ← versione aggiornata (sw_notizie.js)
```

---

## Come pubblicare una notizia

### Metodo 1 (manuale dal foglio):
1. Aprire il foglio "Notizie" nel Google Sheet
2. Aggiungere una riga con i dati
3. Nella colonna G scrivere `SI`
4. Se è configurato il trigger `onNewsSheetEdit`, la push parte automaticamente
5. Altrimenti: GAS Editor → eseguire `sendPushNotification('Titolo', 'Testo')`

### Metodo 2 (trigger automatico):
- GAS Editor → Triggers → + Aggiungi trigger
- Funzione: `onNewsSheetEdit`
- Tipo evento: Da foglio di lavoro → Al modifica
- La notifica parte appena si scrive `SI` nella colonna ATTIVA

---

## Come appare all'utente

**Card home:** la card "Notizie & Aggiornamenti" ha sfondo grigio scuro con bordo arancio,
visivamente distinta da tutte le altre card. Un badge rosso pulsante mostra il numero di
notizie non lette.

**Pagina notizie:** lista cronologica con tag categoria colorati, data, titolo e testo.
Le notizie non ancora cliccate appaiono con bordo arancio a sinistra.

**Notifica push:** appare come una normale notifica di sistema sul telefono, con
icona Formedil, titolo e testo breve. Al clic apre la pagina notizie della PWA.

---

## Test locale

1. Aprire Chrome → DevTools → Application → Service Workers
2. Verificare che il nuovo `sw.js` sia registrato (versione v2)
3. Application → Push → inviare un payload di test:
   ```json
   {"title":"Test Formedil","body":"Notizia di prova","tag":"test"}
   ```
4. Verificare che la notifica appaia e che al clic si apra la pagina notizie
