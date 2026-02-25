# Comprendre Sqily — Guide pour non-Rubyistes

Ce document vous donne une **image mentale complète** du fonctionnement de Sqily. Vous n'avez pas besoin de savoir lire du Ruby pour le comprendre. Son objectif est de vous permettre de formuler des demandes de modification ou de nouvelles fonctionnalités en sachant comment le système fonctionne, quelles structures de données existent, et quels effets domino provoqueront vos changements.

La documentation technique détaillée est dans `ARCHITECTURE.md` et les README de chaque dossier. Ce guide-ci ne répète pas ces détails : il vous construit la machine notionnelle.

---

## 1. L'idée en une phrase

Sqily est une plateforme où des apprenants regroupés en communautés se **valident mutuellement** leurs compétences, selon le principe : « celui qui a déjà appris quelque chose peut vérifier que quelqu'un d'autre l'a appris aussi ».

---

## 2. Les cinq structures fondamentales

Imaginez Sqily comme un emboîtement de cinq couches. Chaque couche contient la suivante.

```
┌─────────────────────────────────────────────────────────────────┐
│  COMMUNITY (la classe, l'école, le groupe)                      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SKILL (une compétence, organisée en arbre)              │   │
│  │                                                          │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  SUBSCRIPTION (un apprenant travaille sur        │    │   │
│  │  │  cette compétence)                               │    │   │
│  │  │                                                  │    │   │
│  │  │  ┌──────────────────────────────────────────┐    │    │   │
│  │  │  │  EVALUATION (le défi à réussir)          │    │    │   │
│  │  │  │                                          │    │    │   │
│  │  │  │  ┌──────────────────────────────────┐    │    │    │   │
│  │  │  │  │  EXAM (la session de validation) │    │    │    │   │
│  │  │  │  └──────────────────────────────────┘    │    │    │   │
│  │  │  └──────────────────────────────────────────┘    │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

Voici ce que chacune représente :

**Community** — C'est le conteneur principal. Pensez-y comme à une salle de classe virtuelle. Elle a un nom, un lien unique (son `permalink` qui apparaît dans l'URL), des membres, et des compétences. Tout se passe à l'intérieur d'une communauté.

**Skill** — Une compétence à acquérir. Les compétences forment un **arbre** : une compétence peut avoir des sous-compétences (enfants). Elles peuvent aussi avoir des **prérequis** entre elles (on ne peut pas commencer B sans avoir validé A). Chaque compétence a un fil de discussion, des évaluations, et un contenu descriptif.

**Subscription** — Le lien entre un utilisateur et une compétence. C'est l'objet-clé de toute la progression pédagogique. Quand vous vous abonnez à une compétence, une Subscription naît. Elle sera soit « en cours » (`completed_at` est vide), soit « validée » (`completed_at` contient une date et `validator_id` identifie qui a validé).

**Evaluation** — Le template d'une épreuve. Un expert crée une évaluation pour une compétence : « Voici ce que tu dois me démontrer ». Il peut en exister plusieurs pour la même compétence.

**Exam** — Une session de validation réelle entre un candidat et un examinateur. C'est un échange de messages (les « Notes ») qui se termine quand l'examinateur dit « validé ».

---

## 3. Les trois rôles

Il n'y a pas de système de rôles au sens classique (pas de table « roles »). Le rôle d'un utilisateur est **déduit de l'état de ses données** :

**Apprenant** — Tout utilisateur qui a une Subscription non complétée sur une compétence. C'est l'état par défaut.

**Expert** — Tout utilisateur qui a une Subscription **complétée** sur une compétence. Il peut alors valider d'autres apprenants sur cette même compétence. Le système choisit automatiquement l'expert le moins occupé pour chaque validation.

**Modérateur** — Un flag sur le lien d'appartenance à la communauté (la Membership). Le modérateur peut gérer les compétences, promouvoir ou rétrograder les validations, et voir les statistiques.

L'idée centrale est : **apprendre une compétence fait de vous un expert qui peut valider les autres**. Le rôle change dynamiquement.

---

## 4. Les deux chemins de validation

Quand un apprenant veut prouver qu'il maîtrise une compétence, il a deux chemins possibles. Comprendre cette dualité est essentiel pour toucher à quoi que ce soit dans le système de progression.

### Chemin A — Le devoir (Homework)

C'est le chemin simple : l'apprenant dépose un fichier, l'auteur de l'évaluation l'examine et dit oui ou non.

```
Apprenant                              Auteur de l'évaluation
    │                                          │
    │  "Voici mon travail" [fichier uploadé]   │
    │ ───────────────────────────────────────► │
    │                                          │
    │           "Validé ✓"  OU  "Refusé ✗"     │
    │ ◄─────────────────────────────────────── │
