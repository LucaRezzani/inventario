# Inventario

Inventario personale in **un solo file HTML**. Niente framework, niente `npm`, niente
compilazione: apri `index.html` e funziona.

I dati stanno in `localStorage`, cioè nella memoria del browser che stai usando. Nessun
account, nessun server, nessun database condiviso.

---

## Cosa fa

- **Categorie e oggetti**, ordinati alfabeticamente all'apertura (ordinamento italiano:
  «àncora» accanto ad «Ancora», e «Scaffale 10» dopo «Scaffale 2»).
- **Scorta minima facoltativa.** Se non la specifichi resta `null` e l'oggetto non entra
  mai in allarme. Se c'è, la riga diventa rossa quando `quantità <= minimo`.
- **Barra di livello** sugli oggetti con soglia: riempimento proporzionale e una tacca
  sul valore minimo.
- **Pulsante Aggiungi** in basso a destra: oggetto singolo, categoria, oppure lista.
- **Ricerca** su tutto l'inventario.
- **Esporta / importa JSON** per il backup e per spostare i dati su un altro dispositivo.
- **Installabile** sulla home del telefono, e funziona offline.

### Il formato della lista

Una riga per oggetto:

```
Passata di pomodoro 6
Riso Carnaroli 2
Olio EVO 3 / 1
Sale grosso
```

- `nome quantità` — il nome prende tutto fino all'ultimo numero, quindi `Farina 00 4`
  viene letto come *Farina 00* → 4.
- `nome quantità / minimo` — imposta anche la soglia di scorta.
- Solo il nome — la quantità vale 1.

Prima di confermare vedi un'anteprima riga per riga, e scegli cosa fare con gli oggetti
già presenti: **Somma** (rifornimento) o **Sostituisci** (conteggio da zero).

---

## I file

```
index.html              tutta l'app: struttura, stile e logica
manifest.webmanifest    nome, colori e icone per l'installazione sul telefono
sw.js                   service worker: fa funzionare l'app senza rete
favicon.svg             icona della scheda del browser
icon-192.png            icona dell'app installata
icon-512.png            icona dell'app installata, alta risoluzione
icon-512-maskable.png   variante con margine, per le icone ritagliate di Android
apple-touch-icon.png    icona per la home di iPhone e iPad
.nojekyll               dice a GitHub Pages di servire i file così come sono
```

Devono stare **tutti nella stessa cartella**, uno accanto all'altro.

---

## Pubblicare su GitHub Pages

1. Crea una repository e carica i file (tutti nella radice, non dentro sottocartelle).
2. **Settings → Pages → Source: Deploy from a branch**, branch `main`, cartella `/ (root)`.
3. Salva e aspetta un minuto.

Nessun workflow, nessuna Action: i file vengono serviti così come sono, perché non c'è
niente da compilare.

Il sito sarà su `https://TUO-UTENTE.github.io/NOME-REPO/`. Aprilo dal telefono e usa
«Aggiungi a schermata Home».

> **Attenzione:** i dati sono legati al browser e all'indirizzo del sito. Cambiare
> dominio, svuotare i dati del sito o usare un altro browser significa ripartire da zero.
> Il pannello **Dati → Esporta** esiste apposta.

### Provarlo in locale

Il doppio clic su `index.html` funziona per tutto tranne il service worker, che richiede
`http`. Per una prova completa, dalla cartella:

```bash
python3 -m http.server 8000
```

e apri `http://localhost:8000`.

---

## Com'è fatto dentro

Il file è diviso in sezioni numerate, in questo ordine:

1. **Icone** — SVG generati da una funzione, così non serve nessuna libreria.
2. **Utility** — id, normalizzazione dei nomi, formattazione dei numeri, confronti.
3. **Persistenza** — `load()` e `save()`, gli unici due punti che toccano `localStorage`.
4. **Lettura dei dati** — raggruppamento per categoria e ordinamento.
5. **Modifiche ai dati** — ogni funzione chiama `commit()`, che salva e ridisegna.
6. **Lettura delle liste** — il parser del testo.
7. **Disegno della pagina** — `render()` ricostruisce la lista da zero.
8–11. Ricerca, pulsante Aggiungi, pannelli, avvio.

Due scelte che vale la pena conoscere:

**Ridisegno completo.** Ogni modifica ricostruisce l'intera lista invece di aggiornare la
singola riga. Su un inventario domestico la differenza non si percepisce, e in cambio
non esiste la classe di errori in cui i dati e ciò che vedi a schermo divergono.

**Delega degli eventi.** La lista ha un solo `addEventListener`, sul contenitore. I
pulsanti dichiarano cosa fare con `data-act`, quindi ridisegnare non richiede di
riagganciare centinaia di ascoltatori.

### Modello dati

```jsonc
{
  "version": 1,
  "categories": [
    { "id": "a1b2c3d", "name": "Dispensa" }
  ],
  "items": [
    {
      "id": "e4f5g6h",
      "name": "Passata di pomodoro",
      "qty": 6,
      "min": 2,          // null quando la soglia non è impostata
      "catId": "a1b2c3d" // "__uncat__" per gli oggetti senza categoria
    }
  ]
}
```

`localStorage` sa memorizzare solo stringhe: l'oggetto viene convertito con
`JSON.stringify` prima di salvarlo e riletto con `JSON.parse`. Sono le due righe dentro
`load()` e `save()`.

Eliminare una categoria non elimina gli oggetti: passano a «Senza categoria».

### Il colore ha un significato

L'interfaccia è in scala di grigi tranne due colori, ognuno con un compito solo:

| Colore | Dove | Vuol dire |
|---|---|---|
| Giallo `#F2B705` | pulsante Aggiungi | agisci |
| Rosso `#BE2F27` | righe e conteggi | scorta esaurita |

È la ragione per cui l'allarme si vede senza cercarlo. Se aggiungi funzionalità, resisti
alla tentazione di colorare altro.

---

## Modificare l'app

Apri `index.html` con un editor di testo. Qualche punto di partenza:

- **Colori** — le variabili in cima al blocco `<style>`, dentro `:root`.
- **Allarme sotto la soglia invece che alla soglia** — nella funzione `isLow`, cambia
  `it.qty <= it.min` in `it.qty < it.min`.
- **Passo dei pulsanti + e −** — le chiamate `bump(id, 1)` e `bump(id, -1)`.

Dopo ogni modifica alza `VERSION` in `sw.js` (`inventario-v1` → `inventario-v2`),
altrimenti il browser continua a mostrare la copia vecchia salvata in cache.

---

## Se un giorno vorrai sincronizzare più dispositivi

`load()` e `save()` sono gli unici due punti che toccano la memoria: cambiando quelli,
il resto dell'app non se ne accorge.

- **Manuale** — esporta il JSON e importalo altrove. È quello che c'è già.
- **Backend-as-a-Service** — [Supabase](https://supabase.com) (Postgres, autenticazione,
  Row Level Security) o [Firebase](https://firebase.google.com/products/firestore), che
  ha i listener realtime integrati: due telefoni si aggiornano da soli. I piani gratuiti
  bastano per un uso personale. Self-hosted: [PocketBase](https://pocketbase.io).
- **Local-first / CRDT** — [Yjs](https://yjs.dev), [Automerge](https://automerge.org):
  scrivi sempre in locale, la libreria riconcilia quando torna la rete.

**Un consiglio sui conflitti.** Per un inventario la quantità è un contatore, non un
valore assoluto. Sincronizzando le *operazioni* (`+3`, `-1`) invece dello *stato finale*
(`qty = 5`), due persone che rifornono insieme non si sovrascrivono a vicenda.

---

## Licenza

MIT.
