# Architecture de Sqily

Sqily est une application web Ruby on Rails (v7.2) qui implémente la **Validation Mutuelle des Compétences (VMC)** — une méthode pédagogique où les apprenants se valident mutuellement leurs compétences, à la manière des arbres de connaissances.

Ce document s'adresse à un développeur Ruby qui découvre le projet. Il explique les concepts métier, les relations entre modèles, et les patterns techniques non-évidents.

---

## Sommaire

- [Concepts métier](#concepts-métier--le-vocabulaire-de-la-vmc)
- [Schéma relationnel](#schéma-relationnel)
- [Cycle de vie d'un apprenant](#cycle-de-vie-dun-apprenant)
- [Workflow de validation d'une compétence](#workflow-de-validation-dune-compétence)
- [Dossiers et responsabilités](#dossiers-et-responsabilités)
- [Patterns techniques notables](#patterns-techniques-notables)
- [Stack technique](#stack-technique)

---

## Concepts métier — le vocabulaire de la VMC

Avant de lire le code, il est essentiel de comprendre le vocabulaire pédagogique du projet. La plupart des incompréhensions viennent de noms de classes qui ont un sens précis dans la VMC.

| Terme technique | Ce que c'est concrètement |
|---|---|
| `Community` | Une classe, une école, un groupe d'apprenants. C'est le conteneur principal. |
| `Membership` | L'appartenance d'un `User` à une `Community`. Porte le rôle (modérateur), l'équipe, les badges et les notifications. |
| `Skill` | Une compétence à acquérir dans la communauté. Les compétences sont organisées en arbre (parent/enfants). Une `Skill` racine peut regrouper des sous-compétences. |
| `Prerequisite` | Un prérequis entre deux `Skill`. On ne peut pas commencer une compétence si ses prérequis ne sont pas validés. |
| `Subscription` | L'inscription d'un `User` à une `Skill`. C'est l'objet qui suit la progression (en cours / validée). Une `Subscription` est **complétée** (`completed_at`) quand la compétence est acquise. |
| `Evaluation` | Un "défi" (ou épreuve) créé par un expert pour une compétence. C'est la description de ce que le candidat doit démontrer. Il peut en exister plusieurs pour une même `Skill`. |
| `Evaluation::Exam` | Une session de validation entre un candidat et un examinateur. Naît quand un candidat accepte un défi. |
| `Evaluation::Note` | Un échange (message) entre le candidat et l'examinateur dans le cadre d'un `Exam`. L'exam se termine quand une `Note` est marquée `is_accepted: true`. |
| `Evaluation::Draft` | Le brouillon qu'un candidat prépare avant de lancer un `Exam`. |
| `Homework` | Une alternative à l'`Exam` : le candidat uploade un fichier, l'auteur de l'`Evaluation` l'approuve ou le rejette. |
| `Workspace` | Un document collaboratif (article, portfolio). Peut être publié et associé à une `Skill`. Versionné via `Workspace::Version`. |
| `Badge` | Une récompense attribuée automatiquement selon des règles de score (ex : avoir créé 2 évaluations = `Badge::Creator`). Il existe 12 types de badges. |
| `Message` | Un message dans le fil d'un forum. Peut être envoyé à une `Community`, une `Skill`, un `Workspace`, ou directement à un `User`. Vaste hiérarchie de sous-types (voir `app/models/message/`). |
| `Notification` | Une alerte in-app pour un `Membership`. Générée automatiquement par des callbacks sur les modèles. |
| `Team` | Un sous-groupe au sein d'une `Community`. Permet de regrouper des membres. |
| `Poll` | Un sondage posté dans un forum (communauté, compétence ou workspace). |
| `Event` | Un événement planifié dans une communauté. Les membres peuvent s'y inscrire (`Participation`). |

---

## Schéma relationnel

Voici les relations entre les modèles centraux. Les détails complets sont dans `db/schema.rb`.

```
User ──────────────── Membership ──────────── Community
 │                      │    │                    │
 │                   Team  Badge/Notification    Skill ──── Skill (enfants)
 │                                                │             │
 │                                           Prerequisite  Subscription ◄── User
 │                                                │             │
 │                                           Evaluation      Homework
 │                                                │          Evaluation::Exam
 │                                          Evaluation::Exam     │
 │                                               │          Evaluation::Note
 │                                          (examiner = User)
 │
 └── Workspace::Partnership ── Workspace ── Workspace::Version
                                   │
                               Message (to_workspace)

Message ──► to_community | to_skill | to_workspace | to_user (polymorphisme par colonnes)
Vote ──────► Message
HashTag ───► Message
```

**Règle importante sur `Message`** : la table `messages` utilise une colonne `type` (STI Rails) pour distinguer 15+ sous-types (`Message::Text`, `Message::Upload`, `Message::EventCreated`, etc.). Les colonnes `to_community_id`, `to_skill_id`, `to_workspace_id`, `to_user_id` déterminent le destinataire — une seule est renseignée à la fois.

---

## Cycle de vie d'un apprenant

```
1. Inscription
   User.signup(...)
   └─► User créé

2. Rejoindre une Community
   community.add_user(user)
   └─► Membership créé
   └─► Message::NewMembership posté dans le fil de la communauté

3. S'abonner à une Skill
   skill.subscribe(user)
   └─► Subscription créée (completed_at: nil)
   └─► Si la Skill a un parent → parent.subscribe(user) récursivement
   └─► Message::NewSubscription posté dans le fil de la compétence

4. Valider la compétence (deux chemins possibles, voir section suivante)

5. Subscription complétée
   subscription.complete(validator)
   └─► completed_at = Time.now, validator_id = validator.id
   └─► Message::SubscriptionComplete posté
   └─► Si c'est une sous-compétence → vérification de la complétion du parent
   └─► Badges éventuellement attribués
```

---

## Workflow de validation d'une compétence

Il existe **deux chemins** pour valider une `Subscription` :

### Chemin A — Homework (validation asynchrone par fichier)

```
Candidat                          Auteur de l'Evaluation
   │                                        │
   ├─► upload fichier (Homework créé)       │
   │   homework.file_node = "..."           │
   │                                        │
   │   [Notification::HomeworkPending] ────►│
   │                                        │
   │                          ┌─ homework.approve(user)
   │                          │  └─► subscription.complete(user)
   │                          │
   │   [Notification::HomeworkApproved] ◄──┘
   │         OU
   │   [Notification::HomeworkRejected] ◄── homework.reject
   │   └─► nouveau Homework créé automatiquement (reject_and_keep_open)
```

### Chemin B — Exam (validation synchrone par échange de messages)

```
Candidat                          Examinateur (un expert de la Skill)
   │                                        │
   ├─► Evaluation::Draft créé               │
   ├─► draft.submit → Evaluation::Exam créé │
   │   └─► examinateur sélectionné par      │
   │       Evaluation#pick_examiner_for     │
   │       (moins occupé, même équipe si    │
   │       possible, jamais le même deux    │
   │       fois de suite)                   │
   │                                        │
   ├─► Evaluation::Note soumise ───────────►│
   │                                        ├─► Note soumise en retour
   │◄───────────────────────────────────────┤   (is_accepted: false)
   │                    ...échanges...      │
   │                                        │
   │◄──── Note finale (is_accepted: true) ──┤
   │      └─► subscription.complete(examiner)
```

**Invariant important** : un candidat ne peut avoir qu'un seul `Exam` en cours (`scope :ongoing`) pour une même compétence. Il doit annuler l'actuel avant d'en démarrer un nouveau.

---

## Dossiers et responsabilités

```
app/
├── controllers/          # Contrôleurs Rails classiques
│   ├── admin/            # Interface d'administration (gestion des Community)
│   ├── evaluations/      # Sous-contrôleurs pour Draft, Exam, Note
│   ├── profile/          # Profil public d'un Membership
│   ├── public/           # Pages accessibles sans connexion
│   └── workspaces/       # Sous-contrôleur pour les Workspace::Partnership
│
├── models/
│   ├── concerns/         # Modules partagés (stockage AWS, hashtags, tri sécurisé)
│   ├── badge/            # 12 sous-classes de Badge, chacune avec sa logique de score
│   ├── evaluation/       # Exam, Note, Draft — le cœur de la VMC
│   ├── message/          # 15+ sous-types de Message (voir README dédié)
│   ├── notification/     # 9 types de Notification (voir README dédié)
│   └── workspace/        # Lock, Version, Partnership
│
├── lib/                  # ⚠️ Dossier non-standard : objets de service maison
│   │                     # (voir README dédié)
│   ├── community/        # Community::Statistics
│   ├── membership/       # Membership::Permissions
│   └── user/             # User::Permissions, User::Statistics
│
├── jobs/
│   ├── community/        # DuplicationJob, DeleteJob, SendStatisticsJob
│   └── skill/            # DuplicateJob, DeleteJob
│
├── mailers/              # Emails transactionnels (UserMailer, ExamMailer, ExportMailer)
└── assets/
    └── javascripts/sqily/ # JS organisé par domaine (pas de framework front, Stimulus-like vanilla)
```

### Le dossier `app/lib/` — à ne pas confondre avec `lib/`

Le dossier `app/lib/` (et non `lib/` à la racine) contient des objets Ruby qui n'entrent pas dans les catégories Rails classiques. Il s'agit principalement de :

- **Form objects** : `SkillForm`, `WorkspaceForm`, `CommunityRequestForm` — valident et encapsulent la logique de création/édition complexe.
- **Service objects** : `Community::Statistics`, `User::Statistics` — calculent des données analytiques sans polluer les modèles.
- **Permission objects** : `User::Permissions`, `Membership::Permissions` — centralisent les règles d'autorisation (qui peut faire quoi).
- **Utilitaires** : `HtmlCompare` (diff HTML), `OrderParam` (tri sécurisé), `Serializer`.

---

## Patterns techniques notables

### 1. STI (Single Table Inheritance) sur `Message` et `Notification`

`Message` et `Notification` utilisent la colonne `type` de Rails pour stocker plusieurs sous-classes dans une seule table. Chaque sous-classe a sa propre logique de déclenchement via des callbacks `after_create` / `after_save` définis **dans les sous-classes elles-mêmes** :

```ruby
# Dans Message::EventCreated
Event.after_create { Message::EventCreated.trigger(self) }
```

Ce pattern permet d'ajouter un nouveau type de message sans modifier la table ni la classe `Message` de base.

### 2. Badges par STI avec `replay`

Chaque badge définit son propre `required_count` et `compute_score`. La méthode `replay` permet de recalculer tous les badges depuis zéro (utile après migration ou correctif) :

```ruby
Badge.replay  # recalcule tous les 12 types
```

### 3. Complétion récursive des Subscriptions

Quand une `Skill` a des enfants, la complétion de la compétence parente est automatiquement recalculée à chaque changement d'une sous-compétence. La règle : toutes les sous-compétences **mandatory** publiées doivent être complétées. La logique est dans `Subscription#refresh_completed_at`.

### 4. Sélection de l'examinateur (`pick_examiner_for`)

L'algorithme dans `Evaluation#pick_examiner_for` cherche l'expert le moins occupé, en privilégiant d'abord la même équipe que le candidat, et en évitant l'examinateur des tentatives précédentes. Il y a une tolérance de dernier recours (`.sample` parmi les 10 moins occupés) si aucun candidat idéal n'est disponible.

### 5. `TypeScopes` gem

Le gem `type_scopes` injecte des scopes dynamiques basés sur le nom des attributs. Par exemple, sur `User`, il génère `name_contains`, `name_starts_with`, etc. Voir la gem pour la liste complète des scopes générés.

### 6. Routage par `permalink`

Les URLs des communautés utilisent un `permalink` (slug) plutôt qu'un `id` numérique. La méthode `Community#to_param` retourne `permalink`, ce qui est transparent dans les helpers Rails mais important à savoir pour débuguer les routes.

---

## Stack technique

| Composant | Technologie |
|---|---|
| Framework | Ruby on Rails 7.2 |
| Base de données | PostgreSQL (avec recherche full-text native via `to_tsvector`) |
| Authentification | `has_secure_password` (bcrypt, pas de Devise) |
| Stockage fichiers | AWS S3 (via `aws-sdk-s3`) |
| Background jobs | ActiveJob + cron via `whenever` (pas de Sidekiq) |
| Serveur | Puma |
| Pagination | Kaminari |
| Images | RMagick (redimensionnement des avatars) |
| Monitoring | RorVsWild |
| Front-end | Vanilla JS + Trix (éditeur riche) + Flatpickr + Awesomplete |
| Internationalisation | Rails i18n (fr-CH par défaut, + de-CH, it-CH, en) |
| Tests | Minitest + Mocha |
| Déploiement | Docker Compose ; chaque push sur `master` → production automatique |
