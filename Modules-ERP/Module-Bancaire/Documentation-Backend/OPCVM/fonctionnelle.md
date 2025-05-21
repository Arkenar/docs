# Documentation Fonctionnelle - Module de Gestion des OPCVM (Fonds d'Investissement)

Ce document détaille les fonctionnalités relatives à la gestion des détentions d'Organismes de Placement Collectif en Valeurs Mobilières (OPCVM) ou fonds d'investissement au sein du système ERP Banking.

## 1. Consultation des Départs d'OPCVM / Détentions d'OPCVM

Permet de visualiser les fonds d'investissement actuellement détenus.

### 1.1. Lister Tous les Fonds Détenus

Permet de récupérer une liste de tous les fonds OPCVM détenus, avec la possibilité de filtrer par compte bancaire de rattachement.

*   **Endpoint :** `GET /api/v1/opcvm-holdings`
*   **Rôles requis :** ADMIN, USER, DATA_PROCESSOR, VIEWER
*   **Paramètres de requête optionnels :**

    | Paramètre   | Type | Description                                                                |
        |-------------|------|----------------------------------------------------------------------------|
    | `accountId` | UUID | ID du compte bancaire pour filtrer les détentions d'OPCVM associées à ce compte. |

**Exemple de requête (tous les fonds) :**

`GET /api/v1/opcvm-holdings`

**Exemple de requête (filtré par `accountId`) :**

`GET /api/v1/opcvm-holdings?accountId=c292f69f-9c96-4b71-962c-f6be7878063d`

**Exemple de réponse (Succès - 200 OK) :**

Chaque élément de la liste est un `OpcvmHoldingDto`.

```json
[
  {
    "id": "e116a71b-1d82-4b86-8de5-02ab9487912a",
    "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
    "name": "Fonds Actions Europe ISR",
    "isinCode": "FR0012345678",
    "quantity": 150.75,
    "unitPrice": 120.50,
    "totalValue": 18165.3750, // quantity * unitPrice
    "currency": "EUR",
    "valuationDate": "2023-10-27"
  },
  {
    "id": "f227b82c-2e93-5c97-9ef6-03bc0598023b",
    "accountId": null, // Peut être un OPCVM non directement rattaché à un compte spécifique
    "name": "Obligations Internationales Diversifiées",
    "isinCode": "LU0098765432",
    "quantity": 50.0,
    "unitPrice": 95.20,
    "totalValue": 4760.00,
    "currency": "EUR",
    "valuationDate": "2023-10-27"
  }
]
```

### 1.2. Obtenir un Fond Détenu Spécifique par son ID

Permet de récupérer les détails d'une détention d'OPCVM spécifique en utilisant son identifiant unique.

*   **Endpoint :** `GET /api/v1/opcvm-holdings/{holdingId}`
*   **Rôles requis :** ADMIN, USER, DATA_PROCESSOR, VIEWER
*   **Paramètre de chemin :**

    | Paramètre   | Type | Description                                  |
        |-------------|------|----------------------------------------------|
    | `holdingId` | UUID | ID unique de la détention d'OPCVM à récupérer. |

**Exemple de requête :**