```

Si c'est validé → la compétence est acquise, la Subscription est complétée.
Si c'est refusé → un nouveau devoir vide est automatiquement recréé pour une nouvelle tentative.

### Chemin B — L'examen interactif (Exam)

C'est le chemin riche : un dialogue entre le candidat et un examinateur (un expert de la compétence, choisi automatiquement par l'algorithme).

```
Apprenant                              Examinateur (choisi automatiquement)
    │                                          │
    │  "Voici ma réponse au défi"              │
    │ ───────────────────────────────────────► │
    │                                          │
    │  "Questions / remarques"                 │
    │ ◄─────────────────────────────────────── │
    │                                          │
    │  "Précisions supplémentaires"            │
    │ ───────────────────────────────────────► │
    │                                          │
    │            ... autant d'allers-retours   │
    │                que nécessaire ...        │
    │                                          │
    │  "Validé ✓"                              │
    │ ◄─────────────────────────────────────── │
```

L'algorithme de sélection de l'examinateur favorise un expert de la même équipe que le candidat, qui n'est pas déjà surchargé, et qui n'a pas déjà examiné ce candidat auparavant.

**Règle : un seul examen en cours à la fois par compétence.** Le candidat doit annuler l'examen actuel pour en démarrer un autre.

---

## 5. L'effet cascade : ce qui se passe quand une compétence est validée

La complétion d'une Subscription n'est pas un événement isolé. C'est un **domino** qui déclenche potentiellement une cascade :

```
Une sous-compétence est validée
    │
    ├─► Le système vérifie : toutes les sous-compétences obligatoires
    │   du parent sont-elles validées ?
    │       │
    │       ├─ Oui → La compétence parente est automatiquement validée
    │       │         └─► Même vérification au niveau supérieur (récursif)
    │       │
    │       └─ Non → Rien de plus
    │
    ├─► Un message "Compétence validée" apparaît dans le fil de la compétence
    │
    ├─► Le système vérifie les critères de badges
    │   (ex : "a validé 5 compétences" → Badge::Specialist)
    │       │
    │       └─ Si le seuil est atteint → Badge attribué → Notification générée
    │
    └─► Les emails de résumé quotidien/hebdomadaire incluront cet événement
```

---

## 6. Le système de messagerie — une seule table, quatre destinations

Sqily a un seul système de messages qui sert à tout : discussions de groupe, messages privés, et événements système. La distinction se fait par le **destinataire** :

```
Un message va toujours vers exactement UNE de ces quatre destinations :

    ┌──────────────┐
    │  Community   │  → Fil de discussion général de la communauté
    └──────────────┘
    ┌──────────────┐
    │  Skill       │  → Fil de discussion d'une compétence spécifique
    └──────────────┘
    ┌──────────────┐
    │  Workspace   │  → Fil de commentaires d'un document collaboratif
    └──────────────┘
    ┌──────────────┐
    │  User        │  → Message privé entre deux personnes
    └──────────────┘
