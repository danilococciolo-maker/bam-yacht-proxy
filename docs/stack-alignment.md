# BAM Vision — Audit dello stack tecnologico end-to-end

**Data:** 27 agosto 2026
**Perimetro:** 5 repository applicativi + 1 proxy, dall'ingestione delle foto sorgente
alla consegna del video finale renderizzato.
**Stato:** analisi. Nessuna riga di codice applicativo è stata modificata.

---

## 0. Perimetro e metodo

Non esiste un repository chiamato `bam-vision`. "BAM Vision" è il nome dell'insieme:
il dominio `vision.bamgroup.it` è referenziato da yacht, auto e real-estate. L'audit
copre tutti i repository accessibili all'account.

| Repository | Ruolo | File | Endpoint API | Ultimo commit |
|---|---|---|---|---|
| `bam-yacht-privato` | Verticale yacht (produzione, il più avanzato) | 366 | 89 | 2026-08-26 |
| `bam-auto-privato` | Verticale automotive (produzione) | 159 | 53 | 2026-08-20 |
| `bam-real-estate-privato` | Verticale immobiliare (produzione) | 146 | 45 | 2026-08-20 |
| `bam-studio-privato` | Motore universale multi-vertical (non validato in produzione) | 284 | 42 | 2026-07-30 |
| `bam-site-privato` | Sito vetrina + raccolta lead | 20 | 1 | 2026-08-27 |
| `bam-yacht-proxy` | Ponte temporaneo di verifica visiva (solo README) | 1 | 0 | 2026-07-20 |

Metodo: lettura diretta dei manifest, delle configurazioni di deploy, degli endpoint
e confronto hash-per-hash dei file omonimi tra i tre verticali.

**Il dato che riassume tutto l'audit:** i tre verticali condividono **133 file con lo
stesso nome**. Di questi, **21 sono identici byte-per-byte e 112 sono divergenti**.
Non sono tre prodotti diversi: sono tre copie dello stesso prodotto che hanno smesso
di essere copie.

---

## 1. STATO ATTUALE per stadio della pipeline

### 1.1 Ingest (upload delle foto sorgente)

| Componente | Verticali (yacht / auto / real-estate) | Studio | File di prova |
|---|---|---|---|
| Endpoint di upload | `api/fal-upload.js` | `api/upload-source.js` | rispettivi |
| Destinazione | fal.ai storage (`rest.fal.ai`) | contratto di upload protetto | `api/fal-upload.js`, `runtime/source-upload-contract.js` |
| Coda di attesa | `api/pending-uploads.js` (**solo yacht**) | assente | `bam-yacht-privato/api/pending-uploads.js` |
| Runtime | Node CommonJS su Vercel Functions | idem | `package.json` (nessun `"type": "module"`) |

### 1.2 Storage e persistenza

