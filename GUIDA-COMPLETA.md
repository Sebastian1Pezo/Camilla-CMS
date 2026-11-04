# GUIDA COMPLETA — Sito Web Camilla Laoshi (CMS & Deploy)

> Documento generato da Hermes Agent. Contiene tutti gli approcci testati, le soluzioni adottate, i bug risolti e le raccomandazioni per progetti futuri.

---

## 1. Obiettivo del progetto

Sito web per Camilla (tutor bilingue italiano-cinese a Torino) con:
- Presentazione, prezzi, orari, contatti.
- Pannello admin per modificare contenuti (orari, contatti, annuncio promo).
- Salvataggio automatico su GitHub → deploy automatico su Netlify.
- Login sicuro (email/password, no GitHub per il cliente).
- Responsive (mobile + desktop).
- Nessun segreto esposto nel codice.

---

## 2. Approcci testati (in ordine cronologico)

### 2.1 Sito vanilla + pannello custom + password hardcoded
**Repo:** `Sebastian1Pezo/Camilla` (commit iniziali)

**Struttura:**
- `index.html` + `styles.css` + `main.js` (sito pubblico)
- `private/admin.html` + `private/admin.js` + `private/auth.js` (pannello admin)
- `private/login.html` (login con password hardcoded)
- `contatti.json` + `disponibilita.json` (dati caricati dal sito)

**Problema:** Password hardcoded in `auth.js` (`CAMILLA_PASSWORD = "camilla2026"`). Visibile a chiunque apra DevTools.

**Soluzione:** Rimossa password hardcoded, sostituita con Netlify Identity.

---

### 2.2 Sveltia CMS (abbandonato)
**Motivo abbandono:** Richiede login GitHub OAuth. Troppo tecnico per Camilla/cliente.

**Problema:** Il cliente dovrebbe creare un account GitHub, autorizzare l'OAuth app, e gestire token. Non accettabile per un utente non tecnico.

**Lezione:** Sveltia CMS è ottimo per sviluppatori, ma non per clienti finali.

---

### 2.3 Tina CMS (abbandonato)
**Motivo abbandono:** Richiede Node.js 22 (non disponibile), build step complesso, 25 vulnerabilità npm, errori di hydration out-of-the-box.

**Problema:** Troppo pesante per un sito vetrina. Richiede riscrittura in React/Next.js. Node 26 non supportato.

**Lezione:** Tina CMS è ottimo per progetti enterprise con budget e team tecnico. Overkill per piccoli siti.

---

### 2.4 Decap CMS + Netlify Identity (testato, poi abbandonato)
**Problema:** Interfaccia datata ("è proprio brutto"). L'utente ha preferito un pannello custom.

**Lezione:** Decap CMS funziona bene ma l'UI è vecchia. Per clienti che vogliono un'esperienza moderna, meglio un pannello custom.

---

### 2.5 Pannello custom + Netlify Identity + Git Gateway (SOLUZIONE FINALE)
**Stato:** ✅ In produzione

**Struttura:**
```
sito-sveltia/
├── admin/
│   └── index.html       # Pannello CMS moderno (sidebar, dashboard, contenuti, impostazioni)
├── admin.css            # Stili responsive
├── index.html           # Sito pubblico
├── main.js              # Logica sito
├── styles.css           # Stili sito
├── netlify.toml         # Config Netlify
├── content/
│   ├── settings.json    # Contatti, orari, annuncio
│   └── news/            # Articoli (futuro)
└── public/uploads/      # Media (futuro)
```

**Funzionalità:**
- Login con email+password (Netlify Identity).
- Overlay di login che nasconde il contenuto finché non è loggato.
- Sidebar responsive (hamburger su mobile).
- Dashboard con statistiche.
- Impostazioni: contatti, orari, annuncio promo.
- Salvataggio automatico su GitHub via Git Gateway.
- Nessun GitHub visibile al cliente.

---

## 3. Sicurezza: cosa è protetto e cosa no

### ✅ Protetto (sempre)
| Aspetto | Come |
|---------|------|
| Password del cliente | Netlify Identity (hashata, mai nel codice) |
| Segreti (chiavi API) | Variabili d'ambiente su Netlify + Netlify Functions |
| Accesso admin | Netlify Identity + Git Gateway (token temporaneo) |
| Repo GitHub | Privato (solo sviluppatore ha accesso) |
| Cronologia modifiche | Git commits con autore e data |

