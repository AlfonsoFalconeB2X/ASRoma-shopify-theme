# Patch al motore di personalizzazione (bundle `assets/react-*`)

Il motore di personalizzazione prodotto (`bitf-customization-engine` + app `react/apps/*`) arriva nel tema **solo come bundle compilato** (Vite). Non abbiamo accesso al sorgente. Le correzioni sotto sono state applicate **direttamente sui file minificati** e vanno riportate al produttore, che deve integrarle nel sorgente. Se non lo fa, la prossima build che ci consegnerà reintrodurrà il bug.

| ID | Data | Problema | Stato nel sorgente del produttore |
|---|---|---|---|
| ASR-PATCH-01 | 2026-11-20 | Taglia sbagliata nel carrello: la variante scelta viene azzerata all'arrivo della risposta del BFF | **Da correggere** |
| ASR-PATCH-02 | 2026-11-20 | Dopo la desincronizzazione lo store ignora tutti i cambi taglia successivi | **Da correggere** |

---

## Sintomo segnalato

Su prodotti personalizzabili (maglie gara home, adulto e bambino), dopo l'aggiunta al carrello compare una taglia diversa da quella selezionata. È sempre la **prima taglia disponibile**: S sulla maglia adulto, 7-8 anni sulla maglia bambino. Il bug è intermittente e dipende dai tempi di risposta del BFF.

Sui prodotti personalizzabili sia "PERSONALIZZA" sia "AGGIUNGI AL CARRELLO" sono componenti React. Aggiungono al carrello `store.currentProductVariant` dello store `shared`, non la variante del form del tema (`snippets/buy-buttons.liquid`).

## Causa

1. La preview in galleria (`CEProductCustomizationPreview`, `watchStoreChanges: true`) imposta `currentProductHandle` e `currentProductVariant`. La variante viene dalla prop `productVariant`, cioè `product.selected_or_first_available_variant` renderizzato da Liquid. Poi parte la chiamata al BFF `/api/ce-product-customization`.
2. Il cliente clicca una taglia prima che il BFF risponda. Il tema emette `variant:change` e lo store passa correttamente alla nuova variante.
3. Arriva la risposta del BFF. `useCEProductLoader` esegue `setStore({ currentProduct: product, currentProductVariant: null, ... })`.
4. La preview vede `currentProductVariant === null` e reimposta la prop **iniziale** (la prima taglia).
5. `useCEVariantEffects` accetta un `variant:change` solo se `event.detail.previousVariant.id === store.currentProductVariant.id`. Da questo momento la condizione non è più vera, quindi lo store resta bloccato sulla prima taglia fino al ricaricamento della pagina.

**Come riprodurlo:** DevTools → Network → throttling "Slow 4G". Apri la PDP di una maglia personalizzabile senza `?variant=` e clicca subito una taglia diversa dalla prima. Attendi che "PERSONALIZZA" si abiliti e aggiungi al carrello. Per verificare lo stato, in console: `window.mappedStore.shared.currentProductVariant.title`.

---

## ASR-PATCH-01: il loader non deve azzerare la variante del prodotto corrente

**Sorgente del produttore (dove va fatta la correzione):**
`react/bitf-customization-engine/hooks/useCEProductLoader.js`, funzione `loadProductCustomizationSettings`, primo `setStore` dopo il parsing della risposta.

Codice originale (dalla source map):

```js
setStore({
  currentProduct: product,
  currentProductVariant: null,
  customizationEngineSettings,
  customizationFormValidators,
  isCustomizationValid: true,
});
```

Correzione suggerita: azzerare la variante **solo se appartiene a un prodotto diverso** da quello appena caricato.

```js
const normalizeId = (id) => String(id ?? '').split('/').pop(); // GID Storefront API o id numerico Liquid
const belongsToProduct = (product, variant) =>
  !!(product && variant && Array.isArray(product.variants) &&
     product.variants.some((v) => normalizeId(v.id) === normalizeId(variant.id)));

setStore((prev) => ({
  currentProduct: product,
  currentProductVariant:
    prev.currentProductVariant && product &&
    (belongsToProduct(product, prev.currentProductVariant) ||
     !prev.currentProduct ||                                    // primo caricamento: la variante è già di questo prodotto
     (product.handle && prev.currentProduct.handle === product.handle))
      ? prev.currentProductVariant
      : null,
  customizationEngineSettings,
  customizationFormValidators,
  isCustomizationValid: true,
}));
```

**Patch applicata al bundle:** `assets/react-useCEProductCustomization-DeDNb-7a-p1.js`, marcatore `/*ASR-PATCH-01*/`.

Stringa originale sostituita:

```js
t({currentProduct:a,currentProductVariant:null,customizationEngineSettings:d,customizationFormValidators:m,isCustomizationValid:!0});
```

## ASR-PATCH-02: `variant:change` deve accettare le varianti del prodotto corrente

**Sorgente del produttore:**
`react/bitf-customization-engine/hooks/useCEVariantEffects.js`, primo `useEffect` (`handleVariantChange`).

Codice originale:

```js
if (previousVariant?.id === storeRef.current.currentProductVariant?.id && variant) {
  setStore({ currentProductVariant: variant });
}
```

