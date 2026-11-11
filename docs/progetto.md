# Wiply Account – documento di progetto

> Stato: **bozza v0.6** (11 novembre 2026) · Responsabile: CEO Wiply Corp OÜ
> Repository: `WiplyCorp/accountfederation`

Questo documento descrive il sistema di account unico per tutti i prodotti Wiply:
cosa fa, come è fatto, quali dati tiene, come lo usano i prodotti e cosa resta da decidere.

---

## 1. Obiettivo

Una persona ha **un solo Wiply Account per sempre**. Da `account.wiply.net` può:

- gestire profilo e sicurezza (password, MFA, sessioni attive, login social);
- vedere e attivare/disattivare i servizi (es. Cassy e Studio sì, social no);
- creare o entrare in organizzazioni (aziende) e gestirne membri, ruoli e limiti;
- gestire pagamenti, abbonamenti e fatture.

Ogni prodotto Wiply rimanda all'Account per tutto questo, e all'accesso chiede
all'Account: *"questa persona, in questo contesto, ha diritto a questo servizio, e con quali limiti?"*

## 2. Prodotti coinvolti

| Prodotto | Indirizzo | Oggi | Domani |
|---|---|---|---|
| Cassy AI | cassy.wiply.net | LibreChat + LiteLLM, login Keycloak, piani venduti dal social | Personale o federato (contesto azienda), limiti per utente via LiteLLM |
| Wiply Assistant | assistant.wiply.net | Due fork: `assistant` e `wiply-voice`; password unica per il pannello, tenant su PostgreSQL | Tutto accorpato in `assistant`; file su S3 cifrati con KMS; fase 1b: login Keycloak, tenant = organizzazione, ruoli |
| Social | wiply.net | Sngine con utenti e pagamenti propri | Fork autonomo collegato al login unico (ultimo passo) |
| Narendi | – | Server di messaggistica su Matrix, chiuso (nessuna federazione con altri server, come Signal) | Login unico con Keycloak; chat private end-to-end per tutti (sez. 9.1); bot AI su invito (sez. 9.2) |
| Wiply Studio | – | – | Servizio attivabile dall'Account |
| SaaS futuri (gestionale…) | – | – | Nascono già sul modello Account |

## 3. Architettura

```
                    ┌────────────────────────────┐
   utente ─────────▶│ Keycloak (auth.wiply.net)  │  identità: login, MFA, social,
                    │ realm "wiply"              │  sessioni, Organizations
                    └─────────────┬──────────────┘
                                  │ OIDC (token con sub + organizzazione)
        ┌─────────────────────────┼───────────────────────────────┐
        ▼                         ▼                               ▼
  ┌───────────┐            ┌─────────────┐                 ┌─────────────┐
  │  Cassy    │            │  Assistant  │      ...        │  Studio     │
  └─────┬─────┘            └──────┬──────┘                 └──────┬──────┘
        │  "ha diritto?" (API + cache) / eventi di revoca         │
        └──────────────┬──────────┴──────────────────────────────┘
                       ▼
              ┌──────────────────────────┐        ┌──────────┐
              │ Wiply Account            │◀──────▶│  Stripe  │ abbonamenti, posti,
              │ (account.wiply.net)      │ webhook└──────────┘ consumi, fatture, IVA
              │ fonte di verità diritti  │
              │ PostgreSQL proprio       │───────▶ LiteLLM (team, budget, limiti)
              └──────────────────────────┘        Keycloak Admin API (org, membri)
```

### 3.1 Ruoli dei componenti

