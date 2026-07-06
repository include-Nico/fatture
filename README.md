# Gestionale Fatture

Web app statica (HTML + CSS + JavaScript vanilla, nessun framework, nessun backend)
per la gestione di fatture, anagrafica clienti/ricette, magazzino e dichiarazioni
di conformità CE per ottici. Funziona interamente lato client, offline-ready
come PWA.

Questa è una versione **unbranded** e **senza dipendenze cloud** derivata da un
progetto originale che utilizzava Google Apps Script / Google Sheets come
database remoto. In questa versione:

- ❌ **Nessuna integrazione con Google Sheets / Google Apps Script.** Tutte le
  chiamate di rete verso `script.google.com` sono state rimosse.
- ✅ **Salvataggio 100% locale** tramite `localStorage` del browser.
- ✅ **Nessun dato aziendale reale hardcoded** (nome, logo, firma, P.IVA, indirizzo
  del progetto originale sono stati rimossi). L'app parte "vuota" e i dati
  aziendali si configurano dalla nuova scheda **Impostazioni**.
- ✅ **Backup/Ripristino manuale** in formato JSON dalla scheda Impostazioni,
  per portare i dati su un altro dispositivo/browser o per non perderli.

## Struttura del progetto

```
gestionale-fatture/
├── index.html          # Struttura dell'app (5 schede + Impostazioni)
├── manifest.json        # Manifest PWA (installabile su mobile/desktop)
├── css/
│   └── style.css        # Stile completo (tema chiaro/scuro incluso)
├── js/
│   └── script.js        # Logica applicativa, nessuna chiamata di rete
└── assets/
    └── logo-placeholder.svg   # Icona generica di default
```

## Come si usa

1. Apri `index.html` in un browser moderno (o pubblica la cartella su un
   qualsiasi hosting statico: GitHub Pages, Netlify, Vercel, un semplice
   server web, ecc. — non serve alcun backend).
2. Vai nella scheda **Impostazioni** e inserisci i dati della tua attività
   (ragione sociale, P.IVA, C.F., indirizzo, email, telefono) e, se vuoi,
   carica il tuo logo e la tua firma: verranno stampati automaticamente
   sulle fatture e sulle dichiarazioni di conformità generate in PDF.
3. Usa le altre schede per gestire clienti/ricette, magazzino, generare
   fatture (con numerazione automatica per anno) e dichiarazioni di
   conformità CE.
4. Esporta periodicamente un backup JSON dalla scheda Impostazioni: è
   l'unico modo per conservare i dati se cambi browser/dispositivo o
   cancelli i dati di navigazione.

## Note importanti

- I dati (clienti, fatture, magazzino, impostazioni, logo, firma) sono
  salvati **solo nel browser che stai usando** (localStorage). Cancellare i
  dati di navigazione del browser cancella anche questi dati: esporta un
  backup regolarmente.
- Le funzionalità relative a montature/lenti/ricette optometriche e alla
  dichiarazione di conformità CE sono pensate per attività di ottica; puoi
  comunque usare la sola parte "Fatture" per qualsiasi altro tipo di
  attività in regime forfettario/con IVA esente, personalizzando i testi
  in Impostazioni.
- Questo progetto non effettua alcuna trasmissione dei dati a server esterni:
  le uniche risorse esterne caricate sono le librerie pubbliche (Font
  Awesome, html2pdf.js, Chart.js) via CDN, necessarie per icone, generazione
  PDF e grafici.
