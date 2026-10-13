# Sécurité du futur backend AgriScan

Le dépôt contient des supports et un plan de développement, pas l'application FlutterFlow exportée ni les paramètres d'un backend déployé. Les corrections de ce dossier ne modifient aucun service distant. L'état réel de Firebase, de FlutterFlow et des clés fournisseur doit être contrôlé par le propriétaire de ces services.

## Règles Firestore proposées

`firestore.rules` remplace les règles permissives précédemment présentes dans le plan : accès limité au propriétaire, propriétaire immuable, profil incapable de s'attribuer un rôle administrateur et création des alertes réservée au serveur. L'identifiant du propriétaire est le champ `user_id` de type chaîne ; le document utilisateur porte l'UID d'Authentication.

Cet exemple porte sur les autorisations. Il ne remplace pas la validation métier des champs de diagnostic, de conversation et de capteurs. Les données créées par le client restent non fiables : le serveur doit valider les types, tailles, droits et valeurs avant toute utilisation privilégiée. Un diagnostic privé n'est pas une donnée validée par un expert.

Avant utilisation avec une application existante : sauvegarder les données, vérifier le schéma réellement employé, adapter les requêtes de liste, tester dans l'émulateur puis publier les règles. Les anciens documents utilisant `user_ref` nécessitent une migration. N'élargir aucune permission pour éviter cette migration.

### Matrice de tests à exécuter dans l'émulateur

Utiliser deux comptes distincts Alice et Bob ainsi qu'un contexte anonyme. Créer les fixtures d'alertes avec les règles désactivées uniquement dans l'émulateur.

| Scénario | Résultat attendu |
| --- | --- |
| Visiteur anonyme : lecture/écriture de toute collection | Refus |
| Alice crée `users/alice` avec rôle `farmer` et horodatage serveur | Autorisé |
| Alice crée ou modifie `users/bob` | Refus |
| Alice crée un profil `admin` ou modifie/supprime son champ `role` | Refus |
| Alice modifie son nom en conservant rôle et date | Autorisé |
| Alice crée un diagnostic avec `user_id: alice` | Autorisé |
| Alice crée un diagnostic avec `user_id: bob` | Refus |
| Alice lit, modifie ou supprime un diagnostic de Bob | Refus |
| Alice transfère son diagnostic à Bob en changeant `user_id` | Refus |
| Alice modifie les symptômes de son diagnostic en conservant propriétaire et date | Autorisé |
| Alice crée une alerte pour elle ou pour Bob | Refus ; le serveur est responsable de la création |
| Alice marque sa propre alerte comme lue | Autorisé |
| Alice modifie destinataire, message ou gravité de son alerte | Refus |
| Alice lit ou modifie l'alerte de Bob | Refus |
| Alice crée sa mesure avec un horodatage serveur | Autorisé |
| Alice crée une mesure au nom de Bob, ou modifie une mesure existante | Refus |
| Lecture de diagnostics avec un filtre `user_id == alice` | Autorisé pour Alice |
| Lecture de tous les diagnostics sans filtre propriétaire | Refus |
| Écriture dans `posts` ou dans une sous-collection non déclarée | Refus |

Cette matrice reste à exécuter avec Firebase Emulator Suite : aucun résultat d'émulateur ni contrôle du backend réel n'est revendiqué dans ce dépôt. Une revue statique ne suffit pas pour annoncer la validation des règles.

## Appels d'IA et clés

La clé OpenAI/Anthropic ne doit apparaître ni dans les appels API publics FlutterFlow, ni dans un APK, JavaScript, dépôt, capture, ou variable d'application cliente. Utiliser une fonction serveur authentifiée, avec un secret serveur, une limite de taille de requête, une limite de sortie, un délai d'expiration et des quotas par UID et globaux. Le serveur choisit le fournisseur et le modèle ; aucune URL libre du client n'est utilisée.

Vérifier le jeton Firebase et les droits côté serveur. Activer App Check en complément lorsque l'application l'intègre ; ce contrôle ne remplace ni l'authentification ni les quotas. Ne pas enregistrer les jetons ou les clés dans les journaux. Aucun service payant n'est activé par ces fichiers : confirmer les conditions de facturation avant de déployer une fonction ou d'utiliser un fournisseur d'IA.

Si une clé LLM a déjà été intégrée à une application distribuée ou à l'historique Git, la révoquer et la remplacer depuis la console du fournisseur. La retirer d'un fichier ne suffit pas. La configuration publique Firebase n'est pas, à elle seule, une clé privée d'administration ; sa sécurité repose aussi sur les règles, les restrictions appropriées et les accès IAM.

## Sources

- [Conditions et données existantes/futures dans les règles Firestore](https://firebase.google.com/docs/firestore/security/rules-conditions)
- [Champs autorisés et champs modifiables](https://firebase.google.com/docs/firestore/security/rules-fields)
- [Tests de règles avec l'émulateur](https://firebase.google.com/docs/rules/unit-tests)
- [Fonctions appelables et contexte d'authentification](https://firebase.google.com/docs/functions/callable)
- [Secrets serveur](https://firebase.google.com/docs/functions/config-env)
