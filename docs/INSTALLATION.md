# Créer un projet API Symfony avec Docker

# Installation du projet 

> [!TIP] 
> La dockerisation du projet permet de mettre en place un environnemnt de travail PHP. Vous trouverez dans le container une version PHP>8 avec ses extensions, un serveur WEB (`NGNIX`), un gestionnaire de dépendances PHP (`Composer`) ainsi que le client SYMFONY (`SYMFONY CLI`)
>
> Vous trouverez aussi une base de données (`mysql`) ainsi qu'un gestionnaire de BDD (`phpmyadmin`)

<details>
<summary>Installation</summary>

## Cloner le dépôt du projet
```bash
git clone <lien>
```
## Créer une image docker

```bash
docker compose build
```
## Créer un container avec les services 

```bash
docker compose up -d
```
## Connectez vous au conteneur 

```bash
docker exec -it php8-symfony bash
```
## Installer les dépendances Symfony
Dans l'invite de commande du container tapez :
```bash
composer install
```

## Vérification du container et des services
Rendez-vous sur les liens 

- http://localhost:8080 : Lien vers symfony
- http://localhost:9000 : Lien vers phpMyAdmin

</details>

> [!WARNING]
> A partir de maitenant toutes les commandes devront se faire dans l'invite de commande du container [voir ici](#connectez-vous-au-conteneur)

## Créer des entités et controllers

Nous avons besoin de 3 librairies :

**Makerbundle**
```sh
composer require --dev symfony/maker-bundle
```

**Profiler Pack**
```sh
composer require --dev symfony/profiler-pack
```

**ORM Doctrine**
```sh
composer require symfony/orm-pack
```

### Création d'une entité

```sh
symfony console make:entity
```
![make-entity](images/make-entity.png)
![field-entity](images/fields-entity.png)

### Configuration de la base de données

Ajoutez la configuration adéquate dans le fichier `.env.local` selon la configuration de docker.

### Migration 

```sh
symfony console make:migration
```
![migration](images/migration.png)

Une fois la vérification de la requête générée :

```sh
symfony console doctrine:migrations:migrate
```

![migrate](images/migrate.png)

Vérifier l'intégration des tables dans votre SGDB

![table](images/table.png)

# Installer API Platform

```sh
symfony composer require api
```

Nous avons maintenant accès à la route `/api`

![API Platform web](images/API-Platform-web.png)

## Définir une entité comme ressource pour l'API Platform

Ouvrez l'entité `Roles` créée précedemment et ajouter les attributs suivants :

![Annotation entity](images/Annotation-entity-api.png)

Retournez sur la page de l'API

![API with ressource](images/API-Platform-with-ressource.png)

## Ajouter des données dans la base de données

Créer 2 valeurs pour la table `Roles`

## Testons API Platform

