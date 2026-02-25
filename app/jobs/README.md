# app/jobs/ — Background Jobs

Les jobs sont des tâches exécutées en dehors du cycle requête/réponse HTTP. Sqily utilise **ActiveJob** sans adaptateur de file d'attente externe (pas de Sidekiq ni de Resque) : les jobs sont lancés de manière synchrone via `perform_now`, ou planifiés via des tâches cron définies dans `config/schedule.rb` (gem `whenever`).

## Jobs récurrents (cron)

Ces jobs sont déclenchés automatiquement selon un planning.

### `DailySummaryJob`

Envoie un email récapitulatif quotidien à chaque utilisateur ayant activé l'option `daily_summary: true`.

Contenu de l'email :
- Nouveaux messages privés non lus
- Événements à venir dans les prochaines 24h
- Notifications non lues
- Messages épinglés récents
- Sondages terminés auxquels l'utilisateur a participé
- Devoirs en attente de correction (pour les auteurs d'évaluations)

Point d'entrée : `DailySummaryJob.perform_now_for_all_users`

### `WeeklySummaryJob`

Envoie un email récapitulatif hebdomadaire par `Membership` (et non par `User`), pour les utilisateurs ayant activé `weekly_summary: true`.

Contenu : nouveaux messages dans la communauté, nouvelles compétences, nouveaux membres, devoirs en attente, notifications non lues.

Point d'entrée : `WeeklySummaryJob.perform_for_all_membeships`

### `EventReminderNotificationJob`

Envoie un email de rappel à chaque participant inscrit à un événement prévu **demain**.

### `Notification::PollFinished#trigger_for_last_24h`

Déclenché depuis un job cron pour notifier les participants aux sondages qui se sont terminés dans les dernières 24h (voir `app/models/notification/poll_finished.rb`).

## Jobs à la demande

Ces jobs sont déclenchés par une action utilisateur, mais en dehors de la requête HTTP (car trop longs).

### `Community::DeleteJob`

Supprime une communauté et **toute** la donnée associée dans le bon ordre : compétences (récursivement), évaluations, devoirs, exams, messages, memberships. La suppression directe via `community.destroy` ne suffit pas à cause des dépendances complexes.

### `Community::DuplicationJob`

Duplique une communauté entière (structure de compétences, évaluations optionnelles, prérequis) vers une nouvelle communauté. Délègue la duplication de chaque compétence à `Skill::DuplicateJob`.

### `Community::SendStatisticsJob`

Génère un export CSV de toutes les statistiques de toutes les communautés et l'envoie par email à l'administrateur.

### `Skill::DeleteJob`

Supprime une compétence et ses dépendances (évaluations, exams, devoirs, messages, workspaces associés) dans le bon ordre transactionnel.

### `Skill::DuplicateJob`

Duplique une compétence vers une autre communauté, en dupliquant récursivement ses enfants, ses tâches, et optionnellement ses évaluations.

### `CancelEventJob`

Annule un événement : envoie un email à tous les participants inscrits (sauf l'organisateur) puis détruit l'événement. Est planifié à l'avance via `Event#scheduled_at`.

## Organisation du dossier

```
jobs/
├── application_job.rb          # Classe de base (ActiveJob::Base)
├── cancel_event_job.rb
├── daily_summary_job.rb
├── event_reminder_notification_job.rb
├── weekly_summary_job.rb
├── community/
│   ├── delete_job.rb
│   ├── duplication_job.rb
│   └── send_statistics_job.rb
└── skill/
    ├── delete_job.rb
    └── duplicate_job.rb
```
