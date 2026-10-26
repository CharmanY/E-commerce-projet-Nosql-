# E-commerce Cloud Native – Projet NoSQL

Projet de groupe du cours **NoSQL – M1 Data Engineering & IA (EFREI Paris)**.
Conception d'une marketplace e-commerce cloud native composée de plusieurs services découplés par événements.

## Équipe

| Membre | GitHub |
|---|---|
| Loïc AKAMGA | [@LOIC754](https://github.com/LOIC754) |
| Deiss YEHOUENOU | [@CharmanY](https://github.com/CharmanY) |

---

## Sommaire

1. [Étape 1 – Initialisation du projet](#étape-1--initialisation-du-projet)
2. [Étape 2 – Les services](#étape-2--les-services)
3. [Étape 3 – Les agrégats](#étape-3--les-agrégats)
4. [Étape 4 – Découplage par événements et réplica](#étape-4--découplage-par-événements-et-réplica)
5. [Étape 5 – Projection](#étape-5--projection)
6. [Stockage des agrégats dans Redis](#stockage-des-agrégats-dans-redis)
7. [Gestion de la survente](#gestion-de-la-survente)
8. [Persistance de Redis](#persistance-de-redis)
9. [Lancer le projet](#lancer-le-projet)
10. [Arborescence](#arborescence)

---

## Étape 1 – Initialisation du projet

- Équipe de 2 personnes.
- Dépôt Git créé, enseignant invité.
- Descriptif du projet : ce README.

**Problème métier traité** : un client passe une commande sur une marketplace. La commande doit être validée par rapport au stock disponible, **sans survente** (exemple : un client commande 10 articles alors qu'il n'en reste que 3), puis être livrée.

---

## Étape 2 – Les services

L'application est découpée en services. Chaque service **possède son propre agrégat** et ne lit jamais les données d'un autre service directement.

| Service | Responsabilité | Agrégat possédé |
|---|---|---|
| **Commande** (`orderService`) | reçoit la commande du client, émet les événements de commande | Commande |
| **Stock** (`stockService`) | réserve les articles et empêche la survente | Stock |
| **Livraison** (`deliveryService`) | prépare et expédie la commande à partir d'un réplica | Livraison (+ réplica Commande) |
| **Query** (`orderQueryService`) | construit une vue consolidée pour l'affichage client | Projection `order_views` |

### Architecture
![Architecture de l'application](images/Architecture.png)

Comme dans le projet de l'enseignant, **Redis coordonne la livraison des événements** entre services. Le routeur copie chaque événement dans **une file par service consommateur**, car une List Redis est consommée de façon destructive (`LPOP` / `LMOVE`) : avec une file partagée, un événement ne serait reçu que par un seul service.

---

## Étape 3 – Les agrégats

Un agrégat est une frontière transactionnelle métier : il est modifié en une seule opération par le service qui le possède.

### Agrégat Commande

```json
{
  "orderId": "CMD-42",
  "customerId": "CLI-7",
  "lines": [
    { "productId": "PROD-12", "quantity": 10 }
  ],
  "deliveryAddress": "12 rue de Paris, 94120 Fontenay",
  "total": 250,
  "status": "PENDING"
}
```

Statuts possibles : `PENDING` → `CONFIRMED` ou `REJECTED` → `SHIPPED`.

### Agrégat Stock

```json
{
  "productId": "PROD-12",
  "available": 3,
  "reserved": 0
}
```

### Agrégat Livraison

```json
{
  "deliveryId": "LIV-43",
  "orderId": "CMD-43",
  "carrier": "",
  "status": "WAITING"
}
```

Statuts possibles : `WAITING` → `SHIPPED`.

---

## Étape 4 – Découplage par événements et réplica

### Enveloppe commune des événements

```json
{
  "eventId": "uuid",
  "type": "OrderCreated",
  "version": 1,
  "occurredAt": "2026-10-26T12:00:00Z",
  "aggregateId": "CMD-42",
  "payload": {}
}
```

Le champ `eventId` sert à l'**idempotence** : chaque service enregistre les `eventId` déjà traités pour ignorer un événement reçu deux fois.

### Liste des événements

| Événement | Émetteur | Récepteurs | Donnée modifiée | Nouvelle valeur |
|---|---|---|---|---|
| `OrderCreated` | Commande | Stock, Livraison, Query | commande complète | lignes + adresse → **crée le réplica** dans Livraison |
| `StockReserved` | Stock | Commande, Livraison, Query | `status` de la commande | `CONFIRMED` |
| `StockRejected` | Stock | Commande, Query | `status` de la commande | `REJECTED` |
| `DeliveryAddressChanged` | Commande | Livraison, Query | `deliveryAddress` | nouvelle adresse → **met à jour le réplica** |
| `OrderShipped` | Livraison | Commande, Query | `status` de la commande | `SHIPPED` |

### Réplica de la Commande dans le service Livraison

Le service Livraison ne garde que les données de la commande dont il a besoin. Le réplica est **créé** par `OrderCreated` et **mis à jour** par `DeliveryAddressChanged` et `StockReserved`.

```json
{
  "orderId": "CMD-43",
  "deliveryAddress": "5 avenue Foch, 75016 Paris",
  "itemsCount": 2,
  "orderStatus": "CONFIRMED",
  "lastEventId": "uuid"
}
```

### Traitement fiable d'un événement (file ready / processing)

1. `LMOVE <service>:ready <service>:processing LEFT RIGHT` : le service prend l'événement, déplacé de façon atomique.
2. Traitement métier + enregistrement de l'`eventId`.
3. `LREM <service>:processing 1 <event>` : acquittement.
4. En cas de crash avant l'acquittement : `LMOVE <service>:processing <service>:ready RIGHT LEFT` remet l'événement en tête de file. Le contrôle de l'`eventId` évite un double traitement.

---

## Étape 5 – Projection

Le service Query consomme tous les événements et construit une **vue dénormalisée** prête à afficher (écran « Mes commandes »), sans interroger les autres services.

```json
{
  "orderId": "CMD-43",
  "customerId": "CLI-8",
  "total": 50,
  "status": "SHIPPED",
  "deliveryAddress": "5 avenue Foch, 75016 Paris",
  "carrier": "Colissimo",
  "updatedAt": "2026-10-26T14:00:00Z"
}
```

---

## Stockage des agrégats dans Redis

Consigne : stocker les 3 agrégats dans Redis, en ligne de commande, avec un Redis persistant.

### Choix de modélisation

Chaque agrégat est stocké dans un **Hash** : un objet plat dont les champs se lisent et se modifient indépendamment (`HGET`, `HMGET`, `HSET`, `HINCRBY`). Les lignes de commande étant une liste imbriquée, elles sont stockées dans un Hash séparé (champ = produit, valeur = quantité).

| Clé Redis | Type | Contenu |
|---|---|---|
| `shop:order:<orderId>` | Hash | en-tête de la commande |
| `shop:order:<orderId>:lines` | Hash | `productId` → quantité |
| `shop:stock:<productId>` | Hash | stock disponible et réservé |
| `shop:delivery:<deliveryId>` | Hash | livraison |

Toutes les clés sont préfixées par `shop:` (namespace du projet) et n'ont **pas de TTL** (`TTL` = `-1`) : ce sont des données métier qui ne doivent pas expirer.

### Connexion

```bash
docker compose -f docker-compose.dev.yml exec redis redis-cli
```

### 1. Agrégat Commande (CMD-42)

```redis
HSET shop:order:CMD-42 orderId CMD-42 customerId CLI-7 deliveryAddress "12 rue de Paris, 94120 Fontenay" total 250 status PENDING
# (integer) 5 → 5 champs créés
HSET shop:order:CMD-42:lines PROD-12 10
# (integer) 1
HMGET shop:order:CMD-42 status deliveryAddress
# 1) "PENDING"  2) "12 rue de Paris, 94120 Fontenay"
TYPE shop:order:CMD-42   # hash
HLEN shop:order:CMD-42   # (integer) 5
TTL shop:order:CMD-42    # (integer) -1 → persistant
```

### 2. Agrégat Stock (PROD-12)

```redis
HSET shop:stock:PROD-12 productId PROD-12 available 3 reserved 0
# (integer) 3
HGETALL shop:stock:PROD-12
# productId PROD-12 / available 3 / reserved 0
```

### 3. Agrégat Livraison (via la commande valide CMD-43)

```redis
# Commande valide de 2 articles
HSET shop:order:CMD-43 orderId CMD-43 customerId CLI-8 deliveryAddress "5 avenue Foch, 75016 Paris" total 50 status PENDING
HSET shop:order:CMD-43:lines PROD-12 2

# Réservation du stock : succès
HINCRBY shop:stock:PROD-12 available -2   # (integer) 1 → résultat positif = stock suffisant
HINCRBY shop:stock:PROD-12 reserved 2     # (integer) 2
HSET shop:order:CMD-43 status CONFIRMED   # (integer) 0 → champ existant mis à jour

# Création de la livraison
HSET shop:delivery:LIV-43 deliveryId LIV-43 orderId CMD-43 carrier "" status WAITING
# (integer) 4

# Expédition
HSET shop:delivery:LIV-43 carrier Colissimo status SHIPPED   # (integer) 0
HSET shop:order:CMD-43 status SHIPPED                        # (integer) 0
```

### État final

```redis
SCAN 0 MATCH shop:* COUNT 100
# shop:order:CMD-42, shop:order:CMD-42:lines,
# shop:order:CMD-43, shop:order:CMD-43:lines,
# shop:stock:PROD-12, shop:delivery:LIV-43

HGETALL shop:stock:PROD-12     # available 1 / reserved 2
HMGET shop:order:CMD-43 status # "SHIPPED"
HGETALL shop:delivery:LIV-43   # carrier Colissimo / status SHIPPED
```

> `HSET` renvoie le nombre de **champs créés**, pas le nombre de valeurs modifiées : d'où `(integer) 0` lors d'une mise à jour de statut.

---

## Gestion de la survente

Cas testé : la commande **CMD-42 demande 10 articles alors que seulement 3 sont disponibles**.

```redis
HINCRBY shop:stock:PROD-12 available -10   # (integer) -7 → négatif = stock insuffisant
HINCRBY shop:stock:PROD-12 available 10    # (integer) 3  → réservation annulée (compensation)
HSET shop:order:CMD-42 status REJECTED     # (integer) 0
```

`HINCRBY` est **atomique** : deux commandes simultanées ne peuvent pas réserver le même article, car chaque décrément est appliqué l'un après l'autre et renvoie la nouvelle valeur. Si le résultat est négatif, la réservation est annulée et la commande est refusée (`StockRejected`).

---

## Persistance de Redis

Configuration dans `docker-compose.dev.yml` :

```yaml
redis:
  image: redis:7-alpine
  ports:
    - "6380:6379"        # 6380 sur la machine (6379 déjà utilisé par le projet de l'enseignant)
  volumes:
    - redis-data:/data   # volume nommé : les fichiers de persistance survivent au conteneur
  command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy noeviction
```

| Option | Rôle |
|---|---|
| `--appendonly yes` | active l'**AOF** : chaque écriture est journalisée sur disque et rejouée au redémarrage |
| `redis-data:/data` | le journal est stocké dans un volume Docker nommé |
| `--maxmemory-policy noeviction` | Redis n'**évince aucune clé** sous pression mémoire : aucun agrégat ni aucune file d'événements n'est perdu |

Vérification :

```redis
CONFIG GET appendonly          # "yes"
CONFIG GET maxmemory-policy    # "noeviction"
```

### Test de redémarrage

```bash
docker compose -f docker-compose.dev.yml restart redis
docker compose -f docker-compose.dev.yml exec redis redis-cli SCAN 0 MATCH "shop:*" COUNT 100
```

Les 6 clés sont toujours présentes après le redémarrage : les données sont rejouées depuis l'AOF stocké dans le volume.

> `docker compose down -v` supprimerait le volume `redis-data` et donc toutes les données. Pour arrêter le projet sans perte : `docker compose down` (sans `-v`).

---

## Lancer le projet

Prérequis : Docker Desktop.

```bash
git clone <url-du-depot>
cd E-commerce-projet-Nosql-
docker compose -f docker-compose.dev.yml up -d     # démarre Redis et MongoDB
docker compose -f docker-compose.dev.yml ps        # shop-redis et shop-mongodb : healthy
docker compose -f docker-compose.dev.yml exec redis redis-cli PING   # PONG
```

---

## Arborescence

```
E-commerce-projet-Nosql-/
├── README.md                 # ce descriptif
├── docker-compose.dev.yml    # infrastructure : Redis + MongoDB
├── settings.gradle           # déclaration des services
├── docs/
│   └── events/               # contrats JSON des événements
├── orderService/             # service Commande
├── stockService/             # service Stock
├── deliveryService/          # service Livraison
├── orderQueryService/        # service Query (projection)
└── event-router/             # routeur d'événements Redis (fan-out)
```
