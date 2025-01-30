# TP 6

![wordpress-logo](images/wordpress-logo.png)

**Saviez vous que [Wordpress](https://wordpress.com/fr/) est le constructeur de site internet le plus utilisé ?**
![wordpress_market](./images/wordpress_market_share.png)

-> 43% des sites internet ont été réalisés avec WordPress et 63% des blogs 🤯

Vous êtes la nouvelle / le nouveau DevOps Engineer d'une startup 👩‍💻👨‍💻
Vous avez pour objectif de configurer l'infrastructure sur GCP qui hébergera le site de l'entreprise 🌏.

Dans ce TP, l'objectif est de **déployer l'application Wordpress** sur Cloud Run puis Kubernetes en utilisant les outils et pratiques vus ensemble : git, Docker, Artifact Registry, Cloud Build, Infrastructure as Code (Terraform) et GKE.

En bon ingénieur·e DevOps, nous allons découper le travail en  3 parties. Les 2 premières sont complètement indépendantes.

## Partie 1 : Infrastructure as Code

Afin d'avoir une configuration facile à maintenir pour le futur, on souhaite utiliser Terraform pour définir l'infrastructure nécessaire à Wordpress.

**💡 Créez les relations de dépendances entre les ressources pour les créer dans le bon ordre**

Nous allons créer les ressources suivantes à l'aide de Terraform :
- Les APIs nécessaires au bon fonctionnement du projet :
  - `cloudresourcemanager.googleapis.com`
  - `serviceusage.googleapis.com`
  - `artifactregistry.googleapis.com`
  - `sqladmin.googleapis.com`
  - `cloudbuild.googleapis.com`

- Dépôt Artifact Registry avec commme repository_id : `website-tools`

- Une base de données MySQL `wordpress` : l'instance de la base de donnée `main-instance` a été crée pendant le préparation du TP avec la commande `gcloud`

- un compte utilisateur de la base de données

1. Commencer par créer le bucket GCS (Google Cloud Storage) qui servira à stocker le state Terraform.  
  ***Réponse:*** Un bucket existait déjà car j'en avais déjà créer un dans les tp précédents avec la commande suivante : gsutil mb gs://${PROJECT_ID}-tfstate
![bucket](./images/bucket.png)

2. Définir les éléments de base nécessaires à la bonne exécution de terraform : utiliser l'exemple sur le [repo du cours](https://github.com/aballiet/devops-dauphine-2024/tree/main/exemple/cloudbuild-terraform) si besoin pour vous aider. 
  ***Réponse:*** voir main.tf
3. Afin de créer la base de données, utiliser la documentation [SQL Database](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_database) et enfin un [SQL User](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_user)
   1. Pour `google_sql_database`, définir `name="wordpress"` et `instance="main-instance"`
   2. Pour `google_sql_user`, définissez le comme ceci :
      ```hcl
      resource "google_sql_user" "wordpress" {
         name     = "wordpress"
         instance = "main-instance"
         password = "ilovedevops"
      }
      ```
4. Lancer `terraform plan`, vérifier les changements puis appliquer les changements avec `terraform apply`. 
  ***Réponse:*** J'ai effectué les commandes suivantes: terrform init, puis terraform plan et une fois avoir valider les changements j'ai fait terraform apply qui m'a donné le résultat suivant: Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

5. Vérifier que notre utilisateur existe bien : https://console.cloud.google.com/sql/instances/main-instance/users (veiller à bien séléctionner le projet GCP sur lequel vous avez déployé vos ressources). 
  ***Réponse:*** Oui, le user wordpress existe
![sql_user](./images/sql_user.png)

6. Rendez-vous sur https://console.cloud.google.com/sql/instances/main-instance/databases. Quelles sont les base de données présentes sur votre instance `main-instance` ? Quels sont les types ?  
  ***Réponse:*** Il y a:
- information_schema --> Système
- mysql --> Système 
- performance_schema --> Système
- sys --> Système
- wordpress --> Utilisateur
![sql_databases](./images/sql_databases.png)

## Partie 2 : Docker

Wordpress dispose d'une image Docker officielle disponible sur [DockerHub](https://hub.docker.com/_/wordpress)

1. Récupérer l'image sur votre machine (Cloud Shell). 
  ***Réponse:*** `docker pull wordpress`
  ![wordpress_image](./images/wordpress_image.png)

2. Lancer l'image docker et ouvrez un shell à l'intérieur de votre container:  
  ***Réponse:*** `docker run --name container-wordpress -p 8080:80 -d wordpress`
  `docker exec -it cb6 bash`
   1. Quel est le répertoire courant du container (WORKDIR) ?  
     ***Réponse:*** avec la commande `pwd` on obtient /var/www/html
   2. Quelles sont les différents fichiers html contenu dans WORKDIR ?  
     ***Réponse:*** avec la commande `ls *.html` on obtient readme.html

3. Supprimez le container puis relancez en un en spécifiant un port binding (une correspondance de port).  
  ***Réponse:*** J'ai déjà fait un port binding dans ma commande plus haut

   1. Vous devez pouvoir communiquer avec le port par défaut de wordpress : **80** (choisissez un port entre 8000 et 9000 sur votre machine hôte => cloudshell)

   2. Avec la commande `curl`, faites une requêtes depuis votre machine hôte à votre container wordpress. Quelle est la réponse ? (il n'y a pas piège, essayez sur un port non utilisé pour constater la différence). 
     ***Réponse:*** Aucune réponse avec `curl http://localhost:8080`
     Mais une erreur avec `curl http://localhost:8081`, donc le port 8080 est bien utilisé

   3. Afficher les logs de votre container après avoir fait quelques requêtes, que voyez vous ?  
     ***Réponse:***
   ![logs_container](./images/logs_container.png)
   On aperçoit le curl dans le dernier log

   4. Utilisez l'aperçu web pour afficher le résultat du navigateur qui se connecte à votre container wordpress
      1. Utiliser la fonction `Aperçu sur le web`
        ![web_preview](images/wordpress_preview.png)
      2. Modifier le port si celui choisi n'est pas `8000`
      3. Une fenètre s'ouvre, que voyez vous ?  

        ***Réponse:*** Si je choisi 8000 une page avec une erreur s'affiche car ce n'est pas le port que j'ai choisi:
         ![web_error](images/web_error.png)
        Si je choisis 8080 j'ai une ceci qui s'affiche:
        ![web_wordpress](images/wordpress_web.png)

4. A partir de la documentation, remarquez les paramètres requis pour la configuration de la base de données.  

  ***Réponse:*** D'après la documentation j'ai l'impression que les variables nécessaires 
   sont les suivantes:
   - WORDPRESS_DB_HOST
   - WORDPRESS_DB_USER
   - WORDPRESS_DB_PASSWORD
   - WORDPRESS_DB_NAME

5. Dans la partie 1 du TP (si pas déjà fait), nous allons créer cette base de donnée. Dans cette partie 2 nous allons créer une image docker qui utilise des valeurs spécifiques de paramètres pour la base de données.
   1. Créer un Dockerfile
   2. Spécifier les valeurs suivantes pour la base de données à l'aide de l'instruction `ENV` (voir [ici](https://stackoverflow.com/questions/57454581/define-environment-variable-in-dockerfile-or-docker-compose)):
        - `WORDPRESS_DB_USER=wordpress`
        - `WORDPRESS_DB_PASSWORD=ilovedevops`
        - `WORDPRESS_DB_NAME=wordpress`
        - `WORDPRESS_DB_HOST=0.0.0.0`

   3. Construire l'image docker.  
     ***Réponse:*** Avec cette commande : `docker build -t image-wordpress:0.1 .`
     ![docker_images](images/docker_images.png)

   4. Lancer une instance de l'image, ouvrez un shell. Vérifier le résultat de la commande `echo $WORDPRESS_DB_PASSWORD`. 
     ***Réponse:*** Pour lancer une instance: `docker run --name docker-container  -p 8090:80 -d image-wordpress:0.1`. 
       Pour ouvrir un bash: `docker exec -it bd6 bash`. 
         Affichage: ilovedevops. 
   ![docker_password](images/docker_password.png)

6. Pipeline d'Intégration Continue (CI):
   1. Créer un dépôt de type `DOCKER` sur artifact registry (si pas déjà fait, sinon utiliser celui appelé `website-tools`). 
     ***Réponse:*** déjà fait dans la partie 1.
     ![artifact](images/artifact.png)

   2. Créer une configuration cloudbuild pour construire l'image docker et la publier sur le depôt Artifact Registry. 
     ***Réponse:*** Il faut créer une fichier cloudbuild.yaml en mettant bien le chemin de mon artifact

   3. Envoyer (`submit`) le job sur Cloud Build et vérifier que l'image a bien été créée. 
     ***Réponse:*** J'utilise cette commande pour submit: `gcloud builds submit --config cloudbuild.yaml .`
     ![cloudbuild_success](images/cloudbuild_success.png)

## Partie 3 : Déployer Wordpress sur Cloud Run puis Kubernetes 🔥

Nous allons maintenant mettre les 2 parties précédentes ensemble.

Notre but, ne l'oublions pas est de déployer wordpress sur Cloud Run puis Kubernetes !

### Configurer l'adresse IP de la base MySQL utilisée par Wordpress

1. Rendez vous sur : https://console.cloud.google.com/sql/instances/main-instance/connections/summary?
   L'instance de base données dispose d'une `Adresse IP publique`. Nous allons nous servir de cette valeur pour configurer notre image docker Wordpress qui s'y connectera.  
   ***Réponse:*** 34.136.148.219

2. Reprendre le Dockerfile de la [Partie 2](#partie-2--docker) et le modifier pour que `WORDPRESS_DB_HOST` soit défini avec l'`Adresse IP publique` de notre instance de base de donnée.
3. Reconstruire notre image docker et la pousser sur notre Artifact Registry en utilisant cloud build. 
***Réponse:*** on peut refaire un submit

### Déployer notre image docker sur Cloud Run

1. Ajouter une ressource Cloud Run à votre code Terraform. Veiller à renseigner le bon tag de l'image docker que l'on vient de publier sur notre dépôt dans le champs `image` ainsi que le port utilisé par notre application.

   Afin d'autoriser tous les appareils à se connecter à notre Cloud Run, on définit les ressources :

   ```hcl
   data "google_iam_policy" "noauth" {
      binding {
         role = "roles/run.invoker"
         members = [
            "allUsers",
         ]
      }
   }

   resource "google_cloud_run_service_iam_policy" "noauth" {
      location    = google_cloud_run_service.default.location # remplacer par le nom de votre ressource
      project     = google_cloud_run_service.default.project # remplacer par le nom de votre ressource
      service     = google_cloud_run_service.default.name # remplacer par le nom de votre ressource

      policy_data = data.google_iam_policy.noauth.policy_data
   }
   ```

   ☝️ Vous aurez besoin d'activer l'API : `run.googleapis.com` pour créer la ressource de type `google_cloud_run_service`. Faites en sorte que l'API soit activé avant de créer votre instance Cloud Run 😌. 
   ***Réponse:*** j'ai lancé cette commande pour m'assurer d'activer l'API: `gcloud services enable run.googleapis.com --project=devops-449120` 

   Appliquer les changements sur votre projet gcp avec les commandes terraform puis rendez vous sur  pendant le déploiement.

2. Observer les journaux de Cloud Run (logs) sur : https://console.cloud.google.com/run/detail/us-central1/serveur-wordpress/logs.
   1. Véirifer la présence de l'entrée `No 'wp-config.php' found in /var/www/html, but 'WORDPRESS_...' variables supplied; copying 'wp-config-docker.php' (WORDPRESS_DB_HOST WORDPRESS_DB_PASSWORD WORDPRESS_DB_USER)`. 
   ***Réponse:*** On retrouve bien cette erreur
   ![log_error](images/log_error.png)
   2. Au bout de 5 min, que se passe-t-il ? 🤯🤯🤯
   3. Regarder le resultat de votre commande `terraform apply` et observer les logs de Cloud Run. 
   ***Réponse:*** Voici la fin de mon apply : Apply complete! Resources: 2 added, 0 changed, 1 destroyed.

3. Autoriser toutes les adresses IP à se connecter à notre base MySQL (sous réserve d'avoir l'utilisateur et le mot de passe évidemment)
   1. Pour le faire, exécuter la commande
      ```bash
      gcloud sql instances patch main-instance \
      --authorized-networks=0.0.0.0/0
      ```

5. Accéder à notre Wordpress déployé 🚀
   1. Aller sur : https://console.cloud.google.com/run/detail/us-central1/serveur-wordpress/metrics?
   2. Cliquer sur l'URL de votre Cloud Run : similaire à https://serveur-wordpress-oreldffftq-uc.a.run.app
   3. Que voyez vous ? 🙈
   ![wordpress_cloudbuild](images/wordpress_cloudbuild.png)


6. Afin d'avoir un déploiement plus robuste pour l'entreprise et économiser les coûts du service CloudSQL, nous allons déployer Wordpress sur Kubernetes
   1. Rajouter le provider kubernetes en dépendance dans `required_providers`
   2. Configure le provider kubernetes pour se connecter à notre cluster GKE

      ```hcl
      data "google_client_config" "default" {}

      data "google_container_cluster" "my_cluster" {
         name     = "gke-dauphine"
         location = "us-central1-a"
      }

      provider "kubernetes" {
         host                   = data.google_container_cluster.my_cluster.endpoint
         token                  = data.google_client_config.default.access_token
         cluster_ca_certificate = base64decode(data.google_container_cluster.my_cluster.master_auth.0.cluster_ca_certificate)
      }
      ```

   3. Déployer wordpress ainsi qu'une base de donnée MySQL sur le cluster GKE, vous pouvez vous aider de ChatGPT ou de la documentation officielle. Exemple de prompt: 
   ```
   Give me the terraform code to deploy wordpress on kubernetes using kubernetes provider. I want to use MySQL.
   ```
   
   ***Réponse:*** Après avoir modifié le main.tf et relancé les commandes terraform j'ai ce résultat: Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
   J'ai ensuite utilisé cette commande pour obtenir l'IP: `kubectl get svc -n wordpress` 
   4. Rendez vous sur l'adresse IP publique du service kubernetes Wordpress et vérifiez que Wordpress fonctionne 🔥
   ***Réponse:*** Au lien suivant http://34.68.88.144 je vois:
   ![wordpress_with_ip](images/wordpress_with_ip.png)



## BONUS : Partie 4

1. Utiliser Cloud Build pour appliquer les changements d'infrastructure
2. Quelles critiques du TP pouvez vous faire ? Quels sont les éléments redondants de notre configuration ?
   1. Quels paramètres avons nous dû recopier plusieurs fois ? Comment pourrions nous faire pour ne pas avoir à les recopier ?
   2. Quel outil pouvons nous utiliser pour déployer Wordpress sur Kubernetes ? Faites les changements nécessaires dans votre code Terraform.
   3. Comment pourrions nous enlever le mot de passe en clair dans notre code Terraform ? Quelle ressource Kubernetes pouvons nous utiliser pour le stocker ? Faites les changements nécessaires dans votre code Terraform.
