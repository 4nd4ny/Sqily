# app/models/message/ — Hiérarchie des Messages

La table `messages` utilise le mécanisme STI de Rails (colonne `type`) pour stocker 15 types de messages distincts dans une seule table. La classe de base `Message` (dans `app/models/message.rb`) définit les associations et scopes communs ; chaque sous-classe ici ajoute sa logique propre.

## Comprendre le routage des destinataires

Un message peut être adressé à **exactement un** des destinataires suivants :

| Colonne | Destinataire | Contexte |
|---|---|---|
| `to_community_id` | une `Community` | Fil de discussion de la communauté |
| `to_skill_id` | une `Skill` | Fil de discussion d'une compétence |
| `to_workspace_id` | un `Workspace` | Fil interne d'un document collaboratif |
| `to_user_id` | un `User` | Message privé direct |

## Types de messages

### Messages saisis manuellement par l'utilisateur

| Classe | Description |
|---|---|
| `Message::Text` | Un message texte classique. Valide la présence de `text`. C'est le type le plus courant. |
| `Message::Upload` | Un fichier partagé dans un fil. Valide la présence de `file_node` (chemin S3). |

### Messages système (déclenchés automatiquement par des callbacks)

Ces messages sont créés sans action directe de l'utilisateur. Ils apparaissent dans les fils pour signaler des événements pédagogiques.

| Classe | Déclencheur | Destinataire | Signification |
|---|---|---|---|
| `Message::NewMembership` | `Membership.after_create` | Community | Quelqu'un vient de rejoindre la communauté. Regroupe plusieurs arrivées consécutives dans un seul message. |
| `Message::NewSubscription` | `Subscription.after_create` | Skill | Quelqu'un commence à apprendre cette compétence. Même logique de regroupement. |
| `Message::NewSkill` | Déclenché manuellement dans `SkillsController` | Community | Une nouvelle compétence vient d'être publiée. |
| `Message::SubscriptionComplete` | `Subscription.after_update` | Skill | Un apprenant a validé cette compétence. Lie au `Homework` approuvé si pertinent. |
| `Message::EventCreated` | `Event.after_create` | Community ou Skill | Un événement vient d'être créé. |
| `Message::PollCreated` | `Poll.after_create` | Community, Skill ou Workspace | Un sondage vient d'être ouvert. |
| `Message::HomeworkUploaded` | `Homework.after_save` | User (privé) | Notifie l'auteur de l'évaluation qu'un devoir a été déposé. Envoie aussi un email. |
| `Message::WorkspacePublished` | `Workspace#publish!` | Community ou Skill | Un document collaboratif vient d'être publié. |

### Messages internes aux Workspaces

Ces messages sont visibles uniquement dans le fil interne du `Workspace` (pas dans les fils communauté/compétence).

| Classe | Événement |
|---|---|
| `Message::WorkspacePartnershipCreated` | Un co-auteur a été invité dans le workspace. |
| `Message::WorkspacePublishedInternal` | Le workspace a été publié (vue interne). |
| `Message::WorkspaceUnpublishedInternal` | Le workspace a été dépublié. |
| `Message::WorkspaceApprovedInternal` | Un lecteur a approuvé le workspace. |
| `Message::WorkspaceRejectedInternal` | Un lecteur a rejeté le workspace. |
| `Message::WorkspaceVersionCreated` | Une nouvelle version du document a été créée. |

## Pattern commun des messages système

Tous les messages système suivent le même pattern :

```ruby
class Message::SomeEvent < Message
  # 1. Le callback est déclaré DANS la sous-classe, pas dans le modèle source
  SomeModel.after_create { Message::SomeEvent.trigger(self) }

  # 2. La méthode .trigger() crée le message
  def self.trigger(record)
    create!(from_user: record.user, to_community_id: record.community_id)
  end
end
```

**Attention** : le fait de déclarer `SomeModel.after_create` dans une sous-classe de `Message` signifie que ce callback n'est enregistré que si la classe `Message::SomeEvent` a été chargée par Rails. En développement ce n'est pas un problème (eager loading), mais c'est à garder en tête pour les tests unitaires isolés.

## Méthodes notables sur `Message` (la classe de base)

- `viewable_by?(user)` — vérifie si l'utilisateur a le droit de voir ce message (logique différente selon le type de destinataire).
- `pinnable_by?(user)` — seuls les experts de la compétence ou les modérateurs peuvent épingler.
- `community` — retourne la `Community` associée, quel que soit le type de destinataire.
- `toggle_deleted_at` — soft delete (le message reste en base mais est masqué).
