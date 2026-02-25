# app/models/evaluation/ — Le cœur de la VMC

Ce dossier contient les modèles qui implémentent le processus de **validation mutuelle des compétences**. C'est la partie la plus complexe du domaine métier de Sqily.

## Rappel du vocabulaire

- **Évaluation** (`Evaluation`) : une épreuve définie par un expert pour valider une compétence. C'est un template.
- **Défi** (`Evaluation::Exam`) : une session de validation entre un candidat et un examinateur, basée sur une `Evaluation`.
- **Note** (`Evaluation::Note`) : un échange (message) dans le cadre d'un défi.
- **Brouillon** (`Evaluation::Draft`) : le texte préparatoire du candidat avant de lancer un défi.

## Relations

```
Evaluation (le template du défi)
├── belongs_to :skill        ← pour quelle compétence ?
├── belongs_to :user         ← qui a créé ce défi ?
└── has_many :exams          ← toutes les sessions basées sur ce template
      │
      └── Evaluation::Exam (une session)
            ├── belongs_to :evaluation    ← le template
            ├── belongs_to :subscription  ← le candidat (via sa subscription)
            ├── belongs_to :examiner      ← l'expert qui valide
            └── has_many :notes           ← les échanges
                  │
                  └── Evaluation::Note
                        ├── belongs_to :exam
                        ├── belongs_to :user   ← qui a écrit cette note ?
                        ├── is_accepted        ← true = la compétence est validée
                        └── is_rejected        ← true = note de refus de l'examinateur

Evaluation::Draft
├── belongs_to :subscription  ← le candidat
└── belongs_to :evaluation    ← le défi visé
```

## Cycle de vie d'un Exam

```
1. Le candidat choisit une Evaluation pour sa Subscription
   └─► Evaluation::Draft créé (avec son texte de candidature)

2. draft.submit → Evaluation#start(subscription, content) appelé
   └─► Examinateur sélectionné par pick_examiner_for(subscription)
   └─► Evaluation::Exam créé
   └─► Evaluation::Note initiale créée (la réponse du candidat)

3. L'examinateur consulte l'exam et ajoute une Note
   └─► Si is_accepted: true  → subscription.complete(examiner) → compétence validée ✓
   └─► Si is_rejected: true  → le candidat peut soumettre une nouvelle Note
   └─► Sinon                 → échange libre jusqu'à résolution

4. Résolution finale
   ├─► Exam complété (is_accepted: true sur une Note)
   └─► Exam annulé par le candidat (exam.cancel)
```

## Sélection de l'examinateur — `Evaluation#pick_examiner_for`

L'algorithme choisit parmi les **experts** de la compétence (utilisateurs dont la `Subscription` est `completed_at`). Il privilégie dans l'ordre :

1. L'expert le **moins occupé** (le moins d'exams en cours pour cette évaluation) **dans la même équipe** que le candidat, **qui n'a pas déjà examiné ce candidat** pour ce défi.
2. L'expert le moins occupé toutes équipes confondues, jamais utilisé pour ce candidat.
3. En dernier recours : un expert parmi les 10 moins occupés, tiré au sort (`.sample`).

## Invariants importants

- **Un seul exam en cours par candidat/compétence** : le scope `ongoing` filtre les exams non annulés et sans note acceptée. Un candidat doit annuler son exam actuel avant d'en démarrer un nouveau.
- **Les droits de validation** sont vérifiés dans `Evaluation::Exam#add_note` : seul l'examinateur peut poser `is_accepted: true`. Si un autre utilisateur envoie cette valeur, elle est ignorée.
- **La complétion de la Subscription** est déclenchée dans une transaction depuis `add_note`, garantissant l'atomicité.

## Homework — chemin alternatif de validation

`Homework` (dans `app/models/homework.rb`) est une alternative plus simple à l'`Exam`. Au lieu d'un échange interactif, le candidat uploade un fichier que l'auteur de l'`Evaluation` approuve ou rejette.

```
Subscription (candidat)
└── has_many :homeworks

Evaluation (auteur)
└── has_many :homeworks

Homework
├── approve(by_user)  → subscription.complete(by_user) + email
├── reject            → email de refus
└── reject_and_keep_open → rejette ET crée un nouveau Homework vide pour une nouvelle tentative
```

Un `Homework` a trois états mutuellement exclusifs : `pending` (déposé, en attente), `approved`, `rejected`. L'état est lu via les colonnes `approved_at` et `rejected_at` (null = en attente).