```

Il y a deux familles de messages :

**Messages humains** — Ce que l'utilisateur écrit directement (texte, fichier uploadé).

**Messages système** — Générés automatiquement quand quelque chose se passe. Quand quelqu'un rejoint la communauté, quand une compétence est validée, quand un sondage est créé, quand un document est publié… ces événements apparaissent dans les fils de discussion sous forme de messages système. Il en existe 16 types différents.

L'important pour vous : **si vous ajoutez une fonctionnalité qui produit un événement visible par la communauté, vous devrez probablement créer un nouveau type de message système**. Le mécanisme est standardisé — chaque type déclenche sa propre création automatiquement quand l'événement source se produit.

---

## 7. Le système de notifications — personnel et contextuel

Les notifications sont distinctes des messages. Un **message** est dans un fil de discussion visible par un groupe. Une **notification** est personnelle, dans le centre de notifications d'un utilisateur précis.

Particularité : une notification est liée à une **Membership** (le lien utilisateur-communauté), pas directement à l'utilisateur. Un même utilisateur dans deux communautés a deux flux de notifications séparés.

Il existe 9 types de notifications, chacun déclenché par un événement spécifique : badge reçu, devoir à corriger, devoir accepté/refusé, vote reçu sur un message, message épinglé, mention @nom, sondage terminé, message dans un workspace.

**Si vous ajoutez une fonctionnalité qui doit alerter un utilisateur spécifique, vous devrez probablement créer un nouveau type de notification.** Le mécanisme est identique à celui des messages système.

---

## 8. Les workspaces — le volet éditorial

Les workspaces sont des documents collaboratifs (articles, portfolios). Ils suivent un cycle de vie propre :

```
Création (brouillon privé)
    │
    ▼
Rédaction par les co-auteurs (versionné automatiquement)
    │
    ▼
Approbation par un lecteur invité
    │
    ▼
Publication (visible par toute la communauté ou lié à une compétence)
    │
    ▼
[Possibilité de dépublier / modifier / republier]
```

Les workspaces sont **versionnés** : chaque modification significative crée une nouvelle version. Le système détecte quand une nouvelle version est nécessaire (quand des commentaires ont été faits depuis la dernière version par des personnes autres que les auteurs).

---

## 9. Le système de badges — la gamification

12 types de badges récompensent les différentes formes de participation. Chaque badge a un **seuil** et un **score calculé**. Quand le score atteint le seuil, le badge est automatiquement attribué.

Exemples : `Creator` (a créé 2 évaluations), `Specialist` (a validé N compétences), `Messenger` (a posté N messages), etc.

Les badges sont recalculables à tout moment via un mécanisme de « replay » — utile après une correction de bug.

---

## 10. Les emails récurrents

Deux emails récapitulatifs tournent en tâche de fond :

**Résumé quotidien** — Envoyé à chaque utilisateur qui l'a activé. Contient : messages privés non lus, événements du lendemain, notifications non lues, devoirs à corriger.

**Résumé hebdomadaire** — Envoyé par communauté. Contient : nouveaux messages, nouvelles compétences, nouveaux membres.

Un rappel d'événement est aussi envoyé la veille de chaque événement aux participants inscrits.

---

## 11. Les permissions — qui peut faire quoi

Il n'y a pas de matrice de permissions globale. Les droits sont vérifiés par des objets dédiés (`User::Permissions` et `Membership::Permissions`) selon la logique suivante :

- **Valider quelqu'un** → être expert de cette compétence OU modérateur de la communauté.
- **Modifier un workspace** → être co-auteur (writer) du workspace.
- **Publier un workspace** → être propriétaire ET le workspace doit être approuvé.
- **Épingler un message** → être expert de la compétence OU modérateur.
- **Supprimer une compétence** → la compétence n'a pas d'enfants ET vous êtes modérateur.
- **Passer un examen** → ne pas avoir de Subscription complétée ET pas d'examen en cours.

La règle générale est : **les droits dépendent de l'état des données, pas d'un rôle assigné**.

---

## 12. Comment les URLs sont construites

Les URLs ne contiennent pas d'identifiants numériques pour les communautés. Elles utilisent un **permalink** (un slug lisible) :

```
/monecole                          → page publique de la communauté
/monecole/skills                   → liste des compétences
/monecole/skills/42                → une compétence
/monecole/skills/42/evaluations    → les défis de cette compétence
/monecole/exams/7                  → un examen
/monecole/discussion               → fil de discussion
/monecole/workspaces/3             → un document collaboratif
/monecole/users/12                 → profil d'un membre
```

---

## 13. Guide de décision : où intervenir selon le type de modification

Voici les cas les plus courants et l'endroit où agir :

**« Ajouter un champ sur un modèle existant »** — Il faut une migration de base de données (un fichier dans `db/migrate/`) et une modification du modèle correspondant dans `app/models/`.

**« Ajouter une nouvelle page »** — Il faut un contrôleur dans `app/controllers/`, une vue dans `app/views/`, et une route dans `config/routes.rb`.

**« Ajouter un nouveau type d'événement qui apparaît dans les fils de discussion »** — Créer un nouveau sous-type de Message dans `app/models/message/`, avec son callback de déclenchement.

**« Alerter un utilisateur quand quelque chose se passe »** — Créer un nouveau sous-type de Notification dans `app/models/notification/`.

**« Modifier les règles de qui peut faire quoi »** — Modifier `app/lib/user/permissions.rb` ou `app/lib/membership/permissions.rb`.

**« Ajouter un email automatique »** — Modifier un mailer existant dans `app/mailers/` ou en créer un nouveau, et éventuellement l'intégrer dans les jobs de résumé.

**« Dupliquer ou supprimer une entité complexe »** — Ne pas utiliser un simple `.destroy`. Regarder les jobs existants dans `app/jobs/` qui gèrent la suppression/duplication dans le bon ordre pour éviter les erreurs d'intégrité.

**« Modifier la logique de validation d'une compétence »** — Attention à la cascade. Toucher à `Subscription#complete`, `Subscription#refresh_completed_at`, ou `Evaluation::Exam#add_note` a des effets potentiellement récursifs.

