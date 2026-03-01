# Yaḥḍuru — Guide d'implémentation MVP
**Équipe 2b1b · Holberton School Thonon-les-Bains · Hackathon Rabhacks**

---

## Table des matières

1. [Vue d'ensemble technique](#1-vue-densemble-technique)
2. [Prérequis et mise en place de l'environnement](#2-prérequis-et-mise-en-place-de-lenvironnement)
3. [Structure du dépôt](#3-structure-du-dépôt)
4. [Base de données](#4-base-de-données)
5. [Backend — API REST](#5-backend--api-rest)
6. [Moteur Ping](#6-moteur-ping)
7. [Frontend — PWA Praticien](#7-frontend--pwa-praticien)
8. [Frontend — Interface Hub](#8-frontend--interface-hub)
9. [Notifications push et fallback SMS](#9-notifications-push-et-fallback-sms)
10. [Sécurité et protection des données](#10-sécurité-et-protection-des-données)
11. [Ordre d'implémentation recommandé](#11-ordre-dimplémentation-recommandé)
12. [Variables d'environnement](#12-variables-denvironnement)
13. [Tests](#13-tests)
14. [Déploiement](#14-déploiement)

---

## 1. Vue d'ensemble technique

Yaḥḍuru est une plateforme **mobile-first, offline-first** à trois niveaux :

```
[ Praticien TCIM ]  ←→  Ping / Formation / Signalement
        │
        ▼
[ Hub Association ]  ←→  Validation / Coordination / Anonymisation
        │
        ▼
[ API Backend ]  ←→  BDD sécurisée  ←→  [ Institution / Dashboard ]
```

### Stack MVP

| Couche | Technologie |
|--------|-------------|
| PWA Praticien | React + Vite + Workbox |
| Dashboard Hub & Institution | React (composants partagés) |
| API | Node.js + Express |
| Base de données | PostgreSQL 15 + PostGIS |
| Auth | JWT (access 1h / refresh 7j) + bcrypt |
| Push notifications | Web Push API (VAPID) |
| SMS fallback | Twilio |
| Hébergement | VPS (OVH ou équivalent souverain) |
| CI/CD | GitHub Actions |

---

## 2. Prérequis et mise en place de l'environnement

### Dépendances système

```bash
node >= 18.x
npm >= 9.x
postgresql >= 15
postgis >= 3.3
redis >= 7      # file de jobs pour le dispatch Ping
```

### Installation

```bash
# Cloner le dépôt
git clone https://github.com/2b1b-team/yahduru.git
cd yahduru

# Backend
cd backend
npm install
cp .env.example .env   # remplir les variables (voir Section 12)

# Frontend PWA
cd ../frontend/pwa
npm install

# Frontend Dashboard
cd ../frontend/dashboard
npm install
```

### Base de données

```bash
# Créer la base
createdb yahduru_dev

# Activer PostGIS
psql yahduru_dev -c "CREATE EXTENSION IF NOT EXISTS postgis;"
psql yahduru_dev -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"

# Appliquer les migrations
cd backend
npm run db:migrate
npm run db:seed:dev   # données de test uniquement
```

---

## 3. Structure du dépôt

```
yahduru/
├── backend/
│   ├── src/
│   │   ├── config/          # env, database, redis
│   │   ├── middleware/      # auth, rbac, error-handler, audit-logger
│   │   ├── modules/
│   │   │   ├── auth/        # inscription, login, refresh token
│   │   │   ├── practitioners/
│   │   │   ├── hubs/
│   │   │   ├── training/
│   │   │   ├── ping/        # moteur Ping (voir Section 6)
│   │   │   ├── referrals/
│   │   │   ├── institutions/
│   │   │   └── analytics/
│   │   ├── services/
│   │   │   ├── push.service.js
│   │   │   ├── sms.service.js
│   │   │   └── geo.service.js
│   │   └── app.js
│   ├── migrations/
│   ├── seeds/
│   └── tests/
├── frontend/
│   ├── pwa/                 # PWA praticien
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── pages/
│   │   │   ├── stores/      # Zustand ou Redux Toolkit
│   │   │   ├── sw/          # Service Worker (Workbox)
│   │   │   └── offline/     # IndexedDB helpers
│   │   └── public/
│   └── dashboard/           # Hub + Institution
│       └── src/
│           ├── components/
│           ├── pages/
│           └── stores/
└── shared/
    └── types/               # types TypeScript partagés (optionnel)
```

---

## 4. Base de données

### Modèle de données principal

> Les entités de données de santé (`ping_events`, `referrals`) sont physiquement séparées des données d'identité (`users`, `practitioner_profiles`). Un `practitioner_id` dans `referrals` est toujours un UUID opaque — jamais une jointure directe exposée à l'institution.

#### `users`
```sql
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL CHECK (role IN ('PRACTITIONER','HUB_SUPERVISOR','INSTITUTION','ADMIN')),
    hub_id        UUID REFERENCES hubs(id),
    is_active     BOOLEAN DEFAULT false,
    created_at    TIMESTAMPTZ DEFAULT now(),
    last_login_at TIMESTAMPTZ
);
```

#### `hubs`
```sql
CREATE TABLE hubs (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name           TEXT NOT NULL,
    type           TEXT NOT NULL CHECK (type IN ('ASSOCIATION','COOPERATIVE','NGO','COMMUNITY_GROUP')),
    coverage_zone  GEOMETRY(POLYGON, 4326),
    contact_email  TEXT,
    contact_phone  TEXT,
    is_active      BOOLEAN DEFAULT true,
    created_at     TIMESTAMPTZ DEFAULT now()
);
```

#### `practitioner_profiles`
```sql
CREATE TABLE practitioner_profiles (
    id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id                UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    hub_id                 UUID REFERENCES hubs(id),
    speciality             TEXT NOT NULL,
    coverage_radius_km     FLOAT DEFAULT 10.0,
    languages              TEXT[] DEFAULT '{}',
    cert_level             TEXT DEFAULT 'NONE' CHECK (cert_level IN ('NONE','BASE','ADVANCED')),
    ping_status            TEXT DEFAULT 'INACTIVE' CHECK (ping_status IN ('ACTIVE','INACTIVE','BUSY')),
    -- Localisation chiffrée : stockée comme bytea, déchiffrée uniquement serveur-side pour calcul de distance
    location_encrypted     BYTEA,
    ping_status_updated_at TIMESTAMPTZ DEFAULT now(),
    created_at             TIMESTAMPTZ DEFAULT now()
);
```

#### `training_modules`
```sql
CREATE TABLE training_modules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title_fr            TEXT NOT NULL,
    title_ar            TEXT,
    category            TEXT NOT NULL,
    content_offline_ref TEXT,   -- chemin vers le bundle offline
    passing_score       INTEGER DEFAULT 70,
    is_offline_available BOOLEAN DEFAULT true,
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ DEFAULT now()
);
```

#### `module_completions`
```sql
CREATE TABLE module_completions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practitioner_id  UUID REFERENCES practitioner_profiles(id),
    module_id        UUID REFERENCES training_modules(id),
    score            INTEGER NOT NULL,
    passed           BOOLEAN GENERATED ALWAYS AS (score >= (
                         SELECT passing_score FROM training_modules WHERE id = module_id
                     )) STORED,
    attempt_count    INTEGER DEFAULT 1,
    completed_at     TIMESTAMPTZ DEFAULT now(),
    UNIQUE(practitioner_id, module_id)
);
```

#### `certifications`
```sql
CREATE TABLE certifications (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practitioner_id  UUID REFERENCES practitioner_profiles(id),
    level            TEXT NOT NULL CHECK (level IN ('BASE','ADVANCED')),
    validated_by_hub UUID REFERENCES hubs(id),
    issued_at        TIMESTAMPTZ DEFAULT now(),
    is_valid         BOOLEAN DEFAULT true
);
```

#### `ping_events`
```sql
CREATE TABLE ping_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hub_id              UUID REFERENCES hubs(id),
    triggered_by_user   UUID REFERENCES users(id),
    source              TEXT CHECK (source IN ('HUB_SUPERVISOR','PRACTITIONER_SELF','COMMUNITY')),
    alert_type          TEXT NOT NULL,
    -- Zone obfusquée : centroïd du douar/zone, rayon ~2km, pas de coordonnée précise
    obfuscated_zone     GEOMETRY(POINT, 4326),
    status              TEXT DEFAULT 'DISPATCHED',
    triggered_at        TIMESTAMPTZ DEFAULT now(),
    resolved_at         TIMESTAMPTZ,
    resolution_note     TEXT
);
```

#### `ping_responses`
```sql
CREATE TABLE ping_responses (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ping_event_id    UUID REFERENCES ping_events(id),
    practitioner_id  UUID REFERENCES practitioner_profiles(id),
    response         TEXT CHECK (response IN ('ATTENDING','UNAVAILABLE','ESCALATING')),
    eta_minutes      INTEGER,
    notified_at      TIMESTAMPTZ DEFAULT now(),
    responded_at     TIMESTAMPTZ
);
```

#### `referrals`
```sql
CREATE TABLE referrals (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hub_id                  UUID REFERENCES hubs(id),
    institution_id          UUID REFERENCES institutions(id),
    practitioner_id         UUID REFERENCES practitioner_profiles(id),
    anonymised_patient_ref  TEXT,   -- hash opaque, pas de nom
    sex_category            TEXT,
    age_group               TEXT,
    symptoms_general        TEXT,
    approximate_location    GEOMETRY(POINT, 4326),
    status                  TEXT DEFAULT 'PENDING',
    practitioner_note       TEXT,
    ping_event_id           UUID REFERENCES ping_events(id),  -- si issu d'un ping
    created_at              TIMESTAMPTZ DEFAULT now(),
    updated_at              TIMESTAMPTZ DEFAULT now()
);
```

#### `audit_logs`
```sql
CREATE TABLE audit_logs (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id     UUID REFERENCES users(id),
    action       TEXT NOT NULL,
    entity_type  TEXT,
    entity_id    UUID,
    metadata     JSONB,
    ip_hash      TEXT,
    timestamp    TIMESTAMPTZ DEFAULT now()
);
-- Index pour les requêtes d'audit
CREATE INDEX idx_audit_actor ON audit_logs(actor_id);
CREATE INDEX idx_audit_timestamp ON audit_logs(timestamp DESC);
```

---

## 5. Backend — API REST

### Endpoints principaux

#### Auth — `/api/auth`

| Méthode | Route | Rôle requis | Description |
|---------|-------|-------------|-------------|
| POST | `/register` | — | Crée un compte (inactif jusqu'à validation Hub) |
| POST | `/login` | — | Retourne access token + refresh token |
| POST | `/refresh` | — | Renouvelle l'access token |
| POST | `/logout` | Authentifié | Invalide le refresh token |

#### Praticiens — `/api/practitioners`

| Méthode | Route | Rôle requis | Description |
|---------|-------|-------------|-------------|
| GET | `/me` | PRACTITIONER | Profil propre |
| PATCH | `/me` | PRACTITIONER | Mise à jour profil |
| PATCH | `/me/ping-status` | PRACTITIONER | Changer statut Ping |
| GET | `/` | HUB_SUPERVISOR | Liste des praticiens du Hub |
| PATCH | `/:id/validate` | HUB_SUPERVISOR | Valider un praticien |

#### Formation — `/api/training`

| Méthode | Route | Rôle requis | Description |
|---------|-------|-------------|-------------|
| GET | `/modules` | PRACTITIONER | Liste des modules disponibles |
| GET | `/modules/:id` | PRACTITIONER | Contenu d'un module |
| POST | `/modules/:id/complete` | PRACTITIONER | Soumettre un quiz |
| GET | `/my-progress` | PRACTITIONER | Progression et certifications |

#### Ping — `/api/ping`

| Méthode | Route | Rôle requis | Description |
|---------|-------|-------------|-------------|
| POST | `/trigger` | HUB_SUPERVISOR, PRACTITIONER | Déclencher une alerte |
| GET | `/active` | HUB_SUPERVISOR | Pings actifs du Hub |
| POST | `/:id/respond` | PRACTITIONER | Répondre à un Ping |
| PATCH | `/:id/close` | HUB_SUPERVISOR | Clôturer un Ping |
| GET | `/:id/responses` | HUB_SUPERVISOR | Réponses à un Ping |

#### Signalements — `/api/referrals`

| Méthode | Route | Rôle requis | Description |
|---------|-------|-------------|-------------|
| POST | `/` | PRACTITIONER, HUB_SUPERVISOR | Créer un signalement |
| GET | `/` | HUB_SUPERVISOR, INSTITUTION | Liste des signalements |
| PATCH | `/:id/status` | HUB_SUPERVISOR, INSTITUTION | Mettre à jour le statut |

#### Hub — `/api/hub`

| Méthode | Route | Rôle requis | Description |
|---------|-------|-------------|-------------|
| GET | `/dashboard` | HUB_SUPERVISOR | Données aggrégées du Hub |
| GET | `/practitioners` | HUB_SUPERVISOR | Praticiens de la zone |
| GET | `/analytics` | HUB_SUPERVISOR | Statistiques anonymisées |

### Middleware RBAC

```js
// middleware/rbac.js
const authorize = (...allowedRoles) => (req, res, next) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Accès non autorisé' });
    }
    next();
};

// Usage dans les routes
router.get('/practitioners',
    authenticateJWT,
    authorize('HUB_SUPERVISOR', 'ADMIN'),
    practitionerController.listByHub
);
```

### Middleware d'audit

```js
// middleware/audit-logger.js
const auditLog = (action) => async (req, res, next) => {
    res.on('finish', async () => {
        if (res.statusCode < 400) {
            await db.auditLogs.create({
                actorId:    req.user?.id,
                action,
                entityType: req.params.entityType,
                entityId:   req.params.id,
                ipHash:     hashIp(req.ip),
                metadata:   { method: req.method, path: req.path }
            });
        }
    });
    next();
};
```

---

## 6. Moteur Ping

Le moteur Ping est le composant le plus critique du MVP. Il s'exécute en tant que service dédié avec une file de jobs Redis (BullMQ).

### Flux de dispatch

```
triggerPing(hubId, alertType, obfuscatedZone)
    │
    ├─ 1. Récupérer tous les praticiens ACTIVE du Hub + zone élargie
    ├─ 2. Calculer distance haversine (côté serveur, localisation déchiffrée en mémoire)
    ├─ 3. Filtrer par spécialité si alertType nécessite une spécialité spécifique
    ├─ 4. Trier : distance → certLevel → responseHistoryScore
    ├─ 5. Sélectionner les N premiers (défaut N=5)
    ├─ 6. Envoyer notification push à chacun (Web Push + fallback SMS si injoignable)
    └─ 7. Démarrer timer d'escalade (défaut : 15 min sans réponse ATTENDING)
```

### Implémentation

```js
// modules/ping/ping.service.js

const PING_RADIUS_KM      = 50;   // rayon de recherche maximal
const DEFAULT_N           = 5;    // nombre de praticiens notifiés
const ESCALATION_DELAY_MS = 15 * 60 * 1000;

async function dispatchPing(pingEventId) {
    const event = await PingEvent.findById(pingEventId);
    const hub   = await Hub.findById(event.hubId);

    // 1. Récupérer praticiens actifs dans la zone
    const candidates = await db.query(`
        SELECT pp.id, pp.speciality, pp.cert_level,
               ST_Distance(
                   ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography,
                   ST_SetSRID(ST_MakePoint(
                       decrypt_location(pp.location_encrypted, $3)
                   ), 4326)::geography
               ) / 1000 AS distance_km
        FROM practitioner_profiles pp
        JOIN users u ON u.id = pp.user_id
        WHERE pp.ping_status = 'ACTIVE'
          AND pp.hub_id = $4
          AND u.is_active = true
          AND ST_DWithin(
              ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography,
              ST_SetSRID(ST_MakePoint(
                  decrypt_location(pp.location_encrypted, $3)
              ), 4326)::geography,
              $5 * 1000
          )
        ORDER BY distance_km ASC,
                 CASE pp.cert_level WHEN 'ADVANCED' THEN 0 ELSE 1 END
        LIMIT $6
    `, [event.lon, event.lat, process.env.LOCATION_ENC_KEY,
        event.hubId, PING_RADIUS_KM, DEFAULT_N]);

    // 2. Créer les PingResponse et notifier
    for (const practitioner of candidates) {
        await PingResponse.create({
            pingEventId: event.id,
            practitionerId: practitioner.id,
            notifiedAt: new Date()
        });
        await pushService.notify(practitioner.id, {
            title: 'Alerte Yaḥḍuru',
            body:  `Intervention demandée près de chez vous (${Math.round(practitioner.distance_km)} km)`,
            data:  { pingEventId: event.id, alertType: event.alertType }
        });
    }

    // 3. Programmer l'escalade automatique
    await pingEscalationQueue.add(
        'escalate',
        { pingEventId: event.id },
        { delay: ESCALATION_DELAY_MS }
    );
}

// Job d'escalade automatique
pingEscalationQueue.process('escalate', async (job) => {
    const event = await PingEvent.findById(job.data.pingEventId);
    const hasAttending = await PingResponse.exists({
        pingEventId: event.id,
        response: 'ATTENDING'
    });

    if (!hasAttending && event.status === 'DISPATCHED') {
        await PingEvent.update(event.id, { status: 'ESCALATED' });
        await referralService.createFromPing(event.id);
        await hubNotificationService.notifyEscalation(event.hubId, event.id);
    }
});
```

### Réponse du praticien

```js
// POST /api/ping/:id/respond
async function respondToPing(req, res) {
    const { response, etaMinutes } = req.body;
    const practitionerId = req.user.practitionerProfile.id;

    await PingResponse.update(
        { pingEventId: req.params.id, practitionerId },
        { response, etaMinutes, respondedAt: new Date() }
    );

    if (response === 'ATTENDING') {
        await PingEvent.update(req.params.id, { status: 'RESPONDING' });
        // Notifier les autres praticiens que l'alerte est prise en charge
        await cancelRemainingNotifications(req.params.id, practitionerId);
    }

    if (response === 'ESCALATING') {
        await PingEvent.update(req.params.id, { status: 'ESCALATED' });
        await referralService.createFromPing(req.params.id);
    }

    res.json({ success: true });
}
```

---

## 7. Frontend — PWA Praticien

### Configuration Workbox (offline-first)

```js
// vite.config.js
import { VitePWA } from 'vite-plugin-pwa';

export default {
    plugins: [
        VitePWA({
            strategies: 'injectManifest',
            srcDir: 'src/sw',
            filename: 'sw.js',
            manifest: {
                name: 'Yaḥḍuru',
                short_name: 'Yaḥḍuru',
                display: 'standalone',
                background_color: '#0D2B4E',
                theme_color: '#1A7A6E',
            }
        })
    ]
};
```

```js
// src/sw/sw.js
import { precacheAndRoute, cleanupOutdatedCaches } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, BackgroundSyncPlugin } from 'workbox-strategies';

precacheAndRoute(self.__WB_MANIFEST);
cleanupOutdatedCaches();

// Modules de formation : cache-first (contenu statique)
registerRoute(
    ({ url }) => url.pathname.startsWith('/api/training/modules'),
    new CacheFirst({ cacheName: 'training-content', plugins: [
        new ExpirationPlugin({ maxAgeSeconds: 60 * 60 * 24 * 30 })
    ]})
);

// Signalements hors ligne : sync différée
registerRoute(
    ({ url }) => url.pathname.startsWith('/api/referrals'),
    new NetworkFirst({ cacheName: 'referrals', plugins: [
        new BackgroundSyncPlugin('referrals-queue', {
            maxRetentionTime: 24 * 60  // 24 heures
        })
    ]}),
    'POST'
);
```

### Stockage local (IndexedDB)

```js
// src/offline/db.js
import { openDB } from 'idb';

export const localDB = openDB('yahduru-offline', 1, {
    upgrade(db) {
        db.createObjectStore('training_modules', { keyPath: 'id' });
        db.createObjectStore('pending_referrals', { keyPath: 'localId', autoIncrement: true });
        db.createObjectStore('ping_cache',        { keyPath: 'id' });
        db.createObjectStore('user_profile',      { keyPath: 'id' });
    }
});
```

### Commutateur Ping

```jsx
// src/components/PingToggle.jsx
import { useState } from 'react';
import { usePractitionerStore } from '../stores/practitioner';

export function PingToggle() {
    const { pingStatus, updatePingStatus } = usePractitionerStore();
    const [loading, setLoading] = useState(false);

    const toggle = async () => {
        setLoading(true);
        const next = pingStatus === 'ACTIVE' ? 'INACTIVE' : 'ACTIVE';
        await updatePingStatus(next);
        setLoading(false);
    };

    const colors = {
        ACTIVE:   'bg-green-500',
        INACTIVE: 'bg-gray-400',
        BUSY:     'bg-amber-500'
    };

    return (
        <button
            onClick={toggle}
            disabled={loading}
            className={`w-20 h-20 rounded-full text-white font-bold text-sm ${colors[pingStatus]}`}
            aria-label={`Statut Ping : ${pingStatus}`}
        >
            {loading ? '...' : pingStatus === 'ACTIVE' ? '● Actif' : '○ Inactif'}
        </button>
    );
}
```

---

## 8. Frontend — Interface Hub

Le dashboard Hub est une SPA React standard (non-offline) avec deux vues principales.

### Vue Ping en temps réel

```jsx
// src/pages/hub/PingDashboard.jsx
import { useEffect, useState } from 'react';
import { usePingStore } from '../../stores/ping';

export function PingDashboard() {
    const { activePings, triggerPing, closePing } = usePingStore();

    // Polling toutes les 10 secondes en MVP
    // Phase 2 : remplacer par WebSocket
    useEffect(() => {
        const interval = setInterval(() => usePingStore.getState().fetchActive(), 10000);
        return () => clearInterval(interval);
    }, []);

    return (
        <div>
            <h1>Pings actifs</h1>
            {activePings.map(ping => (
                <PingCard key={ping.id} ping={ping} onClose={closePing} />
            ))}
            <TriggerPingForm onSubmit={triggerPing} />
        </div>
    );
}
```

> **Note MVP** : le temps réel est simulé par polling (10s). Prévoir une migration vers WebSocket (Socket.io) en Phase 2 pour réduire la latence des alertes.

---

## 9. Notifications push et fallback SMS

### Web Push (VAPID)

```js
// services/push.service.js
import webpush from 'web-push';

webpush.setVapidDetails(
    'mailto:' + process.env.VAPID_EMAIL,
    process.env.VAPID_PUBLIC_KEY,
    process.env.VAPID_PRIVATE_KEY
);

export async function notify(practitionerId, payload) {
    const subscription = await PushSubscription.findByPractitioner(practitionerId);
    if (!subscription) return await smsService.send(practitionerId, payload.body);

    try {
        await webpush.sendNotification(subscription, JSON.stringify(payload));
    } catch (err) {
        if (err.statusCode === 410) {
            // Subscription expirée
            await PushSubscription.delete(subscription.id);
        }
        // Fallback SMS
        await smsService.send(practitionerId, payload.body);
    }
}
```

### Fallback SMS (Twilio)

```js
// services/sms.service.js
import twilio from 'twilio';

const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);

export async function send(practitionerId, message) {
    const phone = await getEncryptedPhone(practitionerId); // déchiffrement en mémoire
    await client.messages.create({
        body: `Yaḥḍuru : ${message}`,
        from: process.env.TWILIO_PHONE,
        to:   phone
    });
}
```

---

## 10. Sécurité et protection des données

### Principes non négociables

| Règle | Implémentation |
|-------|----------------|
| Pas de localisation précise en BDD | Coordonnées chiffrées AES-256-GCM, déchiffrement uniquement en mémoire serveur pour calcul PostGIS |
| Pas d'identité patient dans les signalements | `anonymised_patient_ref` = hash SHA-256 d'un identifiant local éphémère généré par le Hub |
| Séparation données personnelles / santé | Tables distinctes ; accès `referrals` et `ping_events` refusé aux rôles PRACTITIONER et INSTITUTION directement |
| Audit immutable | `audit_logs` : INSERT only, pas d'UPDATE ni DELETE autorisés même en ADMIN |
| HTTPS obligatoire | HSTS avec `max-age=31536000` sur tous les domaines |

### Chiffrement des localisations

```js
// utils/location-crypto.js
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGO = 'aes-256-gcm';
const KEY  = Buffer.from(process.env.LOCATION_ENC_KEY, 'hex'); // 32 bytes

export function encryptLocation(lat, lon) {
    const iv         = randomBytes(12);
    const cipher     = createCipheriv(ALGO, KEY, iv);
    const payload    = JSON.stringify({ lat, lon });
    const encrypted  = Buffer.concat([cipher.update(payload, 'utf8'), cipher.final()]);
    const authTag    = cipher.getAuthTag();
    return Buffer.concat([iv, authTag, encrypted]);
}

export function decryptLocation(buffer) {
    const iv        = buffer.subarray(0, 12);
    const authTag   = buffer.subarray(12, 28);
    const encrypted = buffer.subarray(28);
    const decipher  = createDecipheriv(ALGO, KEY, iv);
    decipher.setAuthTag(authTag);
    const decrypted = Buffer.concat([decipher.update(encrypted), decipher.final()]);
    return JSON.parse(decrypted.toString('utf8'));
}
```

---

## 11. Ordre d'implémentation recommandé

L'ordre ci-dessous permet d'avoir un prototype fonctionnel démontrable le plus tôt possible.

```
Semaine 1
├── [x] Setup BDD (PostgreSQL + PostGIS + migrations de base)
├── [x] Module Auth : inscription, login, JWT, refresh
└── [x] Middleware RBAC + audit logger

Semaine 2
├── [x] Module Praticiens : profil, validation Hub
├── [x] Module Formation : modules, quiz, completion
└── [x] Certification automatique après 3 modules validés

Semaine 3
├── [x] Moteur Ping : dispatch, réponse praticien, escalade timer
├── [x] Service Push + fallback SMS
└── [x] PWA Praticien : auth, profil, toggle Ping, formation (offline)

Semaine 4
├── [x] Interface Hub : dashboard Ping temps réel, gestion praticiens
├── [x] Module Référence (Module E)
└── [x] Dashboard institutionnel basique

Semaine 5-6
├── [ ] Tests d'intégration end-to-end
├── [ ] Mode offline complet (IndexedDB + background sync)
├── [ ] Déploiement VPS + CI/CD
└── [ ] Scénarios de démonstration hackathon
```

---

## 12. Variables d'environnement

```bash
# .env.example

# Application
NODE_ENV=development
PORT=3000
APP_URL=https://app.yahduru.ma

# Base de données
DATABASE_URL=postgresql://user:password@localhost:5432/yahduru_dev

# Redis (BullMQ)
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=<256-bit-random>
JWT_ACCESS_EXPIRY=1h
JWT_REFRESH_EXPIRY=7d

# Chiffrement localisation (AES-256 = 32 bytes = 64 hex chars)
LOCATION_ENC_KEY=<64-hex-chars>

# Web Push (VAPID)
VAPID_EMAIL=contact@yahduru.ma
VAPID_PUBLIC_KEY=<vapid-public-key>
VAPID_PRIVATE_KEY=<vapid-private-key>

# Twilio (SMS fallback)
TWILIO_ACCOUNT_SID=<sid>
TWILIO_AUTH_TOKEN=<token>
TWILIO_PHONE=+212XXXXXXXXX

# Ping
PING_DEFAULT_RADIUS_KM=50
PING_DEFAULT_N=5
PING_ESCALATION_DELAY_MIN=15
```

---

## 13. Tests

### Structure

```
backend/tests/
├── unit/
│   ├── ping.service.test.js      # logique de dispatch et sélection
│   ├── location-crypto.test.js   # chiffrement/déchiffrement
│   └── certification.test.js     # règles de certification progressive
├── integration/
│   ├── auth.test.js
│   ├── ping-flow.test.js         # flux complet : trigger → réponse → clôture
│   └── referral.test.js
└── fixtures/
    ├── seed-hub.js
    └── seed-practitioners.js
```

### Scénario de test critique : flux Ping complet

```js
// tests/integration/ping-flow.test.js
describe('Flux Ping complet', () => {
    it('devrait dispatcher, recevoir une réponse ATTENDING et clôturer', async () => {
        // 1. Créer un Hub, des praticiens ACTIVE et certifiés
        const hub          = await createTestHub();
        const practitioners = await createTestPractitioners(hub.id, 5, 'ACTIVE');

        // 2. Déclencher un Ping
        const res = await api.post('/api/ping/trigger', {
            alertType: 'NEAR_URGENCY',
            obfuscatedZone: { lat: 31.5, lon: -7.8 }
        }, hubSupervisorToken);
        expect(res.status).toBe(201);
        const pingId = res.body.pingEventId;

        // 3. Premier praticien répond ATTENDING
        await api.post(`/api/ping/${pingId}/respond`, {
            response: 'ATTENDING', etaMinutes: 12
        }, practitionerToken(practitioners[0].id));

        // 4. Vérifier le statut
        const ping = await PingEvent.findById(pingId);
        expect(ping.status).toBe('RESPONDING');

        // 5. Hub clôture
        await api.patch(`/api/ping/${pingId}/close`, {
            resolutionNote: 'Intervention réalisée'
        }, hubSupervisorToken);
        const closed = await PingEvent.findById(pingId);
        expect(closed.status).toBe('RESOLVED');
    });

    it('devrait escalader automatiquement après 15 minutes sans réponse', async () => {
        // ... test avec timer accéléré via jest.useFakeTimers()
    });
});
```

---

## 14. Déploiement

### GitHub Actions (CI/CD)

```yaml
# .github/workflows/deploy.yml
name: Deploy to production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:15-3.3
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: yahduru_test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '18' }
      - run: cd backend && npm ci && npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /srv/yahduru
            git pull origin main
            cd backend && npm ci --production
            pm2 restart yahduru-api
            cd ../frontend/pwa && npm ci && npm run build
```

### Checklist avant démonstration

- [ ] Variables d'environnement de production configurées
- [ ] Certificat TLS actif (Let's Encrypt)
- [ ] HSTS activé
- [ ] Clé VAPID générée et configurée
- [ ] Migrations de production appliquées
- [ ] Données de démonstration chargées (Hub + 5 praticiens + 3 modules)
- [ ] Scénario Ping testé de bout en bout sur mobile Android
- [ ] Mode offline vérifié (couper le réseau, remplir un signalement, reconnecter)

---

*Yaḥḍuru — يَحْضُر — Équipe 2b1b · Holberton School Thonon-les-Bains*
