# app/lib/ — Objets de service

Ce dossier contient des classes Ruby qui n'entrent dans aucune des catégories Rails classiques (modèle, contrôleur, helper, mailer…). Il est **différent** du dossier `lib/` à la racine du projet (qui lui contient des tâches Rake et des assets).

## Pourquoi ce dossier existe ?

Rails encourage à garder les modèles et contrôleurs légers. Quand une logique est trop complexe pour un modèle mais ne correspond pas à un contrôleur, elle vit ici. Le dossier est chargé automatiquement par Rails via `autoload_paths`.

## Contenu

### Form objects

Encapsulent la logique de création/édition pour des cas où un formulaire ne correspond pas directement à un seul modèle.

| Fichier | Rôle |
|---|---|
| `skill_form.rb` | Création et édition d'une `Skill`, avec gestion du parent et des prérequis. |
| `workspace_form.rb` | Création d'un `Workspace` avec sa première `Workspace::Version`. |
| `community_request_form.rb` | Formulaire de demande de création de communauté. |

### Permission objects

Centralisent les règles "qui peut faire quoi". Plutôt que de disperser ces règles dans les contrôleurs, chaque classe de permission regroupe toutes les vérifications d'autorisation pour un contexte donné.

| Fichier | Instancié depuis | Rôle |
|---|---|---|
| `user/permissions.rb` → `User::Permissions` | `user.permissions` | Autorisations liées à l'utilisateur global (modifier un workspace, passer un exam, éditer une évaluation…). |
| `membership/permissions.rb` → `Membership::Permissions` | `membership.permissions` | Autorisations liées à l'appartenance à une communauté (créer des équipes). |

Usage typique dans un contrôleur :

```ruby
# Dans un contrôleur
unless current_user.permissions.can_accept_exam?(@exam)
  redirect_to root_path
end
```

### Statistics objects

Calculent des métriques analytiques sans surcharger les modèles. Mis en cache avec `@statistics ||=` dans les modèles correspondants.

| Fichier | Instancié depuis | Contenu |
|---|---|---|
| `user/statistics.rb` → `User::Statistics` | `user.statistics` | Scores et classements de l'utilisateur dans une communauté. |
| `community/statistics.rb` → `Community::Statistics` | `community.statistics` | Métriques globales de la communauté (export CSV). |

### Utilitaires

| Fichier | Rôle |
|---|---|
| `html_compare.rb` → `HtmlCompare` | Compare deux versions HTML d'un `Workspace` et génère un diff visuel. Utilisé dans `Workspace::Version`. |
| `order_param.rb` → `OrderParam` | Valide et sécurise les paramètres de tri reçus du navigateur, pour éviter les injections SQL dans les `ORDER BY`. |
| `serializer.rb` → `Serializer` | Sérialise des objets Rails en JSON pour les réponses AJAX (alternative légère à jbuilder pour certains cas). |
| `profile_page.rb` → `ProfilePage` | Construit les données d'une page de profil public de `Membership`. |

### Scripts de migration (one-shot)

Ces fichiers ne sont **pas** des migrations Rails. Ce sont des scripts Ruby utilisés ponctuellement pour transformer des données en production, et conservés pour historique.

| Fichier | Contexte |
|---|---|
| `homeworks_to_evaluations_migration.rb` | Migration des anciens devoirs vers le système d'évaluations structurées. |
| `skill_groups_migration.rb` | Migration de l'ancienne structure de groupes de compétences vers l'arbre parent/enfant. |
