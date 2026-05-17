# Architettura

Riferimento tecnico per gli interni di AI Teams Config Hub.

## Panoramica del sistema

```mermaid
%%{init: {'theme': 'default'}}%%
graph LR
    browser["Browser (React + Zustand)"]
    firebase_client["Firebase Client SDK"]
    api_routes["Next.js API Routes"]
    services["Service Layer"]
    admin_sdk["Firebase Admin SDK"]
    firestore[("Firestore")]
    storage[("Storage")]
    algolia["Algolia (opzionale)"]
    resend["Resend (opzionale)"]

    browser -->|listener real-time| firebase_client
    browser -->|REST + Bearer token| api_routes
    firebase_client --> firestore
    api_routes -->|withAuth + Zod| services
    services --> admin_sdk
    admin_sdk --> firestore
    admin_sdk --> storage
    services -.->|fire-and-forget| algolia
    services -.->|fire-and-forget| resend

    class browser,api_routes,services core
    class firestore,storage data
    class algolia,resend ext
    class firebase_client,admin_sdk engine

    classDef core fill:#2563eb,stroke:#1d4ed8,color:#fff
    classDef data fill:#d97706,stroke:#b45309,color:#fff
    classDef ext fill:#6b7280,stroke:#4b5563,color:#fff
    classDef engine fill:#059669,stroke:#047857,color:#fff
```

Il browser comunica con Firebase direttamente per i dati real-time (listener Firestore) e con le API Routes di Next.js per le mutazioni che richiedono validazione server-side. Le API Routes sono thin handler: auth guard, validazione Zod, chiamata al service, response HTTP. Il service layer contiene tutta la business logic e usa Firebase Admin SDK per operazioni batch su Firestore. Algolia e Resend sono integrazioni opzionali, fire-and-forget.

## Modello dati

Collezioni Firestore e relative relazioni. Gli array (`channelIds`, `memberIds`) sono campi array di Firestore.

```mermaid
erDiagram
    users {
        string uid PK
        string email
        string displayName
        string channelIds
    }
    channels {
        string id PK
        string name
        string adminId FK
        string memberIds
        string inviteCode
    }
    assets {
        string id PK
        string title
        string type
        string visibility
        string status
        string ownerId FK
        string channelId FK
        string sharedAssetId FK
    }
    pullRequests {
        string id PK
        string assetId FK
        string channelId FK
        string requesterId FK
        string status
    }
    activityLog {
        string id PK
        string channelId FK
        string userId FK
        string action
    }
    ratings {
        string id PK
        string userId FK
        string assetId FK
        number score
    }

    users ||--o{ channels : "adminId"
    users }o--o{ channels : "memberIds"
    users ||--o{ assets : "owns"
    channels ||--o{ assets : "contains"
    assets ||--o| assets : "sharedAssetId"
    assets ||--o{ pullRequests : "assetId"
    channels ||--o{ pullRequests : "channelId"
    users ||--o{ pullRequests : "requests"
    channels ||--o{ activityLog : "channelId"
    users ||--o{ ratings : "rates"
    assets ||--o{ ratings : "rated"
```

Relazioni chiave:

- **Pattern dual-asset**: quando una PR viene approvata, viene creata una copia condivisa separata. L'asset originale punta alla copia tramite `sharedAssetId`, e la copia referenzia la sorgente tramite `sourceAssetId`. L'owner continua a modificare la copia privata indipendentemente.
- **Ratings**: chiave composta `{userId}_{assetId}` per evitare duplicati. Gli aggregati `rating.total`, `rating.count`, `rating.average` vengono aggiornati via transaction Firestore per evitare race condition.
- **Activity log**: scritto sempre nella stessa batch dell'operazione che lo genera, garantendo atomicita.

## Ciclo di vita degli asset

Un asset attraversa i seguenti stati, guidato dall'invio di PR e dalle review dell'admin.

```mermaid
stateDiagram-v2
    [*] --> bozza: Asset creato
    bozza --> in_attesa: PR inviata
    in_attesa --> approvato: Admin approva
    in_attesa --> rifiutato: Admin rifiuta
    approvato --> in_attesa: PR di aggiornamento
    rifiutato --> in_attesa: Nuova PR
    approvato --> [*]: Eliminazione soft
```

- `bozza`: stato iniziale, privato per l'owner
- `in_attesa`: una PR e stata inviata, in attesa di review dall'admin
- `approvato`: l'admin ha approvato, una copia condivisa esiste nel canale
- `rifiutato`: l'admin ha rifiutato, l'owner puo reinviare
- La soft delete imposta `deletedByOwner: true` sulla copia privata; la copia condivisa resta visibile

## Flusso review pull request