**Keycloak = identità (il "portiere").**
- Login unico, MFA, login social, sessioni, "esci ovunque".
- **Organizations** (funzione nativa di Keycloak 26+) per le aziende, con dominio email
  rivendicato (chi si registra con `@azienda.it` viene proposto all'organizzazione).
- Mette nel token: `sub` (id persona, stabile per sempre), email, organizzazioni di appartenenza.
- **Non** gestisce pagamenti, piani, servizi attivi o limiti.

**Wiply Account = fonte di verità su chi ha diritto a cosa.**
- Profilo esteso (dati di fatturazione, partita IVA, preferenze).
- Organizzazioni: membri, ruoli, inviti, posti assegnati.
- Servizi attivi per persona e per organizzazione, piani, limiti, consumi, fatture.
- Tiene Keycloak allineato (crea organizzazioni, aggiunge/toglie membri via Admin API)
  e LiteLLM allineato (team, utenti, budget).

**Stripe = soldi.** Abbonamenti, prezzo a posti, consumi a uso (token, minuti del centralino),
fatture, IVA UE con Stripe Tax / OSS, portale cliente. PayPal eventualmente come metodo aggiuntivo.

**I prodotti** fanno login con Keycloak e chiedono all'Account il diritto d'uso.
Non vendono nulla direttamente: il pulsante "Abbonati" porta all'Account.

### 3.2 Come un prodotto verifica il "servizio attivo"

Scelta proposta: **API dell'Account con cache breve + eventi di revoca**.

1. L'utente entra nel prodotto con Keycloak (OIDC). Il prodotto riceve `sub` e l'organizzazione scelta.
2. Il prodotto chiama `GET /v1/entitlements?subject=<sub>&org=<org_id>&service=cassy`
   (autenticato con credenziali del prodotto, *client credentials* di Keycloak).
3. L'Account risponde, ad esempio:
   ```json
   {
     "active": true,
     "plan": "cassy-business",
     "context": { "type": "organization", "id": "org_123" },
     "role": "member",
     "limits": { "tokens_per_day": 200000, "models": ["claude-sonnet", "titan-embed"] },
     "status": "active",
     "expires_at": "2026-12-11T00:00:00Z"
   }
   ```
4. Il prodotto tiene la risposta in cache per **60 secondi** (configurabile).
5. Quando un diritto cambia (dipendente rimosso, abbonamento scaduto, servizio sospeso)
   l'Account invia un **webhook firmato** ai prodotti, che svuotano la cache e chiudono le sessioni.

Perché non mettere i diritti dentro il token di Keycloak: i token vivono minuti o ore e non si
possono ritirare; i diritti cambiano con i pagamenti. Tenerli separati evita che Keycloak
dipenda dall'Account (e viceversa) per fare il login.

**Se l'Account non risponde (deciso):** il prodotto usa l'ultima risposta in cache fino a un
massimo di 15 minuti, poi nega l'accesso alle funzioni a pagamento.

## 4. Tipi di account

| Tipo | Chi | Pagamento | Nel sistema |
|---|---|---|---|
| Persona | privato | personale | `billing_account` di tipo `person` |
| Professionista | partita IVA, da solo | personale, fattura alla partita IVA | `person` con dati fiscali (P.IVA verificata VIES) |
| Organizzazione | azienda con dipendenti | centralizzato: a posti, piano fisso o consumi | `organization` (tipo `company`) + organizzazione Keycloak |
| Famiglia | genitore/tutore con figli minorenni | il genitore paga per tutti | `organization` (tipo `family`), vedi sezione 4.2 |

Regole:
- Una persona ha **un solo login per sempre** e può appartenere a **più organizzazioni**.
- Il professionista non è un tipo di utente diverso: è una persona con dati fiscali.
  Se poi assume dipendenti, crea un'organizzazione e ne diventa proprietario.
- Ogni abbonamento appartiene a un **conto di fatturazione** (`billing_account`), che è
  o una persona o un'organizzazione. Così lo stesso utente può avere un Cassy personale
  (pagato da sé) e un Cassy aziendale (pagato dall'azienda), separati.

### 4.1 Ruoli in un'organizzazione

| Ruolo | Può |
|---|---|
| `owner` | tutto, compresi pagamenti e cancellazione dell'organizzazione (almeno uno sempre presente) |
| `admin` | membri, inviti, servizi, limiti, vedere consumi |
| `billing` | pagamenti e fatture, nessun accesso ai dati dei servizi |
| `member` | usare i servizi che gli sono assegnati |

I prodotti possono avere ruoli propri in aggiunta (es. in Assistant: chi modifica la base di conoscenza).

### 4.2 Minori e account famiglia

Un minore sotto l'età del consenso digitale non può avere un account autonomo: deve
**agganciarsi all'account di un genitore o tutore**, che funziona come una piccola organizzazione
di tipo `family`.

**Ruoli nella famiglia**

| Ruolo | Può |
|---|---|
| `guardian` (genitore/tutore, maggiorenne) | autorizzare o negare ogni servizio, impostare limiti (orari, tempo, contenuti, modelli AI), vedere consumi e attività, pagare gli abbonamenti, revocare tutto in un clic |
| `minor` | usare solo i servizi autorizzati, entro i limiti impostati; non può pagare né lasciare la famiglia da solo |

Possono esserci due `guardian` (entrambi i genitori). Un `guardian` è anche una persona normale,
con i suoi servizi personali e le sue aziende.

**Regole d'età per Paese e per servizio (non per prodotto in blocco)**

L'Account tiene una tabella di regole, compilata e firmata da un legale:

| Campo | Significato |
|---|---|
| `country` | Paese di residenza dichiarato |
| `service_code` | social, cassy, studio, … |
| `age_alone` | da quest'età si usa il servizio da soli |
| `age_with_consent` | da quest'età si usa con l'autorizzazione del genitore |
| (sotto `age_with_consent`) | **vietato**, anche con il consenso del genitore |

Esempi solo indicativi, **da far verificare a un legale**: l'età del consenso digitale GDPR
è 14 anni in Italia, 13 in Estonia, 15 in Francia, 16 in Germania e Irlanda; in alcuni Paesi
(es. Australia, dal dicembre 2025) i social sono vietati sotto i 16 anni **senza eccezioni per il
consenso dei genitori**; in UE si discute un'età minima per i social uguale per tutti.

> ⚠️ **Limite importante.** L'autorizzazione del genitore vale **solo dove la legge la prevede**
> (cioè tra `age_with_consent` e `age_alone`). Non può far usare un servizio sotto un'età minima
> fissata dalla legge, e una liberatoria del genitore non toglie a Wiply gli obblighi verso i minori
> (GDPR, Digital Services Act: niente pubblicità profilata ai minori, impostazioni di privacy
> massime per default, ecc.). Il sistema quindi applica le regole in modo automatico e non lascia
> margine di "sblocco" al genitore sotto il minimo di legge. Vale anche per le nostre condizioni
> d'uso e per quelle dei fornitori dei modelli AI di Cassy, che possono fissare età minime proprie
> (da verificare).

**Flusso**
1. Alla registrazione Keycloak chiede la data di nascita e il Paese.
2. Se l'età è sotto `age_alone` per tutti i servizi richiesti, l'account nasce **in attesa del genitore**:
   il ragazzo inserisce l'email del genitore, che riceve un invito.
3. Il genitore entra (o si registra) e **dimostra di essere maggiorenne** tramite un servizio
   esterno di verifica dell'età (es. AgeVerif), vedi sotto.
4. Il genitore sceglie servizio per servizio cosa autorizzare e con quali limiti. Ogni consenso
   viene registrato (chi, quando, versione delle condizioni, metodo di verifica) e si può revocare.
5. Al compimento di `age_alone` il ragazzo viene avvisato e l'account diventa **autonomo**: il
   genitore perde la supervisione, i dati restano del ragazzo. Se il genitore pagava, l'abbonamento
   resta sul conto del genitore finché uno dei due non lo sposta.

**Verifica dell'età con servizi esterni (deciso).** L'Account non verifica documenti da sé: rimanda
a un fornitore esterno (es. AgeVerif) che conferma "maggiorenne sì/no" e restituisce l'esito firmato.
- Il codice parla con un'interfaccia unica (`age_verification_provider`), così si può cambiare
  fornitore o aggiungerne altri (es. il portafoglio d'identità digitale europeo) senza toccare il resto.
- L'Account salva **solo l'esito**: fornitore, metodo, risultato, data, riferimento della verifica.
  Nessun documento, selfie o data di nascita del genitore passa da noi.
- Lo stesso servizio si può usare dove la legge chiede di verificare l'età anche dei ragazzi
  (es. per confermare che non siano sotto il minimo assoluto).

**Supervisione, non sorveglianza (deciso).** Il genitore vede consumi, tempo d'uso, elenco dei
contatti e segnalazioni di sicurezza (senza contenuto), ma **non legge mai i messaggi privati**.
Le chat private dei minori sono end-to-end come quelle di tutti (sez. 9.1): nessuna chiave speciale.

**Regole strette sui contatti dei minori (deciso)**, applicate da Narendi e dal social:
- un minore riceve messaggi e inviti **solo da contatti approvati dal genitore**; niente messaggi
  da sconosciuti, nessuna richiesta in attesa visibile al minore senza approvazione;
- niente stanze pubbliche, salvo autorizzazione esplicita del genitore (Narendi è comunque chiuso: nessun server esterno);
- profilo privato e non ricercabile per default; nessuna pubblicità profilata;
- tasto "segnala" sempre visibile: l'app del minore invia a Wiply la conversazione segnalata
  (unico modo in cui vediamo un contenuto privato), il genitore riceve l'avviso senza il contenuto;
- il genitore può bloccare un contatto o sospendere la messaggistica in ogni momento.

**Dati in più nel modello**
```
person          + birthdate, country, age_status (adult|minor_supervised|minor_pending)
organization    + kind (company|family)
membership      + ruoli guardian | minor
age_rule        country, service_code, age_alone, age_with_consent, source, reviewed_at
guardian_consent
  id, guardian_person_id, minor_person_id, service_code, limits (jsonb),
  terms_version, verification_method, granted_at, revoked_at
age_verification
  id, person_id, provider, method, result (adult|not_adult|failed),
  provider_reference, verified_at
```
Nella risposta `entitlements` compare `"supervised": true` con i limiti del genitore, così ogni
prodotto applica le impostazioni per minori (es. nel social: profilo privato, niente messaggi da sconosciuti).

## 5. Modello dati (prima versione)

Database PostgreSQL proprio dell'Account. Gli id di Keycloak sono riferimenti, non copie.

```
person                    -- una per persona
  id (uuid), keycloak_sub (unico), email, display_name, locale,
  created_at, deleted_at

organization
  id, keycloak_org_id (unico), name, slug, verified_domains[],
  created_at, deleted_at

membership                -- persona dentro organizzazione
  person_id, organization_id, role (owner|admin|billing|member),
  status (invited|active|suspended), invited_by, created_at

invitation
  id, organization_id, email, role, token_hash, expires_at, accepted_at

billing_account           -- chi paga: una persona O un'organizzazione
  id, owner_type (person|organization), owner_id,
  legal_name, vat_id, vat_verified_at, country, address,
  stripe_customer_id

service                   -- catalogo: cassy, assistant, studio, social, ...
  code, name, status (live|beta|retired)

plan                      -- piani vendibili, legati a prezzi Stripe
  code, service_code, pricing_model (flat|per_seat|usage|hybrid),
  stripe_price_ids[], default_limits (jsonb)

subscription              -- specchio dell'abbonamento Stripe
  id, billing_account_id, plan_code, stripe_subscription_id,
  status (trialing|active|past_due|paused|canceled),
  seats, current_period_end, cancel_at

entitlement               -- IL dato che i prodotti leggono
  id, service_code, subject_type (person|membership),
  subject_id, organization_id (null se personale),
  subscription_id (null se gratuito/manuale),
  status (active|paused|pending_deletion|revoked),
  limits (jsonb), valid_from, valid_until

usage_record              -- consumi, aggregati per giorno
  billing_account_id, service_code, person_id, metric
  (tokens_in|tokens_out|call_minutes|...), day, quantity,
  reported_to_stripe_at

audit_log                 -- chi ha fatto cosa (obbligatorio per admin e GDPR)
  id, actor_person_id, action, target_type, target_id, data (jsonb), at

webhook_endpoint / outbox -- eventi verso i prodotti, con ritentativi
```

Note:
- **`entitlement` è separato da `subscription`**: un diritto può nascere da un abbonamento,
  da una prova gratuita, da una concessione manuale (partner, staff) o dalla migrazione dal social.
- In un'organizzazione con prezzo a posti, ogni `membership` con il servizio assegnato occupa
  un posto; l'Account aggiorna la quantità su Stripe.

## 6. Flussi principali

### 6.1 Registrazione e primo accesso
1. L'utente va su un prodotto qualsiasi → "Accedi" → Keycloak (registrazione o login, anche social).
2. Al primo accesso l'Account crea la `person` (via evento Keycloak o al primo contatto del prodotto).
3. Il prodotto chiede il diritto d'uso. Se non c'è: pagina "Attiva il servizio" che porta all'Account.

### 6.2 Creare un'organizzazione e invitare membri
1. Dall'Account: "Crea organizzazione" → nome, dati fiscali → diventa `owner`.
2. L'Account crea l'organizzazione anche in Keycloak (Admin API).
3. Invito per email → la persona accetta (con login esistente o nuovo) → `membership` attiva
   → aggiunta all'organizzazione Keycloak.
4. Facoltativo: dominio email verificato → i nuovi utenti con quel dominio vedono la proposta di unirsi.

### 6.3 Attivare un servizio con pagamento
1. L'utente (o l'admin dell'organizzazione) sceglie il piano nell'Account.
2. Stripe Checkout (con Stripe Tax; P.IVA UE valida → inversione contabile).
3. Webhook Stripe → l'Account crea/aggiorna `subscription` ed `entitlement` → evento ai prodotti.

### 6.4 Togliere un dipendente
1. L'admin rimuove il membro dall'organizzazione nell'Account.
2. In un'unica operazione: `membership` sospesa, `entitlement` aziendali revocati,
   rimozione dall'organizzazione Keycloak, utente LiteLLM bloccato, posto liberato su Stripe,
   webhook ai prodotti (che chiudono le sessioni in quel contesto).
3. Il login della persona **resta**: perde solo il contesto di quell'azienda.
   Il suo eventuale Cassy personale non viene toccato.

### 6.5 Disattivare un servizio (deciso: pausa e cancellazione come scelte separate)
- **Pausa**: `entitlement.status = paused`, dati conservati senza limite di tempo, nessun addebito (o addebito ridotto).
- **Cancellazione**: `pending_deletion` per un periodo di ripensamento di 30 giorni,
  poi il prodotto riceve l'evento `service.purge` e cancella i dati. L'audit conserva solo il fatto.

### 6.6 Cancellazione dell'account (GDPR)
Richiesta dall'Account → verifica che non sia l'unico `owner` di un'organizzazione con abbonamenti →
periodo di ripensamento → evento `person.purge` a tutti i prodotti → cancellazione in Keycloak.
Le fatture restano conservate per obbligo fiscale (dati minimi).

## 7. Cassy federato e limiti tramite LiteLLM

Mappatura:

| Wiply Account | LiteLLM |
|---|---|
| Organizzazione con Cassy | **Team** con budget complessivo |
| Membro con Cassy aziendale | **Utente** del team, con limiti propri (budget, token/giorno, modelli permessi) |
| Persona con Cassy personale | Utente senza team (o team personale), separato |

- L'Account crea team e utenti con l'API di amministrazione di LiteLLM e ne aggiorna i limiti
  quando l'admin li cambia.
- I consumi si leggono da LiteLLM (spesa per utente/team), si salvano in `usage_record`
  e si riportano a Stripe se il piano è a consumo.
- In LibreChat il "contesto azienda" separa chat e documenti dell'azienda da quelli personali.
  **Da verificare tecnicamente**: come LibreChat gestisce più contesti per lo stesso utente
  (probabilmente un'identità separata per contesto o un'istanza per organizzazione grande).
  Il contenitore `lobe-chat-vectors-1` (pgvector) non va toccato.

## 8. Wiply Assistant – fase 1b sul nuovo modello

Primo prodotto a nascere sul modello, prima ancora che l'Account sia completo.

**Accorpamento (deciso):** oggi esistono due fork, `assistant` e `wiply-voice`; tutto verrà
accorpato in `assistant`. Proposta d'ordine: login e tenant della fase 1b si fanno direttamente
in `assistant`, così `wiply-voice` confluisce in un codice che ha già il modello giusto
(una sola migrazione invece di due).

**Archiviazione su S3 con chiave KMS (deciso).** Proposte di dettaglio:
- bucket nella regione di Parigi (`eu-west-3`), accesso pubblico bloccato, cifratura SSE-KMS
  con chiave gestita da noi;
- file organizzati per tenant (`tenant/<id>/...`) e contesto di cifratura con l'id del tenant,
  così un tenant non può mai leggere i file di un altro, nemmeno per errore nel codice;
- valutare **una chiave KMS per organizzazione** (circa 1 $ al mese ciascuna): alla cancellazione
  definitiva di un servizio basta eliminare la chiave e tutti i file diventano illeggibili
  ("crypto-shredding"). AWS attende da 7 a 30 giorni prima di eliminare una chiave, il che si
  sposa con i 30 giorni di ripensamento decisi (sez. 6.5);
- registrazioni del centralino e documenti della base di conoscenza con scadenza automatica
  configurabile per tenant.

Passi della fase 1b:

1. Login con Keycloak (client OIDC `assistant` nel realm `wiply`) al posto della password unica.
2. Ogni tenant di `assistant-db` viene collegato a un'**organizzazione Keycloak** (colonna `keycloak_org_id`).
3. Ruoli `admin` e `member` letti dall'organizzazione (in attesa dell'Account, da Keycloak;
   poi dall'Account).
4. Controllo "servizio attivo": un modulo unico nel codice di Assistant che oggi legge una tabella
   locale e domani chiama `GET /v1/entitlements`. Così il passaggio all'Account non tocca il resto del codice.
5. Consumi (minuti del centralino, messaggi chat) registrati in un formato già compatibile con `usage_record`.

## 9. Sicurezza e affidabilità

**Keycloak diventa un punto unico critico** (deciso: prima si collauda, poi gli si affida il compito vero).

Prove da superare **prima** di collegare Assistant e l'Account a Keycloak in produzione,
su un ambiente di prova (copia del realm, nessun utente reale):
1. ripristino completo da backup su un server pulito, con tempo misurato;
2. spegnimento improvviso di Keycloak: cosa vedono i prodotti, quanto tornano su da soli;
3. aggiornamento di versione di Keycloak con ritorno indietro;
4. carico: molti login contemporanei;
5. avvisi di monitoraggio che arrivano davvero (email/telefono).

Requisiti permanenti:
- backup automatici del database di Keycloak (giornalieri + prima di ogni aggiornamento), con prova di ripristino;
- monitoraggio esterno di `auth.wiply.net` e avvisi;
- aggiornamenti di sicurezza regolari, console di amministrazione non esposta pubblicamente (o protetta da IP/VPN);
- valutare un secondo nodo (Keycloak in cluster) o almeno un server di riserva pronto.

Altre regole:
- segreti solo in `/etc/wiply/*.env` (modo 0640), mai nel repository;
- i prodotti parlano con l'Account con credenziali proprie (*client credentials*), con permessi minimi;
- webhook firmati (HMAC) in entrambe le direzioni (Stripe → Account, Account → prodotti);
- ogni azione di amministrazione finisce in `audit_log`;
- dati in UE (Parigi), coerente con il GDPR.

### 9.1 Messaggi privati e richieste delle autorità (deciso)

**Posizionamento:** rispettiamo la legge europea e collaboriamo con le autorità per ciò che abbiamo,
ma i sistemi sono costruiti per **non avere** i messaggi degli utenti: nemmeno Wiply può leggerli.

**Chat private (Narendi / Matrix, e messaggi privati del social): end-to-end vero, per tutti, minori compresi.**
- Si usa la cifratura standard di Matrix (Olm/Megolm), attiva per default su tutte le chat private.
- Le chiavi esistono solo sui dispositivi degli utenti; il backup delle chiavi sul server è
  cifrato con la chiave di recupero dell'utente, che noi non abbiamo.
- Nessuna chiave di accesso per Wiply, per i genitori o per le autorità.
- Il social (fork di Sngine) userà Narendi per i messaggi privati, invece di una messaggistica propria.
- Il contenuto privato arriva a Wiply in un solo caso: quando un utente **segnala** una conversazione,
  la sua app ci invia i messaggi segnalati.

**Cosa possiamo consegnare con un ordine valido** (tribunale o ordine europeo "e-Evidence",
regolamento UE 2023/1543): solo ciò che abbiamo.
- Dati dell'account, indirizzi IP, orari di accesso, eventualmente con chi si è comunicato.
- Post pubblici, e contenuti non cifrati end-to-end come le **chat di Cassy** (un'AI deve leggere
  i messaggi per rispondere: sono cifrate sul server, non end-to-end).
- Regola di **minimizzazione**: conservare il meno possibile e per il minor tempo possibile
  (es. log di IP e accessi con scadenza breve), così anche ciò che possiamo consegnare è poco.

**Procedura**
1. La richiesta viene registrata in `lawful_access_request`.
2. Il legale verifica validità, autorità e perimetro; richieste senza base legale valida vengono
   respinte e registrate.
3. Estrazione solo del perimetro richiesto, con doppia approvazione (due persone), in un registro
   non modificabile.
4. L'utente viene avvisato a pratica chiusa, salvo divieto dell'autorità.
5. Rapporto di trasparenza annuale: richieste ricevute, accolte, respinte.

**Comunicazione:** per le chat private si può dire "end-to-end" e "nemmeno noi possiamo leggerle".
Per Cassy no: si dirà "cifrate e accessibili solo con procedura legale". Testi pubblici da far
rivedere al legale.

**Da seguire:** il regolamento europeo sugli abusi sui minori ("Chat Control") è ancora in discussione;
nella versione approvata dal Consiglio a fine 2025 la scansione dei messaggi è volontaria.
Se l'obbligo dovesse tornare, la scelta va rivista.

**Da verificare su Narendi:** cifratura attiva per default, dati e log conservati e per quanto,
federazione effettivamente disattivata, collegamento a Keycloak.

```
lawful_access_request
  id, authority, legal_basis, document_ref, received_at,
  scope (jsonb: person, tipo di dati, periodo), legal_review_by, decision (accepted|rejected),
  approved_by[], executed_at, delivered_at, user_notified_at, notes
```

### 9.2 Bot AI nelle chat di Narendi (idea, da sviluppare a parte)

Bot AI (supporto, assistenza) invitabili nelle chat personali e di gruppo. Da progettare in una
sessione dedicata; qui solo i principi che lo rendono compatibile con l'end-to-end:
- **Un bot che legge una chat la rende leggibile dai nostri server** (il bot è un partecipante e
  manda i testi all'AI tramite LiteLLM). Va detto chiaramente: in una chat con un bot non vale
  "nemmeno noi possiamo leggerla" per i messaggi che il bot riceve.
- I bot **non li mandiamo noi**: li chiedono e li aggiungono gli utenti, che decidono di ammetterli
  conoscendone i rischi. Al momento dell'invito un avviso breve spiega cosa il bot può leggere.
- Il bot è sempre visibile come membro con un'etichetta "AI", nei gruppi tutti i partecipanti
  vengono avvisati quando entra, e può essere tolto in ogni momento.
- Preferibile che il bot elabori **solo i messaggi rivolti a lui** (menzione o comando), non tutta la chat.
- Nessuna conservazione dei testi oltre il necessario; nessun uso per addestrare modelli.
- Minori: bot solo se il genitore lo autorizza e se le regole d'età dei servizi AI lo permettono.
- Consumi (token) addebitati a chi ha invitato il bot o alla sua organizzazione, tramite l'Account
  (servizio `narendi-ai`).

## 10. Migrazione da Cassy sul social (decisione aperta 3)

Proposta in tre passi:
1. **Importazione**: per ogni utente Cassy oggi attivo, l'Account crea `person` ed `entitlement`
   con origine `legacy_social` e scadenza pari al periodo già pagato.
2. **Convivenza**: il collegamento firmato `cassy_plans` dal social continua a funzionare, ma
   scrive nell'Account invece che direttamente in Cassy.
3. **Passaggio**: al rinnovo, l'utente viene portato su Stripe tramite l'Account
   (gli abbonamenti PayPal/Stripe del social non si spostano automaticamente: si chiede di riabbonarsi
   con un incentivo, oppure si migrano i clienti Stripe se l'account Stripe è lo stesso).

## 11. Tecnologia proposta per il servizio Account

- Backend: stesso linguaggio e struttura di Wiply Assistant, per avere un'unica squadra e un unico stile.
- Database: PostgreSQL dedicato (contenitore separato, backup separati).
- Interfaccia: `account.wiply.net`, testi in inglese con file di traduzione.
- Deploy: Docker sul server cassyai (aaPanel come reverse proxy), poi eventualmente server proprio.
- API versionata (`/v1/...`), documentata con OpenAPI.

## 12. Decisioni aperte

| # | Domanda | Proposta | Stato |
|---|---|---|---|
| 1 | Disattivare = pausa o cancellazione? | Entrambe, come scelte separate: pausa senza limiti di tempo; cancellazione con 30 giorni di ripensamento | **deciso** |
| 2 | Keycloak punto unico critico | Collaudo su ambiente di prova (sez. 9) prima di affidargli il lavoro vero; backup e monitoraggio permanenti; secondo nodo prima del social | **deciso** |
| 3 | Migrazione clienti Cassy dal social | Importazione + convivenza + passaggio al rinnovo (sez. 10) | da decidere |
| 4 | Prezzi: posti, consumo, piano fisso | Assistant: piano fisso + minuti a consumo; Cassy aziendale: posti + token oltre soglia; SaaS gestionale: posti | da decidere |
| 5 | Cassy federato: più contesti in LibreChat | Prova tecnica prima della fase 4 | da verificare |
| 6 | Se l'Account non risponde | Cache fino a 15 minuti, poi blocco delle funzioni a pagamento | **deciso** |
| 7 | PayPal | Solo se richiesto dai clienti, dopo il lancio di Stripe | da decidere |
| 8 | Minori: cosa vede il genitore | Mai i messaggi privati; contatti, consumi e segnalazioni sì; regole strette sui contatti (sez. 4.2). Tabella d'età per Paese ancora da far compilare a un legale | **deciso** (tabella d'età: da fare) |
| 10 | Messaggi privati e autorità | End-to-end vero per tutti su Narendi (Matrix), minori compresi; alle autorità solo metadati, contenuti pubblici e chat di Cassy (sez. 9.1) | **deciso** |
| 9 | Verifica che il genitore sia maggiorenne | Servizi esterni (es. AgeVerif), interfaccia unica, salviamo solo l'esito | **deciso** |

## 13. Ordine di lavoro

1. **Questo documento** (architettura, modello dati, flussi, domande aperte). ← siamo qui
2. Collaudo di Keycloak su ambiente di prova (sez. 9), poi Wiply Assistant fase 1b sul modello (sez. 8).
3. Wiply Account v1: profilo, organizzazioni con membri e inviti, servizi attivi, API `entitlements`;
   data di nascita e Paese raccolti fin da subito; account famiglia prima dell'apertura del social ai minori.
4. Cassy federato con limiti per utente tramite LiteLLM (sez. 7).
5. Pagamenti Stripe dall'Account: abbonamenti, posti, consumi, IVA.
6. Social: fork autonomo di Sngine collegato al login unico.
