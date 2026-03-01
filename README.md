<div align="center">

# يَحْضُر &nbsp;·&nbsp; Yaḥḍuru

### *Celui qui est présent*

**Réseau décentralisé de soins de proximité TCIM pour les populations négligées**

<br/>

[![Hackathon](https://img.shields.io/badge/Hackathon-Rabhacks%20Morocco-1A7A6E?style=for-the-badge)](https://rabhacks.ma)
[![Team](https://img.shields.io/badge/Équipe-2b1b-0D2B4E?style=for-the-badge)](#équipe)
[![School](https://img.shields.io/badge/Holberton%20School-Thonon--les--Bains-2E75B6?style=for-the-badge)](https://www.holbertonschool.fr/)
[![License](https://img.shields.io/badge/Licence-MIT-C45911?style=for-the-badge)](LICENSE)

<br/>

```
  Praticien TCIM  ──── Ping ────▶  Hub Association  ────▶  Système formel
       │                               │                         │
  Disponible 24h         Valide, forme, coordonne        Reçoit, supervise
  Formé, certifié        Anonymise les données           Diffuse protocoles
  Visible, connecté      Dispatch les urgences
```

<br/>

> *Dans les zones rurales du Maroc et d'Afrique, le praticien traditionnel est souvent*
> *la seule présence de santé disponible. Yaḥḍuru le rend visible, formé et connecté.*

</div>

---

## Table des matières

- [Le problème](#-le-problème)
- [La solution](#-la-solution)
- [Fonctionnalité Ping](#-fonctionnalité-ping--le-cœur-du-projet)
- [Architecture](#-architecture)
- [Stack technique](#-stack-technique)
- [Démarrage rapide](#-démarrage-rapide)
- [Structure du projet](#-structure-du-projet)
- [Roadmap](#-roadmap)
- [Références](#-références)
- [Équipe](#-équipe)

---

## 🏥 Le problème

Les systèmes de santé formels sont **centripètes** : ils concentrent leurs ressources dans les centres urbains et exigent que les patients viennent à eux. Pour des millions de personnes vivant dans des zones rurales isolées, cette logique est une barrière systématique.

| Barrière | Réalité terrain |
|----------|----------------|
| **Géographique** | Douars à 40–120 km du centre de santé le plus proche |
| **Économique** | Coût du déplacement = 2–3 jours de revenu |
| **Linguistique** | Tamazight, tachelhit, hassaniya comme langues premières |
| **Temporelle** | Centres fermés la nuit, le week-end, en saison des pluies |

Ces populations ne sont pas sans soins pour autant. **Le praticien TCIM traditionnel est déjà là** — l'herboriste, la sage-femme traditionnelle, le guérisseur communautaire. Il est présent 24h/24, gratuit, et bénéficie d'une confiance communautaire que les institutions mettent des décennies à construire.

**Le problème : ce réseau est invisible, non formé et non connecté.**

Cinq lacunes structurelles l'empêchent d'atteindre son plein potentiel :

```
❌  Pas de registre     — les praticiens n'existent dans aucun système
❌  Pas de formation    — les signes de gravité ne sont pas systématiquement reconnus
❌  Pas de canal        — aucun moyen de signaler un cas grave au système formel
❌  Pas de signal       — impossible de savoir quel praticien est disponible en urgence
❌  Pas de données      — les autorités sanitaires ne peuvent pas planifier l'invisible
```

---

## 💡 La solution

Yaḥḍuru est une plateforme **mobile-first, offline-first** qui adresse ces cinq lacunes simultanément, organisée en trois niveaux :

```
┌─────────────────────────────────────────────────────────────────┐
│  NIVEAU 1 — TERRAIN                                             │
│  Praticien TCIM  ←→  PWA mobile  ←→  Statut Ping              │
└─────────────────────────────────────────────────────────────────┘
                           ↕ Formation · Signalement · Ping
┌─────────────────────────────────────────────────────────────────┐
│  NIVEAU 2 — HUB LOCAL                                           │
│  Association / Coopérative / ONG                                │
│  • Évaluation & certification TCIM (face-à-face)               │
│  • Dispatch Ping & coordination des urgences                    │
│  • Anonymisation des données avant transmission                 │
└─────────────────────────────────────────────────────────────────┘
                           ↕ Données agrégées & anonymisées
┌─────────────────────────────────────────────────────────────────┐
│  NIVEAU 3 — SYSTÈME FORMEL                                      │
│  Centres de santé · Hôpitaux · Autorités sanitaires            │
└─────────────────────────────────────────────────────────────────┘
```

L'**Association-hub** est la clé de voûte. Elle valide les praticiens en face-à-face selon les critères OMS, coordonne les urgences via le Ping, et garantit que seules des données anonymisées remontent vers le système formel.

---

## 🔔 Fonctionnalité Ping — le cœur du projet

Le Ping transforme Yaḥḍuru d'un outil de gestion en un **réseau actif de proximité médicale**. Inspiré du mécanisme d'alerte de l'application [The Sorority](https://www.thesorority.fr/), il permet à une personne en détresse d'atteindre le praticien certifié le plus proche en quelques minutes.

### Comment ça fonctionne

```
1.  Praticien active son statut  ●  ACTIF
                                  │
2.  Personne en détresse          │
    contacte le Hub      ─────────┘
                                  │
3.  Hub déclenche le Ping         │
                                  ▼
4.  Algorithme de sélection
    └─ Statut ACTIF (filtre obligatoire)
    └─ Distance géographique (haversine, côté serveur)
    └─ Spécialité pertinente (ex: sage-femme pour urgence obstétricale)
    └─ Niveau de certification
    └─ Historique de réponse
                                  │
5.  N praticiens notifiés         │
    (Web Push + fallback SMS)     │
                                  ▼
6.  Praticien répond
    ├─  ✅  J'interviens    →  ETA communiqué au Hub
    ├─  ❌  Je ne peux pas  →  Système passe au suivant
    └─  🔺  J'escalade      →  Notification prioritaire Hub + centre de santé
                                  │
7.  Si pas de réponse en 15 min   │
    → Escalade automatique ───────┘
```

### Statuts de disponibilité

| Statut | Couleur | Signification |
|--------|---------|---------------|
| `ACTIVE` | 🟢 | Disponible pour intervenir dans ma zone |
| `BUSY` | 🟡 | En intervention, contactable uniquement pour escalades critiques |
| `INACTIVE` | ⚪ | Non disponible |

> **Règle de sécurité** : le statut `ACTIVE` n'est visible que du Hub — jamais du grand public.
> Le Ping n'est pas une consultation médicale. C'est un mécanisme d'activation de proximité.

---

## 🏗 Architecture

### Modèle de données (simplifié)

```
users ──────────── practitioner_profiles ──── ping_status
  │                        │
  │                        ├── module_completions ── training_modules
  │                        │
  │                        └── certifications
  │
hubs ──────────── hub_supervisors
  │                        │
  │                 ping_events ──── ping_responses
  │                        │
  │                 referrals ─────── institutions
  │
audit_logs          interventions_aggregate
```

### Principes de sécurité des données

```
Niveau Hub          Niveau API central        Niveau Institution
──────────────────  ───────────────────────   ──────────────────
Données identité    Données pseudonymisées     Données agrégées
(nom, localisation) (IDs opaques, zones)       (statistiques anon.)
→ reste au Hub      → transit chiffré          → aucune identité
```

- 🔐 Coordonnées GPS chiffrées **AES-256-GCM** — déchiffrées uniquement en mémoire serveur pour le calcul de distance
- 🚫 Aucune donnée identifiante patient dans les signalements (`referrals`)
- 📋 Piste d'audit **immutable** sur toutes les actions sensibles
- ✅ Conformité visée : Loi **09-08** marocaine · adaptable RGPD · cadres UA

---

## 🛠 Stack technique

| Couche | Technologie | Justification |
|--------|-------------|---------------|
| PWA Praticien | **React + Vite + Workbox** | Offline-first, Service Worker, sync différée |
| Dashboard Hub | **React** | Composants partagés avec la PWA |
| API | **Node.js + Express** | Prototypage rapide, bonne gestion async |
| Base de données | **PostgreSQL 15 + PostGIS** | Requêtes spatiales natives pour le Ping |
| File de jobs | **Redis + BullMQ** | Dispatch Ping et timers d'escalade |
| Auth | **JWT + bcrypt** | Access 1h / Refresh 7j |
| Push | **Web Push API (VAPID)** | Standard multiplateforme |
| SMS fallback | **Twilio** | Couverture zones sans data mobile |
| CI/CD | **GitHub Actions** | Déploiement automatisé |

---

## 🚀 Démarrage rapide

### Prérequis

```
node >= 18.x
postgresql >= 15 avec extension PostGIS
redis >= 7
```

### Installation

```bash
git clone https://github.com/Schpser/2b1b.git
cd 2b1b
# Backend
cd backend && npm install
cp .env.example .env   # renseigner les variables

# PWA Praticien
cd ../frontend/pwa && npm install

# Dashboard Hub
cd ../frontend/dashboard && npm install
```

### Base de données

```bash
createdb yahduru_dev
psql yahduru_dev -c "CREATE EXTENSION IF NOT EXISTS postgis;"
psql yahduru_dev -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"

cd backend
npm run db:migrate
npm run db:seed:dev    # données de démonstration
```

### Lancement

```bash
# Terminal 1 — API
cd backend && npm run dev

# Terminal 2 — PWA Praticien
cd frontend/pwa && npm run dev

# Terminal 3 — Dashboard Hub
cd frontend/dashboard && npm run dev
```

L'API est accessible sur `http://localhost:3000`, la PWA sur `http://localhost:5173`, le dashboard sur `http://localhost:5174`.

### Variables d'environnement essentielles

```bash
DATABASE_URL=postgresql://user:password@localhost:5432/yahduru_dev
REDIS_URL=redis://localhost:6379
JWT_SECRET=<256-bit-random>
LOCATION_ENC_KEY=<64-hex-chars>     # AES-256 pour les coordonnées GPS
VAPID_PUBLIC_KEY=<clé-vapid>
VAPID_PRIVATE_KEY=<clé-vapid>
TWILIO_ACCOUNT_SID=<sid>            # SMS fallback (optionnel en dev)
```

> Voir [`.env.example`](.env.example) pour la liste complète.

---

## 📁 Structure du projet

```
yahduru/
├── backend/
│   ├── src/
│   │   ├── middleware/          # auth JWT, RBAC, audit logger
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── practitioners/
│   │   │   ├── hubs/
│   │   │   ├── training/
│   │   │   ├── ping/            # ← moteur Ping (dispatch + escalade)
│   │   │   ├── referrals/
│   │   │   └── institutions/
│   │   └── services/
│   │       ├── push.service.js  # Web Push VAPID
│   │       ├── sms.service.js   # Twilio fallback
│   │       └── geo.service.js   # calculs haversine
│   ├── migrations/
│   └── tests/
├── frontend/
│   ├── pwa/                     # PWA mobile praticien (offline-first)
│   └── dashboard/               # Interface Hub & Institution
├── docs/
│   ├── yahduru_classes.mmd      # Diagramme de classes
│   ├── yahduru_implementation.md
│   └── yahduru_documentation_fr.docx
└── .github/workflows/
    └── deploy.yml
```

---

## 🗺 Roadmap

### ✅ Phase 1 — MVP Hackathon *(en cours)*
- [x] Authentification RBAC (4 rôles)
- [x] Profils praticiens TCIM
- [x] Modules de formation + certification progressive
- [ ] **Moteur Ping** (dispatch · réponse · escalade automatique)
- [ ] Interface Hub (dashboard temps réel)
- [ ] Signalement & référence médicale anonymisée
- [ ] Dashboard institutionnel basique

### 🔄 Phase 2 — Pilote terrain *(3–6 mois)*
- [ ] Mode offline avancé (sync différée complète)
- [ ] Module d'évaluation TCIM assisté numériquement
- [ ] Contenus de formation en arabe, tamazight et iconographique
- [ ] Application native Android (meilleure expérience Ping offline)
- [ ] Déclenchement Ping communautaire (page publique)
- [ ] WebSocket pour le temps réel (remplace le polling)

### 🌍 Phase 3 — Déploiement *(12+ mois)*
- [ ] Analytique nationale et détection de tendances épidémiologiques
- [ ] Interopérabilité avec les systèmes d'information hospitaliers
- [ ] Extension aux régions africaines avec partenaires ONG
- [ ] Préservation et documentation des savoirs TCIM

---

## 📚 Références

Ce projet s'appuie sur les cadres normatifs et les initiatives suivantes, **sans en revendiquer l'affiliation** :

- **[WHO Global Traditional Medicine Strategy 2025–2034](https://www.who.int/teams/who-global-traditional-medicine-centre)** — stratégie OMS pour l'intégration de la MTCI dans les systèmes de santé universels
- **[TCIH Coalition](https://www.tcih.org)** — mouvement mondial pour la reconnaissance de la médecine traditionnelle, complémentaire et intégrative
- **[The Sorority](https://www.thesorority.fr/)** — application d'entraide communautaire dont le mécanisme d'alerte de proximité a inspiré la fonctionnalité Ping

---

## 👥 Équipe

<div align="center">

| | |
|:---:|:---:|
| **Mèlissa Sbibih** | **Carlos Silva** |
| Holberton School Thonon-les-Bains | Holberton School Thonon-les-Bains |
| [@Schpser](https://github.com/Schpser) | [@CodexSC](https://github.com/CodexSC) |

*Équipe 2b1b · Compétition depuis Thonon-les-Bains, France · Hackathon Rabhacks, Maroc*

</div>

---

<div align="center">

**يَحْضُر**

*Celui qui est là.*

<br/>

[![Made with ♥ in Thonon-les-Bains](https://img.shields.io/badge/Made%20with%20♥%20in-Thonon--les--Bains-2E75B6?style=flat-square)](https://www.holbertonschool.fr/)

</div>