Il flusso di approvazione e l'operazione piu complessa. Tutte le scritture su Firestore avvengono in un singolo batch atomico.

```mermaid
sequenceDiagram
    participant U as Utente
    participant API as API Route
    participant S as Service Layer
    participant DB as Firestore
    participant A as Algolia

    U->>API: POST /review {approve}
    API->>API: withAuth + validazione Zod
    API->>S: reviewPullRequest()
    S->>DB: batch.update(stato PR)
    S->>DB: batch.update(stato asset)
    S->>DB: batch.set(copia condivisa)
    S->>DB: batch.set(activity log)
    S->>DB: batch.commit()
    S-->>A: sync (fire-and-forget)
    S->>API: risultato
    API->>U: 200 OK
```

In caso di rifiuto, la batch aggiorna solo lo stato della PR e dell'asset (nessuna copia condivisa creata).

## Flusso di autenticazione

Auth lato client via Firebase, verifica token lato server via Admin SDK.

```mermaid
sequenceDiagram
    participant B as Browser
    participant AP as AuthProvider
    participant FB as Firebase Auth
    participant Z as Zustand Store
    participant API as API Route
    participant ADM as Admin SDK

    B->>FB: signIn(email, password)
    FB->>AP: onAuthStateChanged(user)
    AP->>FB: onSnapshot(users/{uid})
    FB->>AP: documento profilo utente
    AP->>Z: set(firebaseUser, userProfile)
    B->>API: richiesta + Bearer token
    API->>ADM: verifyIdToken(token)
    ADM->>API: token decodificato (uid)
    API->>API: verifica withChannelAdmin
```

- `AuthProvider` ascolta i cambiamenti di stato Firebase Auth e sincronizza il profilo utente su Zustand
- Le API Routes estraggono il Bearer token dall'header Authorization e lo verificano con Admin SDK
- `withChannelAdmin` estende `withAuth` controllando `channel.adminId === uid`
- L'admin e per-canale, non globale: chi crea un canale ne diventa admin

## Flusso verifica email

Dopo la registrazione, gli utenti devono verificare la propria email prima di accedere all'app. La pagina di verifica email interroga Firebase Auth ogni 3 secondi e reindirizza al completamento.

```mermaid
sequenceDiagram
    participant B as Browser
    participant FB as Firebase Auth
    participant V as Pagina Verifica Email

    B->>FB: createUser(email, password)
    FB->>B: utente (emailVerified: false)
    B->>V: redirect a /verify-email
    FB-->>B: email di verifica inviata
    loop Ogni 3 secondi
        V->>FB: ricarica utente
        FB->>V: stato emailVerified
    end
    V->>V: emailVerified: true
    V->>B: redirect a /get-started o /invite/{code}
```

- Se l'utente si e registrato tramite un invite link, il codice invito viene preservato come parametro query durante il flusso di verifica
- Gli utenti possono richiedere una nuova email di verifica (cooldown di 60 secondi tra i reinvii)
- La pagina offre un'opzione di logout per tornare al login

## Regole di sicurezza

Il controllo accessi e applicato sia a livello applicativo (guard nelle API route) che a livello database (regole di sicurezza Firestore e Storage).

### Regole Firestore

Ogni collezione ha vincoli specifici di lettura/scrittura definiti in `firestore.rules`:

| Collezione        | Lettura                         | Scrittura                        | Note                                                                       |
| ----------------- | ------------------------------- | -------------------------------- | -------------------------------------------------------------------------- |
| `users`           | Solo proprietario               | Solo proprietario                | Eliminazione non consentita                                                |
| `channels`        | Solo membri                     | Solo admin (aggiornamento)       | Chiunque autenticato puo creare                                            |
| `assets`          | Basata sulla visibilita         | Proprietario o admin             | Aggiornamento `copyCount` vincolato al solo incremento da qualsiasi membro |
| `assets/versions` | Proprietario o admin del canale | Solo proprietario (creazione)    | Immutabili dopo la creazione                                               |
| `pullRequests`    | Richiedente o admin del canale  | Admin del canale (aggiornamento) | Eliminazione non consentita                                                |
| `activityLog`     | Membri del canale               | Solo creatore                    | Immutabili dopo la creazione                                               |
| `ratings`         | Proprietario del rating         | Proprietario del rating          | Chiave composta `{userId}_{assetId}` applicata                             |

Le funzioni helper `isAuth()`, `isMemberOf(channelId)`, `isChannelAdmin(channelId)` e `isOwner(resource)` centralizzano i controlli di autorizzazione.

### Regole Storage

Gli upload file in `storage.rules` sono limitati a:

- Utenti autenticati che sono membri del canale di destinazione
- Dimensione massima file: 5 MB
- Content type consentiti: `text/*`, `application/json`, `application/zip`