### ❌ Non protetto (limitazione architetturale)
| Aspetto | Realtà |
|---------|--------|
| Codice HTML/CSS/JS client | Visibile (DevTools, wget, scraping) |
| Logica business client | Visibile (se nel browser) |
| Struttura del sito | Visibile (sitemap, routes) |
| Immagini/video | Copiabili (right-click, scraping) |

### Regola d'oro
**Mai mettere segreti nel codice client.** Password, chiavi API, token → solo server-side (Netlify Functions o variabili d'ambiente).

---

## 4. Bug risolti

### Bug 1: Password hardcoded nel codice
**Problema:** `auth.js` conteneva `CAMILLA_PASSWORD = "camilla2026"` in chiaro.
**Soluzione:** Rimossa password hardcoded. Sostituita con Netlify Identity.

### Bug 2: CSS non caricato nel pannello admin
**Problema:** `admin/index.html` puntava a `admin.css` ma il file era nella root.
**Soluzione:** Cambiato path in `../admin.css`.

### Bug 3: Contenuto admin visibile senza login
**Problema:** La sidebar e il contenuto erano visibili anche senza login.
**Soluzione:** Aggiunto overlay di login che copre tutto lo schermo. Il contenuto è nascosto (`display: none`) finché non è loggato.

### Bug 4: Pulsanti non funzionavano
**Problema:** Gli event listener erano dentro `initApp()`, che veniva chiamato solo dopo il login. Se il login non avveniva, i listener non venivano mai aggiunti.
**Soluzione:** Spostati gli event listener in `DOMContentLoaded`, così funzionano anche prima del login.

### Bug 5: `overlayStatus is not defined`
**Problema:** Le variabili `overlay`, `overlayStatus`, `overlayLogin`, `overlayRecover` non erano dichiarate.
**Soluzione:** Aggiunte dichiarazioni `const` per le variabili.

### Bug 6: Decap CMS "You don't have sufficient permissions"
**Problema:** Decap CMS con `git-gateway` backend richiede il ruolo "admin" in Netlify Identity.
**Soluzione:** Aggiunto `base_url`, `auth_endpoint`, `site_domain` al config.yml. Verificato che Git Gateway sia abilitato.

### Bug 7: Template literal redatto dal sistema
**Problema:** Il sistema mascherava i template literal `` `Bearer ${token}` `` con `***`.
**Soluzione:** Usato `patch` invece di `write_file` per ripristinare il codice corretto.

---

## 5. Impostazioni Netlify (da fare una volta)

### 5.1 Identity
1. Netlify → Site configuration → **Identity** → **Enable Identity**.
2. Registration preferences → **Invite only**.
3. (Opzionale) Services → **Git Gateway** → **Enable Git Gateway**.

### 5.2 OAuth (se si usa Sveltia CMS)
1. GitHub → Settings → Developer settings → OAuth Apps → **New OAuth App**.
   - Application name: `Sito Camilla CMS`
   - Homepage URL: `https://tuosito.netlify.app`
   - Authorization callback URL: `https://api.netlify.com/auth/done`
2. Netlify → Site configuration → **Access & security → OAuth**.
   - Provider: GitHub
   - Client ID e Client Secret: dalla OAuth App.

### 5.3 Headers di sicurezza (netlify.toml)
```toml
[build]
  publish = "."

[[headers]]
  for = "/admin/*"
  [headers.values]
    X-Robots-Tag = "noindex, nofollow"
```

### 5.4 Invitare il cliente
1. Netlify → Identity → **Invite users**.
2. Inserisci email del cliente.
3. Il cliente riceve email con link di conferma.
4. Il cliente clicca il link → imposta password → account attivo.

---

## 6. Flusso completo per il cliente

```
1. Tu vai su Netlify → Identity → Invite users → inviti email di Camilla
2. Camilla riceve email → clicca link → imposta password
3. Camilla va su tuosito.netlify.app/admin/
4. Clicca "Login" → inserisce email + password
5. Vede il pannello admin (dashboard, contenuti, impostazioni)
6. Modifica i campi (orari, contatti, annuncio)
7. Clicca "Salva" → commit su GitHub → Netlify redeploya
8. Il sito pubblico si aggiorna in ~1 minuto
```

**Zero GitHub visibile. Zero codice. Zero JSON. Solo campi di testo e un pulsante "Salva".**

---

## 7. Sicurezza: standard per progetti futuri

### Regola 1: Mai segreti nel codice client
```js
// ❌ MAI
const API_KEY = "***";
const PASSWORD="***";

// ✅ FARE
// Usare variabili d'ambiente su Netlify + Netlify Functions
```

### Regola 2: Minificazione + no source maps in produzione
- Se usi Vite/Webpack → già fatto di default.
- Se vanilla JS → usa un minifier prima del deploy.
- **Non pubblicare source maps** → DevTools mostra codice illeggibile.

### Regola 3: Repo privato + accesso controllato
- Repo sempre privato.
- Solo tu + collaboratori espliciti.
- Il cliente non ha accesso al repo.

### Regola 4: Netlify Identity per admin
- Mai password hardcoded.
- Mai Basic-Auth con password in `_headers` (solo stopgap).
- Sempre Netlify Identity + Git Gateway.

### Regola 5: Protezione legale
- Contratto con clausola anti-copia.
- Copyright sul codice sorgente.
- Pagamenti a milestone (non consegni codice finché non pagato).

### Regola 6: Headers di sicurezza (netlify.toml)
```toml
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

---

## 8. Confronto approcci CMS

| Approccio | Login | Salvataggio | Complessità | Sicurezza | Costo |
|-----------|-------|-------------|-------------|-----------|-------|
| **Pannello custom + Identity** | Email+password | Git Gateway (automatico) | Bassa | Alta | Gratis |
| **Decap CMS + Identity** | Email+password | Git Gateway (automatico) | Media | Alta | Gratis |
| **Sveltia CMS** | GitHub OAuth | GitHub (automatico) | Media | Alta | Gratis |
| **Tina CMS** | Tina Cloud | Tina Cloud / Git | Alta | Alta | Free tier |
| **Sanity** | Sanity auth | Sanity Cloud | Alta | Alta | Free tier |
| **WordPress** | WP login | Database | Bassa (per non-dev) | Bassa | Hosting $ |

**Raccomandazione per piccoli siti:** Pannello custom + Netlify Identity + Git Gateway.

---

## 9. Cosa fare e cosa evitare

### ✅ FARE
- Usare Netlify Identity per login admin.
- Usare Git Gateway per salvataggio automatico.
- Mantenere il repo GitHub privato.
- Aggiungere headers di sicurezza in `netlify.toml`.
- Minificare il codice in produzione.
- Usare variabili d'ambiente per i segreti.
- Proteggere legalmente il codice (contratto).

### ❌ EVITARE
- Password hardcoded nel codice.
- Basic-Auth con password in `_headers` (solo stopgap).
- Obfuscation estrema (security theater).
- "Blocca DevTools" (facilmente aggirabile, pessima UX).
- Mettere chiavi API nel codice client.
- Rendere il repo pubblico (a meno che non sia open source).
- Usare Tina CMS per siti semplici (overkill).

---

## 10. Struttura file finale del progetto

```
sito-sveltia/
├── admin/
│   └── index.html       # Pannello CMS moderno (sidebar, dashboard, contenuti, impostazioni)
├── admin.css            # Stili responsive del pannello
├── index.html           # Sito pubblico
├── main.js              # Logica sito (carica settings, news)
├── styles.css           # Stili sito pubblico
├── netlify.toml         # Config Netlify (headers, build)
├── content/
│   ├── settings.json    # Contatti, orari, annuncio
│   └── news/            # Articoli (futuro)
└── public/uploads/      # Media (futuro)
```

---

## 11. Checklist prima di pubblicare

- [ ] `contatti.json` / `settings.json` con numero + email + WeChat veri
- [ ] Password nuova in Netlify Identity (non hardcoded)
- [ ] Netlify Identity abilitato (Enable Identity + Invite only)
- [ ] Git Gateway abilitato (per salvataggio automatico)
- [ ] Repo GitHub privato
- [ ] Headers di sicurezza in `netlify.toml`
- [ ] Minificazione attiva (se applicabile)
- [ ] Testato il flusso completo (login → modifica → salva → sito aggiornato)
- [ ] Contratto con il cliente (clausola anti-copia)

---

## 12. Note finali

- **Il codice client-side è sempre visibile.** Non esiste un modo tecnico per nasconderlo. La protezione sta nel repo privato, nel login sicuro, e nel contratto legale.
- **Il pannello custom + Netlify Identity + Git Gateway** è il miglior compromesso per piccoli siti: gratis, semplice, sicuro, e facile da usare per il cliente.
- **Per progetti più grandi** (40+ pagine, e-commerce, dashboard), considerare Astro, Next.js, o un CMS headless (Sanity, Strapi).
- **Per clienti non tecnici**, evitare flussi che richiedono GitHub OAuth o token manuali.

---

*Documento generato da Hermes Agent — Nous Research*
*Sito: Camilla Laoshi (camillalaoshi.netlify.app)*
*Repo: Sebastian1Pezo/Camilla*
EOF
echo "written"