---

## 14. Les pièges à connaître

**Le piège de la cascade** — Modifier le comportement de la complétion d'une Subscription peut remonter récursivement dans l'arbre des compétences. Toujours penser aux compétences parentes.

**Le piège des callbacks** — De nombreux comportements sont déclenchés automatiquement quand un objet est sauvegardé (création de messages système, notifications, badges). Si vous modifiez un modèle, ces effets de bord invisibles peuvent se déclencher de manière inattendue.

**Le piège du devoir rechargé** — Quand un devoir est refusé avec `reject_and_keep_open`, un nouveau Homework vide est automatiquement créé. Si vous comptez les devoirs, il y en a toujours un de plus que ce que vous attendez.

**Le piège des prérequis** — Un apprenant ne peut pas commencer une compétence si ses prérequis ne sont pas validés. Il y a un nombre minimum de prérequis ET des prérequis obligatoires. Les deux conditions doivent être remplies.

**Le piège du message à quatre destinations** — Un message a toujours exactement un destinataire parmi quatre possibles. Oublier cette règle crée des messages orphelins ou visibles nulle part.

---

## 15. Résumé visuel — la machine complète

```
                        ┌──────────┐
                        │   USER   │
                        └────┬─────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              Membership          Subscription ◄─── progression
              (communauté)        (compétence)
                    │                 │
              ┌─────┴──────┐    ┌────┴─────────┐
              │            │    │              │
           Badge       Team  Homework      Exam
           (gamif.)  (groupe) (fichier)  (dialogue)
                                │              │
                            approve/       Note → Note → Note
                            reject          └─► is_accepted?
                                │              │
                                └──────┬───────┘
                                       │
                              Subscription.complete ✓
                                       │
                                 ┌─────┴──────┐
                                 │            │
                            Message      cascade vers
                            système      compétence parente
                                 │
                            Notification
                                 │
                         Email de résumé
```

Tout part de l'utilisateur. Tout converge vers la complétion d'une Subscription. Et tout ce qui se passe autour (messages, notifications, badges, emails) est une conséquence automatique de cette progression.