Il controllo su `previousVariant` serve a non far reagire store di altri prodotti (es. quick-buy). È però troppo fragile: basta una sola desincronizzazione perché lo store ignori per sempre i cambi taglia. Correzione suggerita: accettare l'evento anche quando la variante appartiene al prodotto dello store.

```js
if (
  variant &&
  (previousVariant?.id === storeRef.current.currentProductVariant?.id ||
   belongsToProduct(storeRef.current.currentProduct, variant))
) {
  setStore({ currentProductVariant: variant });
}
```

**Patch applicata al bundle:** stesso file, marcatore `/*ASR-PATCH-02*/`.

Stringa originale sostituita:

```js
(c==null?void 0:c.id)===((o=u.current.currentProductVariant)==null?void 0:o.id)&&r&&t({currentProductVariant:r})
```

Nota: questa parte funziona solo se il BFF restituisce `product.variants` con gli `id`, come fa per il quick-buy. Se `variants` manca, il comportamento resta quello originale. Il bug segnalato è comunque risolto da ASR-PATCH-01.

---

## File toccati nel tema

Gli import tra chunk sono relativi e **senza versione** (`./react-xxx.js`). Solo i file d'ingresso `*.min.js` ricevono `?v=` da `asset_url`. Per evitare che i browser o la CDN servano chunk vecchi dalla cache, i chunk modificati sono stati **rinominati con il suffisso `-p1`**, con `git mv` per conservare la storia.

Tutti i moduli importano lo stesso URL. È indispensabile: con due URL diversi esisterebbero due istanze del modulo, e quindi due store `shared` separati.

| Originale | Nuovo nome | Modifica |
|---|---|---|
| `react-useCEProductCustomization-DeDNb-7a.js` | `react-useCEProductCustomization-DeDNb-7a-p1.js` | ASR-PATCH-01, ASR-PATCH-02 + commento di intestazione |
| `react-CEAddToCartButton-BMNHdCoc.js` | `react-CEAddToCartButton-BMNHdCoc-p1.js` | solo percorso dell'import |
| `react-CEProductCustomizationForm-DFPs0B4h.js` | `react-CEProductCustomizationForm-DFPs0B4h-p1.js` | solo percorso dell'import |
| `react-CEProductCustomizationPreview-BaUR2-NT.js` | `react-CEProductCustomizationPreview-BaUR2-NT-p1.js` | solo percorso dell'import |
| `react-CEProductCustomizationPrice-C3k5aiJy.js` | `react-CEProductCustomizationPrice-C3k5aiJy-p1.js` | solo percorso dell'import |

File d'ingresso aggiornati solo negli import, con il nome invariato: `react-addToCartButton.min.js`, `react-productCustomizationButton.min.js`, `react-productCustomizationDrawer.min.js`, `react-productCustomizationForm.min.js`, `react-productCustomizationPreviewLoader.min.js`, `react-productCustomizationPrice.min.js`, `react-quickBuy.min.js`.

Note:

- Le source map (`*.js.map`) **non** sono state rigenerate. I file `-p1` puntano ancora alle mappe originali, che descrivono il codice **senza patch** (le colonne della riga patchata non corrispondono più).
- Gli altri chunk `react-*` con hash diversi (es. `react-useCEProductCustomization-UIvdj_NF.js`) sono build precedenti non referenziate da nessun file d'ingresso. Non sono stati toccati.

## Verifica eseguita

I bundle reali (originali e patchati) sono stati eseguiti in Node + jsdom, con un BFF simulato che risponde in 300 ms:

| Scenario | Originale | Con patch |
|---|---|---|
| Clic su L prima della risposta del BFF → variante nello store dopo la risposta | S ❌ | L ✅ |
| Clic successivo su M | ignorato (S) ❌ | M ✅ |
| `variant:change` di un altro prodotto | ignorato ✅ | ignorato ✅ |
| Cambio prodotto (scenario quick-buy) → variante del vecchio prodotto azzerata | null ✅ | null ✅ |

Ogni scenario è stato provato sia con `product.variants` presente nella risposta BFF, sia senza.

Da verificare in staging prima della pubblicazione: PDP maglia adulto e bambino, drawer di personalizzazione, quick-buy nel blocco "Il match perfetto".

## Come riapplicare o rimuovere

- **Nuova build dal produttore:** verificare che contenga le correzioni descritte sopra. Se non le contiene, riapplicare le due sostituzioni (cercare le stringhe originali indicate) e ripetere la rinomina degli import.
- **Rollback:** fare il revert del commit che introduce questo file. Nomi e import tornano quelli originali.

## Problema aperto, non coperto da queste patch

L'errore "ID o parametri di linea non validi" nel carrello è la risposta di Shopify a `/cart/change.js` con una line key che non esiste più. Probabilmente il drawer del carrello mostra un render non aggiornato: viene ridisegnato da più flussi asincroni senza ordinamento (`cart:change`, `cart:refresh` da `free-product.js`, aggiunta React) e `free-product.js` / `cart-validator.js` modificano il carrello in autonomia. Non è confermato e richiede una riproduzione con il pannello Network aperto. È indipendente dal bug della taglia.
