# Aigle Vente Web API

API ASP.NET Core 9 pour le schéma MySQL `aiglevente` fourni dans
[`aiglevente.sql`](./aiglevente.sql).

## Modèle et périmètre

Les modèles EF Core correspondent aux tables métier du dump : catégories et
produits, boutiques et vendeurs, clients, commandes, paniers, stocks,
promotions, avis, factures, livraisons, messages et tables métier associées. Les tables
techniques Laravel (cache, jobs, sessions, migrations, permissions) et Livewire
ne font pas partie du contexte API. Le modèle `User` est conservé pour les
relations, mais ses données d'authentification ne sont exposées par aucune route.

Les routes CRUD actuellement publiées sont :

| Ressource | Routes |
| --- | --- |
| Authentification | `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/google`, `GET /api/auth/me` |
| Catégories produit | `GET/POST /api/categories`, `GET/PUT/DELETE /api/categories/{id}` |
| Produits | `GET/POST /api/products`, `GET/PUT/DELETE /api/products/{id}` |
| Lieux | `GET/POST /api/locations`, `GET/PUT/DELETE /api/locations/{id}` |
| Catégories de boutiques | `GET/POST /api/shop-categories`, `GET/PUT/DELETE /api/shop-categories/{id}` |
| Boutiques | `GET/POST /api/shops`, `GET/PUT/DELETE /api/shops/{id}` |
| Catégories produit par boutique | `GET/POST /api/category-product-shops`, `GET/PUT/DELETE /api/category-product-shops/{id}` |
| Visites des boutiques | `GET/POST /api/shop-visits`, `GET/PUT/DELETE /api/shop-visits/{id}` |
| Détails produit | `GET/POST /api/product-details`, `GET/PUT/DELETE /api/product-details/{id}` |
| Photos | `GET/POST /api/photos`, `GET/PUT/DELETE /api/photos/{id}` |
| États produit | `GET/POST /api/product-states`, `GET/PUT/DELETE /api/product-states/{id}` |
| Promotions | `GET/POST /api/promotions`, `GET/PUT/DELETE /api/promotions/{id}` |
| Avis | `GET/POST /api/reviews`, `GET/PUT/DELETE /api/reviews/{id}` |
| Stocks | `GET/POST /api/stocks`, `GET/PUT/DELETE /api/stocks/{id}` |
| Mouvements de stock | `GET/POST /api/stock-movements`, `GET/PUT/DELETE /api/stock-movements/{id}` |
| Messages | `GET/POST /api/messages`, `GET/PUT/DELETE /api/messages/{id}` |
| Commandes | `GET/POST /api/orders`, `GET/PUT/DELETE /api/orders/{id}` |
| Lignes de commande | `GET/POST /api/order-products`, `GET/PUT/DELETE /api/order-products/{id}` |
| Paniers | `GET/POST /api/paniers`, `GET/PUT/DELETE /api/paniers/{id}` |
| Articles de panier | `GET/POST /api/panier-items`, `GET/PUT/DELETE /api/panier-items/{id}` |
| Réseaux sociaux des vendeurs | `GET/POST /api/socials`, `GET/PUT/DELETE /api/socials/{id}` |
| Listes de souhaits | `GET/POST /api/wishlists`, `GET/PUT/DELETE /api/wishlists/{id}` |

Les listes acceptent `limit` de 1 à 100 (50 par défaut). Une catégorie ne peut
pas être supprimée tant qu'elle est utilisée par un produit. Les références
boutique, catégorie et état sont validées avant la création ou modification
d'un produit.
Les listes des nouvelles ressources acceptent aussi `limit` de 1 à 100 (50 par
défaut). Un lieu référencé par un client ou un vendeur ne peut pas être
supprimé. La suppression d'une boutique est refusée si elle a des produits,
visites ou associations de catégories produit, afin d'éviter une suppression
en cascade de données métier. Les mouvements de stock empêchent aussi la
suppression d'une boutique. L'IP et le User-Agent d'une visite sont capturés
depuis la requête HTTP plutôt qu'acceptés dans son corps.

Les opérations sur les stocks sont auditées : initialiser une quantité ou
l'ajuster crée un mouvement correspondant; l'API refuse les retraits qui
dépasseraient la quantité disponible. Modifier ou supprimer un mouvement
applique aussi sa différence au stock dans une transaction. Un stock ne peut
être supprimé que lorsque sa quantité est nulle.

Les photos sont envoyées en `multipart/form-data` sur `POST /api/photos` avec
les champs `file` et `productId`, et éventuellement `description`. Les images
JPEG, PNG, GIF et WebP jusqu'à 10 Mo sont acceptées; le type est vérifié à
partir du contenu et non du nom de fichier. L'API retourne dans `image` une URL
publique de la forme `/uploads/<nom-généré>`. `PUT /api/photos/{id}` peut aussi
recevoir un nouveau `file`; sans fichier, il ne modifie que les métadonnées et
conserve l'image existante. `DELETE /api/photos/{id}` supprime l'enregistrement
et le fichier téléversé associé. Les anciennes URL externes sont conservées et
ne provoquent pas de suppression de fichier local.

