# Architecture

Technical reference for AI Teams Config Hub internals.

## System overview

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
    algolia["Algolia (optional)"]
    resend["Resend (optional)"]

    browser -->|real-time listeners| firebase_client
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

The browser communicates with Firebase directly for real-time data (Firestore listeners) and with Next.js API Routes for mutations that require server-side validation. API Routes are thin handlers: auth guard, Zod validation, service call, HTTP response. The service layer contains all business logic and uses Firebase Admin SDK for Firestore batch operations. Algolia and Resend are optional, fire-and-forget integrations.

## Data model

Firestore collections and their relationships. Arrays (`channelIds`, `memberIds`) are stored as Firestore array fields.

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

Key relationships:

- **Dual-asset pattern**: when a PR is approved, a separate shared copy is created. The original asset links to the copy via `sharedAssetId`, and the copy references the source via `sourceAssetId`. This allows the owner to keep editing their private copy independently.
- **Ratings**: compound key `{userId}_{assetId}` prevents duplicates. Aggregate `rating.total`, `rating.count`, `rating.average` are updated via Firestore transaction to avoid race conditions.
- **Activity log**: always written as part of the same batch as the triggering operation, ensuring atomicity.

## Asset lifecycle

An asset goes through the following states, driven by PR submissions and admin reviews.

```mermaid
stateDiagram-v2
    [*] --> draft: Asset created
    draft --> pending: PR submitted
    pending --> approved: Admin approves
    pending --> rejected: Admin rejects
    approved --> pending: Update PR submitted
    rejected --> pending: New PR submitted
    approved --> [*]: Soft deleted
```

- `draft`: initial state, private to the owner
- `pending`: a PR has been submitted, awaiting admin review
- `approved`: admin approved, a shared copy exists in the channel
- `rejected`: admin rejected, owner can resubmit
- Soft delete sets `deletedByOwner: true` on the private copy; the shared copy remains visible

## Pull request review flow

The approval flow is the most complex operation. All Firestore writes happen in a single atomic batch.

```mermaid
sequenceDiagram
    participant U as User
    participant API as API Route
    participant S as Service Layer
    participant DB as Firestore
    participant A as Algolia

    U->>API: POST /review {approve}
    API->>API: withAuth + Zod validation
    API->>S: reviewPullRequest()
    S->>DB: batch.update(PR status)
    S->>DB: batch.update(asset status)
    S->>DB: batch.set(shared asset copy)
    S->>DB: batch.set(activity log)
    S->>DB: batch.commit()
    S-->>A: sync (fire-and-forget)
    S->>API: result
    API->>U: 200 OK
```

On rejection, the batch updates only the PR and asset status (no shared copy created).

## Authentication flow

Client-side auth via Firebase, server-side token verification via Admin SDK.

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
    FB->>AP: user profile doc
    AP->>Z: set(firebaseUser, userProfile)
    B->>API: request + Bearer token
    API->>ADM: verifyIdToken(token)
    ADM->>API: decoded token (uid)
    API->>API: withChannelAdmin check
```

- `AuthProvider` listens to Firebase Auth state changes and syncs the user profile to Zustand
- API Routes extract the Bearer token from the Authorization header and verify it with Admin SDK
- `withChannelAdmin` extends `withAuth` by checking `channel.adminId === uid`
- Admin is per-channel, not global: whoever creates a channel becomes its admin

## Email verification flow

After registration, users must verify their email before accessing the app. The verify-email page polls Firebase Auth every 3 seconds and redirects on success.

```mermaid
sequenceDiagram
    participant B as Browser
    participant FB as Firebase Auth
    participant V as Verify Email Page

    B->>FB: createUser(email, password)
    FB->>B: user (emailVerified: false)
    B->>V: redirect to /verify-email
    FB-->>B: verification email sent
    loop Every 3 seconds
        V->>FB: reload user
        FB->>V: emailVerified status
    end
    V->>V: emailVerified: true
    V->>B: redirect to /get-started or /invite/{code}
```

- If the user registered via an invite link, the invite code is preserved as a query parameter through the verification flow
- Users can request a new verification email (60-second cooldown between resends)
- The page provides a sign-out option to return to login

## Security rules

Access control is enforced at both the application layer (API route guards) and the database layer (Firestore and Storage security rules).

### Firestore rules

Each collection has specific read/write constraints defined in `firestore.rules`:

| Collection        | Read                       | Write                  | Notes                                                          |
| ----------------- | -------------------------- | ---------------------- | -------------------------------------------------------------- |
| `users`           | Owner only                 | Owner only             | No deletes allowed                                             |
| `channels`        | Members only               | Admin only (update)    | Anyone authenticated can create                                |
| `assets`          | Visibility-based           | Owner or admin         | `copyCount` update constrained to increment-only by any member |
| `assets/versions` | Owner or channel admin     | Owner only (create)    | Immutable after creation                                       |
| `pullRequests`    | Requester or channel admin | Channel admin (update) | No deletes allowed                                             |
| `activityLog`     | Channel members            | Creator only           | Immutable after creation                                       |
| `ratings`         | Rating owner               | Rating owner           | Compound key `{userId}_{assetId}` enforced                     |

Helper functions `isAuth()`, `isMemberOf(channelId)`, `isChannelAdmin(channelId)`, and `isOwner(resource)` centralize authorization checks.

### Storage rules

File uploads in `storage.rules` are restricted to:

- Authenticated users who are members of the target channel
- Maximum file size: 5 MB
- Allowed content types: `text/*`, `application/json`, `application/zip`