`GET /api/v1/opcvm-holdings/e116a71b-1d82-4b86-8de5-02ab9487912a`

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "e116a71b-1d82-4b86-8de5-02ab9487912a",
  "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "name": "Fonds Actions Europe ISR",
  "isinCode": "FR0012345678",
  "quantity": 150.75,
  "unitPrice": 120.50,
  "totalValue": 18165.3750,
  "currency": "EUR",
  "valuationDate": "2023-10-27"
}
```

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si aucune détention d'OPCVM ne correspond à l'`holdingId` fourni.

## 2. Gestion des Départs d'OPCVM / Détentions d'OPCVM

Permet d'ajouter, de mettre à jour ou de supprimer des lignes de détention d'OPCVM.

### 2.1. Créer une Nouvelle Ligne de Détention d'OPCVM

Permet d'enregistrer une nouvelle détention de parts d'un fonds d'investissement.

*   **Endpoint :** `POST /api/v1/opcvm-holdings`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR
*   **Corps de la requête :** `OpcvmHoldingDto` (sans `id` ni `totalValue` qui sont générés/calculés par le serveur)

    | Champ           | Type         | Obligatoire | Description                                                                    |
        |-----------------|--------------|-------------|--------------------------------------------------------------------------------|
    | `accountId`     | UUID         | Non         | ID du compte bancaire auquel cette détention est rattachée.                      |
    | `name`          | String       | Oui         | Nom du fonds OPCVM (max 255 caractères).                                        |
    | `isinCode`      | String       | Non         | Code ISIN du fonds (max 12 caractères). Doit être unique par `accountId` si fourni. |
    | `quantity`      | BigDecimal   | Oui         | Nombre de parts détenues (doit être positif).                                    |
    | `unitPrice`     | BigDecimal   | Oui         | Valeur liquidative (VL) ou prix unitaire de la part (positif ou nul).            |
    | `currency`      | String       | Oui         | Code ISO de la devise de valorisation (3 lettres).                               |
    | `valuationDate` | LocalDate    | Oui         | Date à laquelle la quantité et le prix unitaire sont valides.                    |

**Exemple de requête :**

```json
{
  "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "name": "Nouveau Fonds Monétaire Dynamique",
  "isinCode": "FR0054321098",
  "quantity": 200.0,
  "unitPrice": 105.75,
  "currency": "EUR",
  "valuationDate": "2023-10-28"
}
```

**Exemple de réponse (Succès - 201 Created) :**

```json
{
  "id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11", // ID généré par le système
  "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "name": "Nouveau Fonds Monétaire Dynamique",
  "isinCode": "FR0054321098",
  "quantity": 200.0,
  "unitPrice": 105.75,
  "totalValue": 21150.00, // Calculé par le système
  "currency": "EUR",
  "valuationDate": "2023-10-28"
}
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` :
    *   Si les données fournies sont invalides (ex: `name` manquant, `quantity` négative).
    *   Si l'`accountId` fourni n'existe pas : `"Compte bancaire associé non trouvé: <accountId>"`
*   `409 Conflict` : Si une détention avec le même `isinCode` existe déjà pour le `accountId` spécifié (ou en tant que titre générique si `accountId` est nul) : `"Un OPCVM avec l'ISIN '<isinCode>' existe déjà pour ce compte."` / `"Un OPCVM avec l'ISIN '<isinCode>' existe déjà en tant que titre générique."`

### 2.2. Mettre à Jour une Ligne de Détention d'OPCVM

Permet de modifier les informations d'une détention d'OPCVM existante.

*   **Endpoint :** `PUT /api/v1/opcvm-holdings/{holdingId}`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR
*   **Paramètre de chemin :**

    | Paramètre   | Type | Description                                     |
        |-------------|------|-------------------------------------------------|
    | `holdingId` | UUID | ID de la détention d'OPCVM à mettre à jour.     |
*   **Corps de la requête :** `OpcvmHoldingDto` (les champs à mettre à jour)

**Exemple de requête (mise à jour de la quantité et du prix unitaire) :**

`PUT /api/v1/opcvm-holdings/e116a71b-1d82-4b86-8de5-02ab9487912a`

```json
{
  "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "name": "Fonds Actions Europe ISR", // Le nom peut aussi être modifié
  "isinCode": "FR0012345678",
  "quantity": 175.0, // Nouvelle quantité
  "unitPrice": 122.30, // Nouveau prix
  "currency": "EUR",
  "valuationDate": "2023-10-29" // Nouvelle date de valorisation
}
```

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "e116a71b-1d82-4b86-8de5-02ab9487912a",
  "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "name": "Fonds Actions Europe ISR",
  "isinCode": "FR0012345678",
  "quantity": 175.0,
  "unitPrice": 122.30,
  "totalValue": 21402.50, // Recalculé
  "currency": "EUR",
  "valuationDate": "2023-10-29"
}
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` :
    *   Si les données fournies sont invalides.
    *   Si le `accountId` (nouveau ou modifié) fourni n'existe pas : `"Nouveau compte bancaire associé non trouvé: <accountId>"`
*   `404 Not Found` : Si aucune détention d'OPCVM ne correspond à l'`holdingId` fourni.
*   `409 Conflict` : Si la modification de `isinCode` et/ou `accountId` crée un doublon.

### 2.3. Supprimer une Ligne de Détention d'OPCVM

Permet de supprimer une ligne de détention d'OPCVM.

*   **Endpoint :** `DELETE /api/v1/opcvm-holdings/{holdingId}`
*   **Rôles requis :** ADMIN
*   **Paramètre de chemin :**

    | Paramètre   | Type | Description                                    |
        |-------------|------|------------------------------------------------|
    | `holdingId` | UUID | ID de la détention d'OPCVM à supprimer.        |

**Exemple de requête :**

`DELETE /api/v1/opcvm-holdings/e116a71b-1d82-4b86-8de5-02ab9487912a`

**Exemple de réponse (Succès - 204 No Content) :**

(Aucun contenu dans la réponse)

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si aucune détention d'OPCVM ne correspond à l'`holdingId` fourni.

## 3. Données Historiques

La structure actuelle se concentre sur la dernière valorisation connue d'une détention (`valuationDate`, `quantity`, `unitPrice`). Il n'y a pas de fonctionnalité explicite pour récupérer l'historique des valorisations d'une même ligne de détention d'OPCVM (par exemple, l'évolution de la quantité ou du prix d'un fonds spécifique au fil du temps). Chaque mise à jour écrase les valeurs précédentes de quantité et de prix pour la `valuationDate` donnée. Pour un suivi historique, il faudrait enregistrer des "snapshots" de valorisation à différentes dates, ce qui n'est pas géré par le modèle actuel.