Sur l'interface d'API Platform (http://127.0.0.1:8000/api), allez dans la partie `GET : /api/roles` puis déroulez l'onglet. Cliquez ensuite sur le boutton `Try it out` puis `Execute`

![tryoutit](images/Tryitout-execute.gif)

Voici un exemple de résultat : 

```json
{
    "@context": "/api/contexts/Roles",
    "@id": "/api/roles",
    "@type": "Collection",
    "totalItems": 2,
    "member": [
        {
        "@id": "/api/roles/1",
        "@type": "Roles",
        "id": 1,
        "name": "ROLE_ADMIN"
        },
        {
            "@id": "/api/roles/2",
            "@type": "Roles",
            "id": 2,
            "name": "ROLE_USER"
        }
    ]
}
```

# Configurer l'authentification JWT 

## Créer d'un entité User

```sh
symfony console make:user
```
![make user](images/make-user.png)

> [!IMPORTANT]
> N'oubliez pas de générer la migration de la table `User` vers la base de données

## Mettre l'entité User en ressource POST

Ajoutez les attributs API correspondants à votre entité

<details>
<summary>Correction</summary>

```php
namespace App\Entity;

use ApiPlatform\Metadata\Post;
use App\Repository\UserRepository;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;

#[ORM\Entity(repositoryClass: UserRepository::class)]
#[ORM\UniqueConstraint(name: 'UNIQ_IDENTIFIER_EMAIL', fields: ['email'])]
#[Post]
class User implements UserInterface, PasswordAuthenticatedUserInterface
```
</details>

Nous avons maintenant accès à la méthode `POST`

![api platform user](images/api-platform-user.png)

## Installation du bundle JWT

### Installation du bundle
```sh
composer require lexik/jwt-authentication-bundle
```

### Génération des clés "public" et "private"

```sh
symfony console lexik:jwt:generate--keypair
```
La génération des clés ajoute des informations dans votre projet :
1. Création des fichiers private.pem et public.pem dans le dossier config/jwt
2. Ajout des clés dans les variables d'environnement de Symfony
   
```
###> lexik/jwt-authentication-bundle ###
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=f98bb1b8f2e797abfd79c9f1b281cc46c9401ea39937fc8c320cd9522e94af52
###< lexik/jwt-authentication-bundle ###
```
3. Création d'un fichier de configuration du bundle : lexik_jwt_authentication.yaml
dans le dossier config\packages\

```yml
lexik_jwt_authentication:
    secret_key: '%env(resolve:JWT_SECRET_KEY)%'
    public_key: '%env(resolve:JWT_PUBLIC_KEY)%'
    pass_phrase: '%env(JWT_PASSPHRASE)%'
```

### Configuration du SecurityBundle de Symfony 

```yml
# config/packages/security.yaml
security:
    ...
    firewalls:
    ...
        main:
            stateless: true
            provider: app_user_provider
            json_login:
                check_path: /auth
                # Route d'authentification
                username_path: email
                # Identifiant d'identification
                password_path: password
                # Mot de passe
                success_handler: lexik_jwt_authentication.handler.authentication_success
                # Actions si l'authentification est un succès
                failure_handler: lexik_jwt_authentication.handler.authentication_failure
                # Actions si l'authentification est un échec
            jwt: ~
    ...
```

Modifions aussi les controles d'accès au url en ajoutant les routes vers l'API et l'authentification

```yml
access_control:
# Tout le monde peut accèder à cette URL, même s'il n'est pas connecté.
- { path: ^/api/$, roles: PUBLIC_ACCESS }
# Les pages ou services liés à l'authentification (connexion, inscription, ect..) sont publiques.
- { path: ^/auth,roles: PUBLIC_ACCESS }
# Pour accèder à toutes les autres URL qui commencent par /api/ ( autre que la racine)
# il faut être connecté et avoir une session authentifiée.
- { path: ^/api/, roles: IS_AUTHENTICATED_FULLY }
```
Enfin, déclarer la route utilisée pour l'authentification
/auth

```yml
# config/routes.yaml
controllers:
    resource: routing.controllers
    type: attribute

auth:
    path: /auth
    methods: ['POST']
```

Vérifier la route dans votre navigateur.

### Générer un token d'authentification

Créer d'abord un utilisateur dans votre base de données.

Rendez-vous dans l'onglet `Login Check` d'API Platform puis sélectionner ``Try it out`` comme nous l'avions fait pour tester les
``Roles`` et éditer les valeurs du schéma :

```json
// Le mot de passe envoyé est en clair, API Platform et Symfony se charge de le crypter/chiffrer
{
    "email": "jhon@doe.fr",
    "password": "monsupermotdepasse"
}
```

J'obtiens en réponse :

```json
{
"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpYXQiOjE3ODkxMDg4NzUsImV4cCI6MTc4OTExMjQ3NSwicm9sZXMiOlsiUk9MRV9VU0VSIl0sInVzZXJuYW1lIjoiamhvbkBkb2UuZnIifQ.CZ4Ne0gX4XPCFE4PXrTM18ULJQml5eostCP5ZOVLO8KX5Kij1SLEsCt5O2rV_WbtJ-_W5dDHVgt-y9surNU1KYXnNvOaboSxXDnKGGwjqg7DH9OUW90KcNJhv7oqU-JM4WUB-0JADyNOdaE01ZaPMXJCWMnk3S1zaS_82bd7uGFqDWD9JD-1ki9XlEm-2rO8Y9Q83VfJBYz8FkHb2LLI1xBCsJ0n8h0Y1OuTUuu14PGre-ANKcJJhs0NYMewaz4Nz520OKxzXPg0VcKXUs6v-VH1r2Q81Dctj7LGZeTmhwap9K2qxLiVCVPyyYOT82t0IODblt9Exbms_JQY1y-j_A"
}
```

> [!WARNING]
> Attention, chaque token est différent, ne faites pas un copier/coller du code ci-dessus car il sera forcément différent de votre application

### Ajouter l'authentification sur API Platform

#### Configuration d'API Platform

```yml
# api/config/packages/api_platform.yaml
api_platform:
    swagger:
        api_keys:
            JWT:
                name: Authorization
                type: header
```

Le bouton « Authorize » s'affichera automatiquement dans Swagger UI. Vous observerez aussi des cadenas sur chaque opération :

![cadenas](images/cadenas.png)

Maintenant si vous essayez d'envoyer une requête à l'API vous devriez avoir le message suivant:

```json
{
    "code": 401,
    "message": "JWT Token not found"
}
```

#### Ajouter une clé API

Cliquez sur le bouton `Authorize` puis ajouter le token généré précédemment dans la value:

![token](images/token-jwt.png)

![valid token](images/valid-token.png)

Vous pouvez maintenant observer que les cadenas sur API Platform sont tous fermés.

#### Tester les opérations avec l'authentification 

Nous pouvons observer dans la parti ``Responses`` la requête ``Curl`` envoyé à l'API. Notre système a permit l'ajout d'un paramètre ``Authorization: Bearer`` suivi de notre Token, afin d'envoyer une authentification et l'autorisation d'accéder à nos opérations.

```sh
curl -X 'GET' \
    'http://127.0.0.1:8000/api/roles?page=1'\ 
    -H 'accept: application/ld+json' \ 
    -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpYXQiOjE3ODkxMTM2NDksImV4cCI6MTc4OTExNzI0OSwicm9sZXMiOlsiUk9MRV9VU0VSIl0sInVzZXJuYW1lIjoiamhvbkBkb2UuZnIifQ.eRhPWncsuNM4icFjiFtL6gBccDjGiPT_mb4NLAPsTkzF0Yt2_3PiLi9HfvcMetc1-xy0O1iBTmloMbt0JMeTGGPoGIgCDR5Ew73gEqJU0NrVMzJYQ1oZN9y9e1l555fVJO_WnawMPUFOjgeLOGQ0WOKBGBrrNsjn1QU86wtnzlQMdD6jO10WhmAsljzgk6DOLBK4DPULFk65ADZ8OgWQfKI67UgX7zj8EGKiZuwTz1QOr93-10g1ecH8XeG9zDTwHJW0U5oyzW__TKokeU4XUe4YzLZ5XZVuuYJ0yVqUUkZT3RSK8vib-myQKMryfetb2aJmSp_3gj6aOGJqtt-Ndg'
```

Nous recevons bien les informations de notre API.

Si le ``Token`` renseigné est incorrect, vous devriez avoir le message suivant :

```json
{
    "code": 401,
    "message": "Invalid JWT Token"
}
```

Par contre, si vous avez le message suivant :
```json
{
    "code": 401,
    "message": "Expired JWT Token"
}
```
C'est que votre token n'est plus valide, il faut donc revenir à l'étape "générer un token d'authentification".
