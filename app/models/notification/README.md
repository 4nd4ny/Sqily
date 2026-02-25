# app/models/notification/ — Système de notifications

Les notifications sont des alertes in-app affichées à un `Membership` (et non à un `User` directement, car une notification est toujours contextuelle à une communauté). Elles utilisent STI (colonne `type`) exactement comme les `Message`.

## Différence entre Notification et Message

- Un **Message** est visible dans un fil de discussion par tous les membres.
- Une **Notification** est personnelle : elle est adressée à un `Membership` spécifique et apparaît dans le centre de notifications de cet utilisateur.

## Types de notifications

| Classe | Déclencheur | Destinataire | Quand ? |
|---|---|---|---|
| `Notification::BadgeReceived` | `Badge.after_create` | Le membership qui a reçu le badge | Un badge vient d'être attribué |
| `Notification::HomeworkPending` | `Homework.after_save` | L'auteur de l'`Evaluation` | Un candidat a déposé un devoir à corriger |
| `Notification::HomeworkApproved` | `Homework.after_save` | Le candidat (auteur de la `Subscription`) | Son devoir a été accepté |
| `Notification::HomeworkRejected` | `Homework.after_save` | Le candidat | Son devoir a été refusé |
| `Notification::VoteReceived` | `Vote.after_create` | L'auteur du message qui a reçu le vote | Quelqu'un a aimé un de ses messages |
| `Notification::MessagePinned` | `Message.after_save` | Tous les membres de la communauté ou de la compétence (sauf l'auteur) | Un message a été épinglé |
| `Notification::Mention` | `Message.after_save` | L'utilisateur mentionné avec `@nom` | Quelqu'un l'a mentionné dans un message |
| `Notification::PollFinished` | Déclenché manuellement via `DailySummaryJob` | Tous les participants au sondage | Un sondage vient de se terminer |
| `Notification::WorkspaceMessaged` | `Message::Text.after_save` | Tous les co-auteurs du workspace (sauf l'auteur du message) | Un nouveau message a été posté dans le workspace |

## Pattern commun

```ruby
class Notification::SomeEvent < Notification
  belongs_to :related_record  # ex: belongs_to :homework

  # Callback déclaré dans la sous-classe de Notification
  SomeModel.after_save { Notification::SomeEvent.trigger(self) }

  def self.trigger(record)
    # Idempotent : ne crée pas en doublon
    return if where(related_record: record).exists?
    create!(related_record: record, to_membership: ...)
  end

  # Permet de recalculer toutes les notifications de ce type depuis zéro
  def self.replay
    SomeModel.find_each { |record| trigger(record) }
  end
end
```

**Idempotence** : toutes les méthodes `trigger` vérifient d'abord si la notification existe déjà (`where(...).exists?`), ce qui les rend sûres à appeler plusieurs fois (utile via `replay`).

**`Notification.replay`** (dans la classe de base) recalcule l'ensemble des notifications de tous types — utile après une correction de bug ou une migration de données.

## Notification::Mention — détail

La mention est déclenchée par l'analyse du texte d'un `Message::Text`. L'algorithme cherche les tokens `@nom` et tente de matcher des `User` dans la même communauté dont le nom commence par ce token. Il utilise `CHAR_LENGTH(name) DESC` pour favoriser le nom le plus long correspondant (évite de mentionner "Jean" quand on voulait "Jean-Claude").
