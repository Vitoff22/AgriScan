# AgriScan AI — Plan d'implémentation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construire une app mobile avec FlutterFlow de diagnostic de maladies des plantes via chatbot IA conversationnel, avec Firebase comme backend.

**Architecture:** FlutterFlow (frontend) → Firebase Auth + Firestore pour les données privées ; FlutterFlow → fonction serveur authentifiée → API LLM pour le chatbot. La clé du fournisseur LLM reste exclusivement sur le serveur.

**État de ce dépôt :** il contient un plan, une présentation, une charte graphique et un trailer. Aucun export de l'application FlutterFlow ni configuration Firebase déployée n'est fourni. Les règles proposées dans `security/firestore.rules` sont un exemple à intégrer et à valider dans l'émulateur avant tout déploiement ; modifier ce dépôt ne sécurise pas automatiquement un projet Firebase existant.

**Tech Stack:** FlutterFlow (no-code frontend), Firebase Firestore (base de données), Firebase Authentication, API Anthropic/OpenAI

---

## Prérequis

- Compte Firebase (gratuit) sur [console.firebase.google.com](https://console.firebase.google.com)
- Compte FlutterFlow (gratuit) sur [flutterflow.io](https://flutterflow.io)
- Clé API pour le LLM (Anthropic : [console.anthropic.com](https://console.anthropic.com) ou OpenAI : [platform.openai.com](https://platform.openai.com))

---

## PHASE 1: Configuration Firebase (A à Z)

### Task 1: Création du projet Firebase et Firestore

- [ ] **Step 1: Créer le projet Firebase**
  - Aller sur [console.firebase.google.com](https://console.firebase.google.com)
  - Cliquer sur "Ajouter un projet"
  - Nom : `AgriScan AI`
  - Désactiver Google Analytics (optionnel)
  - Cliquer sur "Créer un projet"
  - Attendre que le projet soit prêt (~2-3 min)

- [ ] **Step 2: Activer Firestore Database**
  - Dans le menu de gauche → "Firestore Database"
  - Cliquer sur "Créer une base de données"
  - Mode de sécurité : "Mode production" (accès refusé par défaut)
  - Pour le développement, utiliser l'émulateur Firebase avec des données fictives ; ne pas ouvrir une base accessible sur Internet en mode test.
  - Localisation : sélectionner la région la plus proche (ex: `eur3` pour Europe)
  - Cliquer sur "Créer"

- [ ] **Step 3: Activer Firebase Authentication**
  - Menu de gauche → "Authentification"
  - Cliquer sur "Commencer"
  - Onglet "Méthode de connexion"
  - Activer "Email/Mot de passe"
  - Cliquer sur "Enregistrer"

---

### Task 2: Structure de la base de données Firestore

**Collections à créer :**

- [ ] **Step 1: Collection `users`**
  - Ouvrir Firestore Database
  - Cliquer sur "Démarrer une collection"
  - ID de collection : `users`
  - ID du document : UID du compte Firebase Authentication (`users/{uid}`), jamais un ID aléatoire
  - Ajouter les champs :
    - `email` — Type: `string`
    - `name` — Type: `string`
    - `farm_name` — Type: `string` (optionnel)
    - `location` — Type: `geopoint` (ou `string` si vous stockez une adresse textuelle)
    - `role` — Type: `string`, initialisé à `farmer` ; non modifiable par le client. Les droits d'administration doivent être attribués côté serveur et ne jamais dépendre d'un champ de profil librement éditable.
    - `created_at` — Type: `timestamp` (utiliser Server Timestamp depuis l'app)
  - Cliquer sur "Enregistrer"

- [ ] **Step 2: Collection `diagnostics`**
  - Cliquer sur "Ajouter une collection"
  - ID de collection : `diagnostics`
  - Ajouter les champs :
    - `user_id` — Type: `string` (UID Firebase Authentication du propriétaire, immuable)
    - `culture` — Type: `string` (ex: "tomates", "blé")
    - `symptoms` — Type: `string` (description libre)
    - `conversation` — Type: `array` of `map` ; chaque map contient:
        - `role` (string), `content` (string), `created_at` (timestamp)
    - `diagnosis` — Type: `string` (maladie identifiée)
    - `probability` — Type: `int64` (0-100) ou `double` (0.0-1.0) — choisissez un format et restez cohérent
    - `treatment` — Type: `string` (recommandation)
    - `location` — Type: `geopoint` (coordonnées) ou `string` (adresse)
    - `status` — Type: `string` ("en cours" | "traité" | "surveillance")
    - `created_at` — Type: `timestamp` (Server Timestamp recommandé)

- [ ] **Step 3: Collection `sensor_data`**
  - Cliquer sur "Ajouter une collection"
  - ID de collection : `sensor_data`
  - Ajouter les champs :
    - `user_id` — Type: `string` (UID Firebase Authentication du propriétaire, immuable)
    - `temperature` — Type: `double` (°C)
    - `humidity` — Type: `double` (%)
    - `rainfall` — Type: `double` (mm)
    - `timestamp` — Type: `timestamp` (Server Timestamp ou horodatage capteur)

- [ ] **Step 4: Collection `alerts`**
  - Cliquer sur "Ajouter une collection"
  - ID de collection : `alerts`
  - Ajouter les champs :
    - `user_id` — Type: `string` (UID Firebase Authentication du destinataire, immuable)
    - `type` — Type: `string` ("maladie" | "météo")
    - `message` — Type: `string`
    - `severity` — Type: `string` ("info" | "warning" | "critical")
    - `read` — Type: `boolean`
    - `created_at` — Type: `timestamp`

---

### Task 3: Configurer les règles de sécurité Firestore

- [ ] **Step 1: Accéder aux règles Firestore**
  - Console Firebase → Firestore Database
  - Onglet "Règles"

- [ ] **Step 2: Intégrer et tester les règles restrictives**
  - Utiliser le fichier versionné [`security/firestore.rules`](security/firestore.rules), source unique de l'exemple de règles.
  - Les profils restent privés, le propriétaire d'un document ne peut pas être transféré et un compte client ne peut pas se promouvoir administrateur.
  - Seul le serveur de confiance crée les alertes. Le destinataire peut uniquement les lire, les marquer comme lues ou les supprimer.
  - Les mesures sont créées par leur propriétaire puis restent immuables côté client ; les diagnostics peuvent être modifiés uniquement par leur propriétaire.
  - Les collections et sous-collections non déclarées restent interdites. La communauté conserve des données de démonstration tant qu'un modèle de publication et de modération n'est pas défini.
  - Toutes les requêtes de liste privées doivent filtrer `user_id == current_user.uid` : les règles ne filtrent pas les résultats à la place de la requête.
  - Vérifier la matrice d'autorisation dans [`security/README.md`](security/README.md) avec deux comptes de test et un visiteur anonyme, dans l'émulateur.
  - Si une application utilise déjà `user_ref`, migrer explicitement le schéma et les requêtes vers `user_id` avant d'appliquer ces règles. Ne pas ouvrir les permissions pour contourner une erreur de migration.
  - Publier les règles seulement après validation de l'application et sauvegarde des données existantes.

---

### Task 4: Obtenir les clés d'accès Firebase pour FlutterFlow

- [ ] **Step 1: Récupérer les identifiants du projet**
  - Console Firebase → Paramètres du projet (icône engrenage en haut)
  - Onglet "Général"
  - Copier et sauvegarder :
    - **Project ID** (ex: `AgriScan AI-abc123`)
    - **Project Number** (ex: `123456789`)
    - **API Key** (à créer si absente)

- [ ] **Step 2: Créer une clé API Web**
  - Console Firebase → Paramètres du projet
  - Onglet "Clés API"
  - Cliquer sur "Créer une clé API"
  - Copier la clé générée (ex: `AIza...`)

- [ ] **Step 3: Copier la config Firebase**
  - Console Firebase → Paramètres du projet → Onglet "Général"
  - Scroller vers le bas → "Vos applications"
  - Cliquer sur le bouton "</>" (Web)
  - Copier tout le bloc `firebaseConfig` (il contient apiKey, projectId, databaseURL, etc.)

---

## PHASE 2: Configuration de FlutterFlow

### Task 5: Création du projet FlutterFlow et connexion Firebase

- [ ] **Step 1: Créer un projet FlutterFlow**
  - Aller sur [flutterflow.io](https://flutterflow.io)
  - Se connecter ou créer un compte
  - Cliquer sur "Create New Project" ou "+ New Project"
  - Nom : `AgriScan AI`
  - Template : "Blank" ou "Starter"
  - Cliquer sur "Create"

- [ ] **Step 2: Connecter Firebase à FlutterFlow**
  - Ouvrir le projet FlutterFlow
  - Menu : Settings → "Firebase"
  - Cliquer sur "Connect Firebase"
  - Sélectionner le mode : "Automatic" (recommandé) ou "Manual"
  - Si Automatic : 
    - Cliquer sur "Sign in with Google"
    - Sélectionner le projet Firebase `AgriScan AI`
    - Autoriser FlutterFlow
  - Si Manual :
    - Coller les identifiants Firebase (projectId, apiKey, etc.)
  - Cliquer sur "Verify"

- [ ] **Step 3: Vérifier la connexion**
  - Dans FlutterFlow, aller à "Backend" → "Firestore"
  - Vérifier que les collections apparaissent : `users`, `diagnostics`, `alerts`, `sensor_data`

---

### Task 6: Configuration de l'authentification Firebase dans FlutterFlow

- [ ] **Step 1: Activer le contrôle d'authentification**
  - Menu : Settings → "Authentication"
  - Activer "Firebase Authentication"
  - Sélectionner "Email / Password"

- [ ] **Step 2: Créer la page de connexion**
  - Cliquer sur "Pages"
  - Ajouter une nouvelle page : "login"
  - Ajouter des composants :
    - TextInput pour Email
    - TextInput pour Mot de passe
    - Button "Se connecter"
  - Dans le workflow du bouton :
    - Action : Firebase → Sign in with Email
    - Récupérer email et password des inputs
    - Si succès : naviguer vers "home"
    - Si erreur : afficher message d'erreur

- [ ] **Step 3: Créer la page d'inscription**
  - Ajouter une nouvelle page : "signup"
  - Ajouter des composants :
    - TextInput pour Nom
    - TextInput pour Email
    - TextInput pour Mot de passe
    - TextInput pour Nom de la ferme
    - TextInput pour Localisation
    - Button "Créer mon compte"
  - Dans le workflow du bouton :
    - Action Firebase → Create User (email, password)
    - Créer `users/{UID du compte créé}` avec les infos, `role: farmer` et `created_at: Server Timestamp`
    - Si succès : naviguer vers "home"
    - Si erreur : afficher message

- [ ] **Step 4: Gérer la déconnexion**
  - Dans la navigation principale, ajouter un menu utilisateur
  - Option "Déconnexion"
  - Workflow : Firebase → Sign Out → Rediriger vers login

---

### Task 7: Appeler le LLM par une fonction serveur authentifiée

- [ ] **Step 1: Préparer le backend de confiance**
  - Prévoir une fonction serveur `chat_diagnostic`, par exemple une fonction Firebase appelable, après vérification du coût et du mode de facturation. Ce dépôt n'implémente ni ne déploie cette fonction.
  - Stocker la clé du fournisseur dans un gestionnaire de secrets serveur. Ne jamais l'intégrer dans une API publique FlutterFlow, une variable cliente, un export web ou mobile, ni dans Git.
  - Vérifier le jeton Firebase à chaque appel, refuser les appels anonymes et contrôler les droits sur les diagnostics demandés.
  - Appliquer côté serveur des quotas par UID et globaux, une taille maximale des messages, un nombre maximal de messages, une limite de sortie et un délai d'expiration.
  - Configurer le fournisseur, le modèle et le prompt système côté serveur. Ne pas accepter une URL fournisseur arbitraire ou un rôle système soumis par le client.

- [ ] **Step 2: Connecter FlutterFlow au backend**
  - Appeler la fonction avec le SDK ou le mécanisme compatible avec le protocole choisi ; transmettre le jeton utilisateur et les seuls champs nécessaires au diagnostic.
  - Envoyer par exemple `{ "message": "Les feuilles présentent des taches jaunes." }` après validation de la saisie. La clé fournisseur n'est jamais transmise au client.
  - Intégrer App Check comme protection complémentaire lorsque possible, sans remplacer les contrôles d'identité ni les quotas.
  - Ne pas journaliser les jetons, les clés ou le contenu privé intégral des conversations.

- [ ] **Step 3: Vérifier les limites et les erreurs**
  - Tester les appels anonymes, les jetons expirés, l'accès à un diagnostic tiers, les corps trop volumineux, les limites de quota et les indisponibilités du fournisseur.
  - Afficher les réponses comme du texte non fiable. Ne pas interpréter une réponse LLM comme du HTML exécutable, une instruction privilégiée ou un diagnostic médical/agronomique confirmé.
  - Si une clé fournisseur a déjà été distribuée dans une application ou exposée dans Git, la révoquer et en créer une nouvelle avant de la stocker côté serveur.
  - Voir [`security/README.md`](security/README.md) pour les limites de vérification et les sources officielles.

---

## PHASE 3: Construction des pages et workflows FlutterFlow

### Task 8: Page d'accueil (home)

- [ ] **Step 1: Créer la page home**
  - Ajouter une nouvelle page : "home"
  - Ajouter un AppBar avec :
    - Logo ou titre "🌱 AgriScan AI"
    - Menu utilisateur (avatar + bouton déconnexion)

- [ ] **Step 2: Ajouter une section alertes**
  - Ajouter un ListView connecté à la collection `alerts`
  - Filtrer par : `user_id == current_user.uid` et `read == false`
  - Afficher : icône + message + date + bouton "Marquer comme lu"

- [ ] **Step 3: Ajouter des boutons d'accès rapide**
  - Grille 2x2 avec 4 boutons :
    - "🔍 Diagnostic IA" → naviguer vers "diagnostic"
    - "🗺️ Carte" → naviguer vers "map"
    - "👥 Communauté" → naviguer vers "community"
    - "📊 Tableau de bord" → naviguer vers "dashboard"

- [ ] **Step 4: Ajouter une section actualités**
  - Afficher 2-3 cartes avec :
    - Météo 7 jours (données simulées ou API météo)
    - Conseil de saison (texte fixe)
    - Astuce du mois (texte fixe)

---

### Task 9: Page Diagnostic IA (chatbot)

- [ ] **Step 1: Créer la page diagnostic**
  - Ajouter une nouvelle page : "diagnostic"
  - Ajouter un ListView pour le chat (messages)
  - Chaque message a un conditional :
    - Si `role == "user"` : aligné à droite, fond vert
    - Si `role == "assistant"` : aligné à gauche, fond bleu
    - Si `role == "diagnosis"` : fond orange

- [ ] **Step 2: Ajouter la zone de saisie**
  - TextInput multiligne : "Décrivez les symptômes..."
  - Button "Envoyer" (avec icône flèche ou texte)

- [ ] **Step 3: Créer le workflow d'envoi**
  - Au clic sur "Envoyer" :
    1. Récupérer le texte de l'input
    2. Ajouter le message utilisateur à la liste locale (UI update)
    3. Effacer l'input
    4. Appeler la fonction serveur authentifiée `chat_diagnostic`
    5. Récupérer la réponse
    6. Ajouter la réponse à la liste
    7. Créer un document dans `diagnostics` avec :
       - `user_id`, `culture`, `symptoms`, `conversation` (array de messages), `created_at`
       - Si la réponse contient "DIAGNOSTIC:" : extraire et remplir `diagnosis`, `probability`, `treatment`

- [ ] **Step 4: Gérer les erreurs API**
  - Si erreur réseau : afficher "Impossible de se connecter"
  - Si limite d'API atteinte : afficher "Limite dépassée, réessayez plus tard"

---

### Task 10: Page Carte interactive

- [ ] **Step 1: Créer la page map**
  - Ajouter une nouvelle page : "map"
  - Ajouter un composant Google Map (intégration native FlutterFlow)

- [ ] **Step 2: Afficher les diagnostics sur la carte**
  - Récupérer tous les documents `diagnostics` de l'utilisateur
  - Pour chaque diagnostic, créer un marqueur avec :
    - Position : `location`
    - Info bulle : `culture`, `diagnosis`, `date`
    - Couleur selon `status` (rouge=urgent, orange=surveillance, vert=traité)

- [ ] **Step 3: Ajouter des contrôles**
  - Zoom +/-
  - Centrer sur la localisation actuelle (géolocalisation)
  - Filtrer les marqueurs par date ou statut

---

### Task 11: Tableau de bord (dashboard)

- [ ] **Step 1: Créer la page dashboard**
  - Ajouter une nouvelle page : "dashboard"

- [ ] **Step 2: Afficher des statistiques**
  - Nombre total de diagnostics
  - Maladies détectées (top 5)
  - Cultures principales
  - Taux de réussite des traitements

- [ ] **Step 3: Afficher l'historique**
  - ListView des 10 derniers diagnostics
  - Afficher : date, culture, maladie, statut
  - Cliquer sur un diagnostic pour voir les détails

---

### Task 12: Section Communauté

- [ ] **Step 1: Créer la page community**
  - Ajouter une nouvelle page : "community"

- [ ] **Step 2: Ajouter un forum simplifié**
  - ListView des posts (texte fixe ou données statiques pour la démo)
  - Chaque post : auteur, date, contenu, nombre de likes

- [ ] **Step 3: Définir les permissions avant d'activer le partage**
  - TextInput pour écrire un nouveau post
  - Button "Partager"
  - Conserver les données statiques de démonstration tant que le schéma `posts`, les droits auteur/lecteur, les limites et la modération ne sont pas définis et testés. Les règles proposées refusent cette collection par défaut.
  - Activer ensuite la création d'un post uniquement pour un utilisateur authentifié, en empêchant la modification de l'auteur et la modification/suppression des posts d'autrui.

---

## PHASE 4: Tests et Déploiement

### Task 13: Tests locaux

- [ ] **Step 1: Tester l'authentification**
  - Créer un compte de test
  - Se connecter
  - Vérifier que le document utilisateur est créé dans Firestore
  - Se déconnecter et reconnecter

- [ ] **Step 2: Tester le module diagnostic**
  - Envoyer un message au chatbot
  - Vérifier que la réponse de l'API LLM s'affiche
  - Vérifier que le diagnostic est sauvegardé dans Firestore

- [ ] **Step 3: Tester la navigation**
  - Vérifier que tous les liens fonctionnent
  - Tester sur mobile (mode responsive)

- [ ] **Step 4: Vérifier les données Firestore**
  - Console Firebase → Firestore Database
  - Vérifier les collections : `users`, `diagnostics`, `alerts`

---

### Task 14: Déploiement

- [ ] **Step 1: Générer la version Android/iOS**
  - Dans FlutterFlow : Menu "Deploy" → "Build"
  - Sélectionner Android / iOS
  - Cliquer sur "Build"
  - Attendre la compilation (~5-10 min)

- [ ] **Step 2: Télécharger l'APK (Android) ou IPA (iOS)**
  - Une fois la compilation terminée, télécharger le fichier
  - Tester sur un appareil mobile ou un émulateur

- [ ] **Step 3: Vérifier les protections Firebase avant publication**
  - Console Firebase → Firestore Database
  - Onglet "Règles"
  - Vérifier que la base n'a jamais été ouverte en mode test sur Internet et que les règles validées sont effectivement déployées.
  - Exécuter la matrice de tests avec des comptes de test distincts dans l'émulateur, puis contrôler la configuration réelle sans accéder aux données privées d'autres utilisateurs.
  - Vérifier le schéma, les filtres propriétaire, les quotas serveur, les restrictions des accès IAM et l'absence de clé fournisseur dans l'application distribuée.

---

## Architecture Finale

```text
FlutterFlow ── Firebase Authentication
     │
     ├── Firestore (règles propriétaire ; aucun accès anonyme)
     │
     └── Fonction serveur (authentification, droits, quotas, validation)
                 └── Fournisseur LLM (clé secrète côté serveur)
```

---

## Points clés

✅ **Firebase** = toute la logique backend (authentification, stockage, sécurité)
✅ **FlutterFlow** = interface mobile + workflows
✅ **API LLM** = intelligence du chatbot de diagnostic
✅ **Règles Firestore** = sécurité des données (chaque user ne voit que ses données)
✅ **Collections Firestore** = structure claire et scalable