| Componente | Yacht | Auto | Real-estate | Studio |
|---|---|---|---|---|
| Database + auth | Supabase `ftblcuklncliherrbmgz` | Supabase `vqkoqepotapamkzlomtz` | Supabase `ruedykgseogsmieqmffd` | **nessuno** |
| Client browser | `@supabase/supabase-js@2` | idem | idem | — |
| Archivio a freddo | S3 via `@aws-sdk/client-s3` ^3.658.0 | idem | idem | — |
| Schema versionato | **assente** | **assente** | **assente** | `supabase/product_hub.sql` (non collegato a variabili d'ambiente) |
| Bucket | `dossier-pdf`, `VIDEO_BUCKET`, tabella `profiles` | idem | idem | — |

Tre progetti Supabase distinti, nessuno dei tre con migrazioni nel repository: lo schema
di produzione esiste solo dentro la console Supabase.

### 1.3 Elaborazione / AI (linguaggio, brief, giudici)

| Componente | Verticali | Studio | File di prova |
|---|---|---|---|
| Accesso al modello | `fetch` grezzo su `api.anthropic.com`, header `anthropic-version: 2023-06-01` | AWS Bedrock via `@aws-sdk/client-bedrock-runtime` + OIDC | `api/judge.js`, `runtime/bedrock.js:96` |
| SDK ufficiale | **nessuno** in nessun `package.json` | — | tutti i `package.json` |
| Credenziale | `ANTHROPIC_API_KEY` | `AWS_ROLE_ARN` + `BEDROCK_MODEL_ID` | env var per repo |
| Modello — giudici, dossier | `claude-sonnet-5` | da env, nessun default | `api/judge.js`, `api/judge2.js`, `api/dossier-text.js` |
| Modello — composizione prompt, orchestrazione | `claude-sonnet-4-6` | idem | `api/inngest.js`, `api/room-brief-router.js` |
| Modello — compose-prompt auto | `claude-fable-5` | — | `bam-auto-privato/api/compose-prompt.js` |

I tre ID modello sono tutti validi e attuali. Il problema non è quale modello, è che
sono **tre fasce diverse senza una policy scritta** e che la scelta è codificata nei
file invece che nella configurazione.

### 1.4 Generazione immagini

| Modello fal.ai | Yacht | Auto | Real-estate | Studio |
|---|---|---|---|---|
| `fal-ai/nano-banana-pro/edit` | ✅ | ✅ | ✅ | ✅ |
| `fal-ai/gpt-image-1.5/edit` | ✅ | ✅ | ✅ | — |
| `fal-ai/flux-2-max/edit` | ✅ | ✅ | — | — |
| `fal-ai/flux-pro/kontext/max` | — | ✅ | — | ✅ |
| `fal-ai/bytedance/seedream` | ✅ | — | — | — |
| `fal-ai/evf-sam` (segmentazione) | — | ✅ | ✅ | — |
| `fal-ai/image-preprocessors/depth-anything/v2` | ✅ | — | — | — |

Client: `@fal-ai/client` (versione `latest`) solo in studio; i verticali chiamano
`queue.fal.run` con `fetch` grezzo.

### 1.5 Generazione video (clip)

| Modello | Yacht | Auto | Real-estate | Studio |
|---|---|---|---|---|
| `fal-ai/veo3.1/fast/image-to-video` | ✅ | ✅ | ✅ | ✅ |
| `fal-ai/veo3.1/fast/first-last-frame-to-video` | — | — | ✅ | — |
| `fal-ai/veo3.1/fast/reference-to-video` | — | — | ✅ | — |
| `fal-ai/bytedance/seedance/v1/pro/image-to-video` | ✅ | — | — | — |
| `fal-ai/kling-video/o3/standard/image-to-video` | — | — | — | ✅ |
| `fal-ai/topaz/upscale/video` | ✅ | ✅ | ✅ | ✅ |
| Worker Python dedicato | — | Modal (`identity-worker/modal_app.py`) | — | — |
| Replicate (proxy `/v1/*`) | ✅ | ✅ | ✅ | — |

### 1.6 Composizione e rendering finale

| Componente | Yacht | Auto | Real-estate | Studio |
|---|---|---|---|---|
| Motore di montaggio | `ffmpeg-static` ^5.3.0 (43 file lo invocano) | ^5.2.0 (8 file) | ^5.2.0 (9 file) | **nessun ffmpeg** |
| Motore alternativo | Creatomate (12 file) | Creatomate (6 file) | Creatomate (7 file) | **Creatomate esclusivo** (19 file) |
| Codec video | `libx264`, `yuv420p` | idem | idem | `h264` / `hevc` (via Creatomate) |
| CRF | 15, 16, 17, 18, 20, 24 | 20, 24 | 18, 20, 24 | n/d |
| Preset | `ultrafast`, `veryfast`, `fast`, `medium` | `veryfast`, `medium` | `veryfast`, `medium` | n/d |
| FPS | 24, 30, 6 | 24 | 24, **25** | n/d |
| Risoluzione target | 1920×1080 | 1920×1080 | 1920×1080 | **3840×2160** |
| Audio | `aac`, `assets/bam-musica.mp3` (2,0 MB) | `aac`, **nessun file audio nel repo** | idem | `aac`, `assets/bam-studio-official-music.mp3` (720 KB) |
| Watermark | `api/watermark.js` | `api/watermark.js` | `api/watermark.js` | `assets/bam-logo-watermark.svg` |

Due traccie musicali "ufficiali" diverse, e due verticali su tre senza alcun file audio
nel repository.

### 1.7 Orchestrazione e job

| Componente | Yacht | Auto | Real-estate | Studio |
|---|---|---|---|---|
| Coda | Inngest ^4.11.0 | ^4.11.0 | ^4.11.0 | **nessuna** |
| Cron Vercel | 3 (due **ogni minuto**) | 2 (uno ogni minuto) | 1 | 0 |
| Recupero job | `api/render-recover.js`, `api/reap-stale-jobs.js` | idem | parziale | `runtime/run-recovery.js`, `runtime/resilience.js` |
| Limite funzione max | 800 s / 3009 MB | 800 s / 3009 MB | 800 s / 3009 MB | **60 s**, memoria default |

Lo scarto sui limiti di esecuzione è di oltre un ordine di grandezza: studio non è
configurato per reggere il montaggio che i verticali fanno oggi.

### 1.8 Delivery e frontend

| Componente | Yacht | Auto | Real-estate | Studio | Site |
|---|---|---|---|---|---|
| Hosting | Vercel | Vercel | Vercel | Vercel | Vercel |
| Dominio | `vision.bamgroup.it` | `bamauto.bamgroup.it` | `realestate.bamgroup.it` | `studio.bamgroup.it` | `bamgroup.it` |
| Entry point | `app.html` (105 righe) | `app.html` (130) | `app.html` (130) | `index.html` (74) | `index.html` |
| Tag `<script>` | **5** (bundle) | **63** | **63** | pochi, modulari | — |
| Fogli di stile | 20 `<link>` | 22 | 22 | 1 | — |
| JS a livello root | 78 file / 29 709 righe | 61 file / 8 701 righe | 61 file / 9 207 righe | `frontend/` + `runtime/` modulari | — |
| Vendor | **self-hosted** (`vendor/bam-vendor-supabase-2.112.3.js`, `lucide-0.468.0`) | CDN | CDN | nessun CDN | — |
| CDN esterni | jsdelivr, cdnjs, unpkg, sentry-cdn | idem | idem | **nessuno** | — |
| React | UMD 18 da unpkg | UMD 18 da unpkg | UMD 18 da unpkg | assente | — |
| Babel standalone | **sì, in produzione** | **sì** | **sì** | no | — |
| CSP | assente | assente | assente | **completa e restrittiva** | — |
| Build step | `vercel-build` → presidi + impronta asset | `postinstall` → 4 verifiche | `postinstall` → 1 patch | `verify` (check + test) | nessuno |

### 1.9 Qualità, test, CI

| Repository | Test | Workflow CI | Lockfile | `engines.node` |
|---|---|---|---|---|
| yacht | 56 file | **nessuno** | `package-lock.json` | `22.x` |
| studio | 103 file | 3 (`ci`, `verify`, sentry) | **assente** | `24.x` |
| auto | **0** | 3 (quality, Modal, sentry) | **assente** | **assente** |
| real-estate | **0** | 1 (sentry) | **assente** | `22.x` |
| site | 0 | nessuno | assente | nessun `package.json` |

Il repository con più test non ha CI. I due repository senza alcun test sono in produzione.

---

## 2. DISALLINEAMENTI

Ordinati per impatto.

### D1 — Tre copie divergenti dello stesso prodotto
133 file omonimi tra i verticali, **112 divergenti**, incluso tutto il cuore della
pipeline: `api/inngest.js`, `api/finalize-video.js`, `api/render-start.js`,
`api/judge.js`, `api/crossfade.js`, `api/save-video.js`, `api/workspace.js`.
Ogni correzione va scritta tre volte, e oggi non lo è: yacht ha 89 endpoint, auto 53,
real-estate 45. **Costo reale: ogni bug fix ha 2/3 di probabilità di non essere applicato.**

### D2 — Due architetture inconciliabili per lo stesso scopo
| | Verticali | Studio |
|---|---|---|
| LLM | Anthropic diretto, chiave API | AWS Bedrock, OIDC |
| Persistenza | Supabase | nessuna |
| Coda | Inngest | nessuna |
| Montaggio | ffmpeg in-function | Creatomate |
| Output | 1080p | 2160p |

Studio si dichiara nel proprio README «universal multi-vertical engine» e definisce i
profili `GENERIC / YACHT / REAL_ESTATE / AUTOMOTIVE / JET / MOBILHOME`
(`runtime/vertical-profiles.js`). È il bersaglio dichiarato della convergenza — ed è
anche il repository **meno aggiornato** (fermo al 30 luglio, quattro settimane dietro
i verticali) e quello con i limiti di funzione più bassi.

### D3 — Due motori di composizione video in parallelo
ffmpeg e Creatomate coesistono nei verticali senza una regola dichiarata su chi vince.
In studio ffmpeg non esiste. Non esiste un punto unico che decida come si monta un video.

### D4 — Parametri di codifica incoerenti
CRF da 15 a 24 (differenza di qualità e peso di circa 3×), quattro preset diversi,
FPS 24/25/30, risoluzione 1080p contro 2160p. Il video consegnato al cliente non ha
una specifica: ha una media di accidenti storici.

### D5 — Transpilazione JSX nel browser in produzione
`@babel/standalone` caricato da unpkg in tutti e tre i verticali. Costa un download di
oltre 2 MB e una compilazione a ogni caricamento di pagina, ed è una dipendenza di
terze parti nel percorso critico di rendering. Yacht ha già iniziato a uscirne
(vendor self-hosted, 5 script contro 63): la migrazione esiste, non è stata propagata.

### D6 — Nessuna Content-Security-Policy sui tre verticali
Studio ha una CSP completa. I verticali caricano codice eseguibile da quattro CDN
diversi (unpkg, jsdelivr, cdnjs, sentry-cdn) senza alcuna restrizione. `supabase-js`
è caricato da **due** CDN diversi nello stesso repository.

### D7 — Versioni divergenti sulle stesse dipendenze
| Dipendenza | Yacht | Auto | Real-estate | Studio |
|---|---|---|---|---|
| `@sentry/node` | ^10.64.0 | **^10.22.0** | ^10.64.0 | ^10.64.0 |
| Sentry browser (CDN) | 10.65.0 | 10.65.0 | 10.65.0 | — |
| `ffmpeg-static` | ^5.3.0 | ^5.2.0 | ^5.2.0 | — |
| `engines.node` | 22.x | **assente** | 22.x | **24.x** |

Studio dichiara `"latest"` per quattro dipendenze (`@aws-sdk/client-bedrock-runtime`,
`@fal-ai/client`, `@vercel/oidc-aws-credentials-provider`, `sharp`): ogni deploy può
installare codice diverso dal precedente senza che nulla nel repository cambi.

### D8 — Un solo lockfile su cinque repository
Solo yacht ha `package-lock.json`. Gli altri quattro risolvono le dipendenze a ogni
build: le build non sono riproducibili e un rollback del codice non è un rollback
delle dipendenze.

### D9 — Copertura di test e CI invertita rispetto al rischio
Auto e real-estate: zero test, in produzione. Yacht: 56 test, nessuna CI che li esegua
(la verifica gira solo dentro `vercel-build`). Studio: 103 test e 3 workflow, ma non è
in produzione.

### D10 — Schema del database non versionato
Nessuna migrazione SQL nei tre verticali. Tre progetti Supabase separati il cui schema
esiste solo nella console: non è ricostruibile, non è diffabile, non è allineabile.

### D11 — Codice morto accumulato per versioni
In `bam-yacht-privato/lib/`: `render-safe-workflow.js` esiste in sei varianti
(base, v4, v5, v6, v7, v8). **v5, v6 e v7 non sono referenziati da nessun file**; v4 da
uno solo; la base da 13 e la v8 da 9 — due workflow attivi in parallelo.
Stesso schema sui giudici: `render-fidelity-judge-corrected.js` non è referenziato da
nulla, mentre `render-fidelity-judge.js` (14 referenze) e `-active.js` (3) convivono.

### D12 — Asset di marca divergenti
Due file musicali "ufficiali" diversi (2,0 MB vs 720 KB, hash diversi), presenti solo in
yacht e studio. Auto e real-estate non hanno alcun file audio nel repository, pur
avendo `api/watermark.js` e la stessa pipeline di finalizzazione.

### D13 — Un worker Python isolato
`bam-auto-privato/identity-worker/modal_app.py` gira su Modal con FastAPI: un secondo
linguaggio, un secondo provider di calcolo, un secondo ciclo di deploy
(`.github/workflows/modal-identity-worker.yml`), usato da un solo verticale.

### D14 — Cron ridondanti e non allineati
Yacht esegue due cron **ogni minuto** (`reap-stale-jobs`, `sync-inngest-71bd8c9f`),
auto uno, real-estate nessuno — pur condividendo lo stesso motore di job Inngest e
gli stessi modi di fallire.

### D15 — Nessun SDK ufficiale per il modello
Tutte le chiamate al modello sono `fetch` grezzi con `anthropic-version: 2023-06-01`
scritto a mano in ogni file. Nessun `@anthropic-ai/sdk` in nessun `package.json`.
Retry, gestione degli errori tipizzati e streaming sono reimplementati (o assenti)
endpoint per endpoint.

### D16 — Segreto derivato in chiaro nel codice
`bam-studio-privato/api/workspace-session.js:14` contiene un `OWNER_TOKEN_SHA256`
hardcoded. È un hash, non il segreto, ma è comunque materiale di autenticazione
versionato: va spostato in variabile d'ambiente insieme al resto.

---

## 3. TARGET UNIFICATO

Una sola scelta per categoria. Il criterio non è "il migliore in assoluto" ma
"quello che è già in produzione e regge il carico", perché la migrazione più economica
è quella che non sposta la produzione.

| Categoria | Scelta | Perché (2 righe) | Costo di migrazione |
|---|---|---|---|
| **Linguaggio** | JavaScript CommonJS su Node | È già l'unico linguaggio applicativo su tutti e cinque i repository. Introdurre TypeScript ora aggiunge un build step a un frontend che oggi non ne ha uno. | Nullo |
| **Runtime** | **Node 24.x**, dichiarato in `engines` + `.nvmrc` in tutti i repo | 24 è la scelta più recente già in uso (studio) e resta LTS più a lungo di 22. Un solo numero elimina le differenze di comportamento tra ambienti. | Basso — allineare 3 manifest, verificare ffmpeg-static su 24 |
| **Package manager** | **npm con lockfile committato** ovunque | npm è già in uso; manca solo il lockfile. Cambiare gestore non risolve nulla che il lockfile non risolva. | Basso — `npm install` + commit in 4 repo |
| **Versioning dipendenze** | Range `^` espliciti, **zero `latest`** | `latest` rende ogni deploy non riproducibile. Le versioni vanno pinnate dove sono oggi, poi allineate. | Basso — 4 righe in studio |
| **Accesso al modello** | **`@anthropic-ai/sdk` ufficiale**, un solo modulo condiviso `lib/llm.js` | Elimina 40+ `fetch` con header scritto a mano; porta retry, errori tipizzati e streaming gratis. La scelta del modello passa da codice a configurazione. | Medio — riscrivere i punti di chiamata, comportamento invariato |
| **Fascia di modello** | Default unico da env (`BAM_LLM_MODEL`), `claude-sonnet-5` come valore iniziale | È già il modello dominante (giudici e dossier). Un default configurabile permette di alzare o abbassare fascia senza toccare il codice. | Basso — sostituzione di stringhe |
| **Piattaforma modello** | **API Anthropic diretta** per tutto | È il percorso di tutta la produzione; Bedrock esiste solo in studio, non validato, e costringe a due catene di credenziali. Bedrock resta un'opzione futura dietro lo stesso `lib/llm.js`. | Medio — studio adotta il modulo comune |
| **Provider media** | **fal.ai**, client `@fal-ai/client` pinnato | Già usato da tutti e quattro i repo per immagini e video. Replicate resta solo come proxy legacy e va ritirato. | Basso-medio |
| **Database + auth** | **Supabase**, un progetto per verticale, **con migrazioni SQL nel repo** | Il multi-progetto è corretto (isolamento dei dati per cliente); il problema è che lo schema non è versionato. | Medio — estrarre lo schema esistente, non modificarlo |
| **Storage oggetti** | **Supabase Storage** per il caldo, **S3** per l'archivio a freddo | È già la divisione in atto e funziona. Va solo resa esplicita e identica nei tre verticali. | Basso |
| **Coda / job** | **Inngest** | Già in produzione sui tre verticali con recupero dei job. Studio non ha alternativa: ha un vuoto. | Medio — studio adotta Inngest |
| **Montaggio video** | **ffmpeg** come motore unico, Creatomate ritirato | ffmpeg è dove sta il lavoro reale (43+8+9 file contro 12+6+7) e non ha costo per minuto renderizzato. Creatomate resta solo se studio dimostra un caso che ffmpeg non copre. | Alto — è il cambiamento più rischioso, va per ultimo |
| **Specifica di output** | **1920×1080, H.264 `libx264`, CRF 20, preset `medium`, 24 fps, `yuv420p`, AAC** — in un solo file di costanti | Sono i valori già più diffusi nella pipeline. Fissarli rende il video consegnato una specifica invece di un accidente. | Basso — file di costanti + sostituzioni |
| **Target di deploy** | **Vercel Functions** | Già unico per cinque repository su cinque. | Nullo |
| **Frontend** | **Vanilla JS + vendor self-hosted**, nessun CDN, nessun Babel a runtime | È la direzione che yacht ha già preso e validato (5 script contro 63). Toglie 4 dipendenze esterne dal percorso critico e rende possibile una CSP. | Medio — portare il pattern di yacht su auto e real-estate |
| **Sicurezza header** | **La CSP di studio**, applicata a tutti | Esiste già, scritta e funzionante. Va copiata, non progettata. | Basso — dopo la rimozione dei CDN |
| **Osservabilità** | **Sentry**, stessa versione ovunque (node ^10.64.0 + browser 10.65.0) | Già presente in tutti e quattro; manca solo l'allineamento. | Basso |
| **Test** | `node --test`, cartella `test/`, minimo i presidi di yacht in ogni verticale | `node --test` è già lo standard di fatto in yacht e studio, senza dipendenze aggiuntive. | Medio — auto e real-estate partono da zero |
| **CI** | **GitHub Actions**, un `verify.yml` identico in tutti i repo | Studio ha già il workflow giusto. Va replicato, non inventato. | Basso |
| **Codice condiviso** | **Un pacchetto `bam-core`** (profili verticali, llm, render, montaggio, giudici, specifica video) consumato dai tre verticali | È l'unica risposta strutturale a D1. I profili di studio (`vertical-profiles.js`) sono già il modello giusto: il verticale cambia terminologia e vincoli, non la pipeline. | Alto — ma è il fine, non l'inizio |

### Sul ruolo di BAM Studio

Studio dichiara di essere il motore universale, ma oggi è il repository meno
aggiornato, senza persistenza, senza coda, con limiti di funzione di 60 secondi contro
gli 800 dei verticali. **Il target giusto non è "migrare tutto dentro studio"**: è
estrarre `bam-core` dal codice di yacht — che è quello provato in produzione — e
adottare da studio le tre cose che studio fa meglio: i profili verticali, la CSP e la
disciplina dei test. Il verso della migrazione conta: yacht porta la pipeline, studio
porta la struttura.

---

## 4. PIANO DI MIGRAZIONE

Passi ordinati per rischio crescente. Ognuno è spedibile da solo, in un PR separato,
senza dipendere dai successivi.

---

### Passo 1 — Lockfile e pin delle versioni · rischio: minimo
**Cosa:** `npm install` e commit di `package-lock.json` in studio, auto, real-estate e
site; sostituzione dei quattro `"latest"` di studio con le versioni attualmente
risolte; `engines.node: "24.x"` + `.nvmrc` in tutti e cinque.
**Se sbaglio:** una build fallisce all'install e il deploy non parte. Nessun effetto
sui video già prodotti, rollback immediato con revert del lockfile.
**Verifica:** deploy di preview su ogni repo, `npm run verify` dove esiste.

### Passo 2 — Allineamento delle dipendenze condivise · rischio: minimo
**Cosa:** `@sentry/node` a `^10.64.0` ovunque (auto sale da 10.22), `ffmpeg-static` a
`^5.3.0` ovunque, versione Sentry browser identica.
**Se sbaglio:** l'unico rischio reale è ffmpeg-static: un cambio di build di ffmpeg può
alterare marginalmente l'output. Va verificato con un render di confronto prima del merge.
**Verifica:** `npm run confronto:prima` / `confronto:dopo` di yacht su una clip campione.

### Passo 3 — CI identica su tutti i repository · rischio: minimo
**Cosa:** copiare `verify.yml` di studio in yacht, auto, real-estate. Su yacht esegue
subito i 56 test esistenti; su auto e real-estate parte come solo `node --check`.
**Se sbaglio:** la CI diventa rossa su codice già in produzione. È un'informazione, non
un danno — ma va guardata prima di renderla bloccante.
**Verifica:** un PR di prova per repository.

### Passo 4 — Rimozione del codice morto · rischio: basso
**Cosa:** eliminare da yacht `render-safe-workflow-v5/v6/v7.js` e
`render-fidelity-judge-corrected.js` (zero referenze verificate). Decidere e unificare
i due workflow attivi (`render-safe-workflow.js` a 13 referenze contro `-v8.js` a 9) e
i due giudici di fedeltà — questa seconda parte è una decisione di prodotto, non
meccanica.
**Se sbaglio:** una referenza dinamica (costruita a stringa) non intercettata dal grep
rompe un percorso di render. Mitigazione: cercare anche le composizioni di stringhe
prima di cancellare, e spedire la cancellazione separata dall'unificazione.
**Verifica:** `npm test` di yacht + un render end-to-end.

### Passo 5 — Specifica video unica · rischio: basso-medio
**Cosa:** un file `lib/specifica-video.js` con la specifica di output (1080p, libx264,
CRF 20, preset medium, 24 fps, yuv420p, AAC); sostituzione di tutti i valori sparsi;
allineamento dei tre verticali agli stessi parametri; aggiunta del file musicale
mancante ad auto e real-estate.
**Se sbaglio:** i video cambiano peso e resa percepita. Il real-estate passa da 25 a
24 fps e va verificato che nessuna clip sorgente sia legata a 25. Da spedire con un
confronto visivo prima/dopo su almeno una clip per verticale.
**Verifica:** `collaudo:magenta` e `confronto:pixel` di yacht, estesi agli altri due.

### Passo 6 — Schema del database versionato · rischio: basso, ma irreversibile se sbagliato
**Cosa:** estrarre lo schema attuale dei tre progetti Supabase in
`supabase/migrations/0001_baseline.sql` per repository. **Solo lettura ed estrazione:
nessuna modifica allo schema in questo passo.**
**Se sbaglio:** se qualcuno applica per errore la baseline a un progetto già popolato,
si perdono dati. Mitigazione: la baseline nasce marcata come già applicata, e il
passo non introduce alcun comando di apply automatico.
**Verifica:** confronto dello schema estratto con la console Supabase, progetto per progetto.

### Passo 7 — Modulo LLM unico · rischio: medio
**Cosa:** introdurre `@anthropic-ai/sdk` e un `lib/llm.js` condiviso; migrare i punti
di chiamata (circa 20 file per verticale) mantenendo prompt e modelli **identici**;
spostare la scelta del modello su `BAM_LLM_MODEL`. Nessun cambio di comportamento
previsto in questo passo.
**Se sbaglio:** giudici e composizione prompt sono nel percorso critico: un errore
blocca la generazione. Mitigazione: migrare un endpoint alla volta, partendo da
`dossier-text.js` (il meno critico) e finendo con `inngest.js`.
**Verifica:** confronto degli output dei giudici su un set di render archiviati,
prima e dopo, a parità di input.

### Passo 8 — Frontend senza CDN e CSP attiva · rischio: medio
**Cosa:** portare su auto e real-estate il pattern già validato di yacht: vendor
self-hosted, bundle unificati (da 63 tag script a ~5), rimozione di
`@babel/standalone`, poi applicazione della CSP di studio.
**Se sbaglio:** una CSP troppo stretta rompe l'app in modo visibile e immediato per
tutti gli utenti. Mitigazione: spedire prima la rimozione dei CDN, poi la CSP in
`Content-Security-Policy-Report-Only` per una settimana, e solo dopo in enforcement.
**Verifica:** `verifica:funzionale` di yacht adattata, più un giro manuale su iPad
(il target dichiarato per la consegna dei master).

### Passo 9 — Test di base su auto e real-estate · rischio: basso, effort alto
**Cosa:** portare i presidi di yacht (`scripts/presidi.js`, 56 test) sui due verticali
scoperti, adattando i riferimenti verticali.
**Se sbaglio:** nulla in produzione. Il rischio è di scrivere test che passano senza
verificare niente. Vanno validati facendoli fallire di proposito.
**Verifica:** ogni test deve fallire se si rompe di proposito il codice che copre.

### Passo 10 — Estrazione di `bam-core` · rischio: alto
**Cosa:** estrarre da yacht il nucleo condiviso (profili verticali da studio, llm,
render, montaggio, giudici, specifica video) come pacchetto interno; far consumare
`bam-core` prima a real-estate (il più piccolo), poi ad auto, infine a yacht.
**Se sbaglio:** è il passo che può rompere tutti e tre i verticali insieme. Mitigazione:
un verticale alla volta, con il vecchio codice ancora presente e selezionabile da
variabile d'ambiente per almeno un ciclo di rilascio.
**Verifica:** render end-to-end completo per verticale, confrontato con l'output
pre-migrazione.

### Passo 11 — Motore di montaggio unico · rischio: il più alto
**Cosa:** ritirare Creatomate e portare tutto su ffmpeg (o la decisione opposta, se
studio dimostra un caso non coperto). Include il rientro di studio nella pipeline
comune: Inngest, Supabase, limiti di funzione a 800 s.
**Se sbaglio:** si perde la capacità di consegnare video. È l'unico passo che va
preceduto da una decisione esplicita e da un periodo con entrambi i motori attivi e
confrontati sullo stesso input.
**Verifica:** almeno 20 render in parallelo sui due motori, confronto di durata, peso,
continuità e resa prima dello spegnimento.

---

### Passi 12+ — decisioni aperte, non pianificabili senza una tua scelta

Tre punti richiedono una decisione di prodotto prima di poter essere pianificati:

1. **Il worker Modal di auto** (`identity-worker/`, Python + FastAPI): va portato dentro
   la pipeline comune, generalizzato agli altri verticali, o resta un'eccezione
   dichiarata? Oggi è un secondo linguaggio e un secondo provider per un solo verticale.
2. **Il ruolo di studio**: motore che assorbe i verticali, o libreria di struttura da
   cui i verticali prendono profili e disciplina? Questo audit assume la seconda; la
   prima cambierebbe l'ordine dei passi 10 e 11.
3. **Replicate**: il proxy `/v1/*` è ancora attivo in tutti e tre i verticali. Se è
   legacy va ritirato in un passo dedicato; se è ancora in uso va documentato come
   scelta.

E un punto di igiene da chiudere in ogni caso: il `OWNER_TOKEN_SHA256` hardcoded in
`bam-studio-privato/api/workspace-session.js:14` va spostato in variabile d'ambiente.

---

## 5. Sintesi

Lo stack non è disallineato perché sono state fatte scelte diverse: è disallineato
perché la **stessa** scelta è stata fatta tre volte in tre posti che poi hanno preso
strade diverse. La cura strutturale è un solo nucleo condiviso (passo 10); i nove passi
che lo precedono servono a rendere quel passo sicuro invece che temerario.

L'ordine conta più della velocità. I passi 1-4 si possono spedire questa settimana e
non toccano un solo pixel del video consegnato.