En local, les fichiers sont enregistrés dans `wwwroot/uploads`. Docker Compose
monte ce répertoire dans le volume persistant `photo_uploads`, afin que les
fichiers survivent au remplacement ou au redémarrage du conteneur API.

Les commandes reçoivent un numéro unique généré par l'API; le compteur journalier
est également déterminé à la création. Une commande ne peut être supprimée
tant qu'elle possède des lignes de commande, une livraison ou une facture.
La suppression d'un panier supprime aussi ses articles, conformément aux clés
étrangères du schéma.

## MySQL

Créer/importer au préalable la base `aiglevente` avec le script SQL. L'API ne
crée ni ne migre automatiquement la base existante.

Fournir la chaîne de connexion via la configuration .NET, sans enregistrer le
mot de passe dans le dépôt :

```sh
export ConnectionStrings__AigleVente='Server=127.0.0.1;Port=3306;Database=aiglevente;User=VOTRE_UTILISATEUR;Password=VOTRE_MOT_DE_PASSE;CharSet=utf8mb4'
export Jwt__Secret="$(openssl rand -hex 32)"
dotnet run
```

Le serveur MySQL doit être compatible avec MySQL 8.0.36 ou supérieur. La
configuration de lancement HTTP utilise le port 5237 et HTTPS le port 7165.

## Démarrage avec Docker Compose

Docker Engine avec le plugin Compose est requis. Copier le modèle de variables,
puis définir des mots de passe propres à votre environnement et une clé
`JWT_SECRET` générée avec `openssl rand -hex 32` :

```sh
cp .env.example .env
# Définir MYSQL_PASSWORD, MYSQL_ROOT_PASSWORD et JWT_SECRET dans .env
docker compose up --build
```

Compose démarre l'API, MySQL 8.0.36 et phpMyAdmin. Le serveur MySQL attend son
initialisation avant que les deux autres services ne démarrent.

| Service | Adresse sur la machine hôte |
| --- | --- |
| API | `http://localhost:8080` |
| OpenAPI | `http://localhost:8080/openapi/v1.json` |
| Scalar | `http://localhost:8080/scalar/v1` |
| phpMyAdmin | `http://localhost:8081` |
| MySQL | `127.0.0.1:3307` |

Les ports peuvent être remplacés dans `.env` (`APP_PORT`, `PHPMYADMIN_PORT`,
`MYSQL_PORT`). Connectez-vous à phpMyAdmin avec le nom d'hôte `mysql` et les
identifiants `MYSQL_USER` / `MYSQL_PASSWORD` configurés dans `.env`.

Au premier démarrage, MySQL importe automatiquement `aiglevente.sql`, puis
`auth-google-logins.sql` crée la table de liaison des identités externes. Les
données sont conservées dans un volume persistant. Les scripts d'initialisation
ne sont rejoués que si le volume `mysql_data` est neuf. Pour
arrêter les services sans supprimer les données, utilisez `docker compose down`.
La suppression de ce volume efface définitivement les données MySQL.

## Authentification

- `POST /api/auth/register` crée un compte `Client` ou `Seller` avec un mot de
  passe et un profil métier. Le vendeur doit fournir un téléphone; le sexe du
  client peut rester non renseigné.
- `POST /api/auth/login` accepte l'e-mail ou le téléphone et le mot de passe.
  Les mots de passe sont hachés avec BCrypt; les hachages bcrypt importés depuis
  Laravel sont vérifiés avec le même algorithme.
- `POST /api/auth/google` vérifie un **Google ID token** côté serveur à partir
  de `GOOGLE_CLIENT_ID`. À la première connexion, indiquer aussi `accountType`.
  Un compte existant n'est automatiquement lié que si Google confirme l'e-mail.
  L'identité `sub` est conservée dans `external_logins`; le jeton Google n'est
  pas stocké.
- La connexion retourne un jeton JWT Bearer. Configurer
  `JWT_SECRET`, `JWT_ISSUER`, `JWT_AUDIENCE` et `JWT_EXPIRES_MINUTES`.
  Configurer `GOOGLE_CLIENT_ID` avec l'identifiant du client OAuth Google
  autorisé à émettre les ID tokens pour l'API.
- `GET /api/auth/me` requiert l'en-tête `Authorization: Bearer <jeton>`.

La propriété `_id` du modèle identifie les entités Aigle Vente; elle n'est pas
réutilisée comme identifiant Google. Le `sub` Google est conservé dans la table
dédiée `external_logins`.

Si le volume MySQL existe déjà, les scripts d'initialisation ne sont pas
rejoués. Importer une seule fois `auth-google-logins.sql` dans `aiglevente`
avec phpMyAdmin pour activer les liaisons Google.

## Documentation et tests manuels

En environnement `Development` :

- OpenAPI : `https://localhost:7165/openapi/v1.json`
- Interface Scalar : `https://localhost:7165/scalar/v1`

Les exemples pour chacune des routes sont dans
[`aigle-vente-webapi-2.http`](./aigle-vente-webapi-2.http).

Les erreurs utilisent Problem Details. Les champs `DateTime` du dump MySQL sont
exposés comme `DateTimeOffset` UTC dans les DTO de réponse.
# aigle-vente-webapi-2
