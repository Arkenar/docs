# Documentation Fonctionnelle - Module des Taux de Change

Ce document détaille les fonctionnalités relatives à la gestion et à la consultation des taux de change entre devises.

## 1. Consultation des Taux de Change

### 1.1. Obtenir un Taux de Change Spécifique

Permet de récupérer le taux de change entre deux devises à une date donnée.

*   **Endpoint :** `GET /api/v1/exchange-rates`
*   **Rôles requis :** ADMIN, USER, DATA_PROCESSOR, VIEWER
*   **Paramètres de requête :**

    | Paramètre      | Type        | Obligatoire | Description                                                     |
        |----------------|-------------|-------------|-----------------------------------------------------------------|
    | `fromCurrency` | String      | Oui         | Code ISO à 3 lettres de la devise source (ex: USD).             |
    | `toCurrency`   | String      | Oui         | Code ISO à 3 lettres de la devise cible (ex: EUR).                |
    | `date`         | LocalDate   | Oui         | Date pour laquelle le taux est demandé (format ISO : AAAA-MM-JJ). |

**Exemple de requête :**

`GET /api/v1/exchange-rates?fromCurrency=USD&toCurrency=EUR&date=2023-10-27`

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "fromCurrency": "USD",
  "toCurrency": "EUR",
  "rate": 0.95, // 1 USD = 0.95 EUR
  "date": "2023-10-27"
}
```

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si aucun taux n'est trouvé pour la combinaison devise source/cible et date spécifiée.

### 1.2. Obtenir tous les Taux de Change pour une Date Donnée

Permet de récupérer une liste de tous les taux de change enregistrés pour une date spécifique.

*   **Endpoint :** `GET /api/v1/exchange-rates/by-date`
*   **Rôles requis :** ADMIN, USER, DATA_PROCESSOR, VIEWER
*   **Paramètres de requête :**

    | Paramètre | Type        | Obligatoire | Description                                                        |
        |-----------|-------------|-------------|--------------------------------------------------------------------|
    | `date`    | LocalDate   | Oui         | Date pour laquelle les taux sont demandés (format ISO : AAAA-MM-JJ). |

**Exemple de requête :**

`GET /api/v1/exchange-rates/by-date?date=2023-10-27`

**Exemple de réponse (Succès - 200 OK) :**

```json
[
  {
    "id": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "fromCurrency": "USD",
    "toCurrency": "EUR",
    "rate": 0.95,
    "date": "2023-10-27"
  },
  {
    "id": "b2c3d4e5-f6g7-8901-2345-678901abcdef",
    "fromCurrency": "GBP",
    "toCurrency": "EUR",
    "rate": 1.15,
    "date": "2023-10-27"
  }
  // ... autres taux pour cette date
]
```

## 2. Gestion des Taux de Change

Ces fonctionnalités permettent de créer, mettre à jour ou supprimer les taux de change dans le système.

### 2.1. Créer ou Mettre à Jour un Taux de Change

Permet d'ajouter un nouveau taux de change pour une paire de devises à une date donnée, ou de mettre à jour un taux existant. L'opération est idempotente : si un taux existe déjà pour la même combinaison `fromCurrency`, `toCurrency` et `date`, il sera mis à jour ; sinon, un nouveau taux sera créé.

*   **Endpoint :** `POST /api/v1/exchange-rates`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR
*   **Corps de la requête :** `ExchangeRateDto`

**Exemple de requête (Création) :**

```json
{
  "fromCurrency": "CAD",
  "toCurrency": "EUR",
  "rate": 0.70,
  "date": "2023-10-28"
}
```

**Exemple de requête (Mise à jour d'un taux existant pour CAD/EUR à la date 2023-10-28) :**

```json
{
  "fromCurrency": "CAD",
  "toCurrency": "EUR",
  "rate": 0.71, // Nouveau taux
  "date": "2023-10-28"
}
```

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "c3d4e5f6-g7h8-9012-3456-789012abcdef", // ID généré pour une nouvelle création ou ID existant pour une mise à jour
  "fromCurrency": "CAD",
  "toCurrency": "EUR",
  "rate": 0.71,
  "date": "2023-10-28"
}
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` : Si les données fournies sont invalides (ex: devise non conforme au format 3 lettres, taux non positif, date manquante).
*   `500 Internal Server Error` : En cas d'erreur inattendue lors de la sauvegarde.

### 2.2. Créer ou Mettre à Jour Plusieurs Taux de Change en Lot (Batch)

Permet d'ajouter ou de mettre à jour plusieurs taux de change en une seule requête. Chaque taux dans la liste suit la même logique d'idempotence que la création/mise à jour unitaire.

*   **Endpoint :** `POST /api/v1/exchange-rates/batch`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR
*   **Corps de la requête :** `List<ExchangeRateDto>`

**Exemple de requête :**

```json
[
  {
    "fromCurrency": "AUD",
    "toCurrency": "USD",
    "rate": 0.65,
    "date": "2023-10-28"
  },
  {
    "fromCurrency": "CHF",
    "toCurrency": "EUR",
    "rate": 1.02,
    "date": "2023-10-28"
  }
]
```

**Exemple de réponse (Succès - 200 OK) :**

```json
[
  {
    "id": "d4e5f6g7-h8i9-0123-4567-890123abcdef",
    "fromCurrency": "AUD",
    "toCurrency": "USD",
    "rate": 0.65,
    "date": "2023-10-28"
  },
  {
    "id": "e5f6g7h8-i9j0-1234-5678-901234abcdef",
    "fromCurrency": "CHF",
    "toCurrency": "EUR",
    "rate": 1.02,
    "date": "2023-10-28"
  }
]
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` : Si l'un des objets `ExchangeRateDto` dans la liste est invalide.
*   `500 Internal Server Error` : En cas d'erreur inattendue lors de la sauvegarde.

### 2.3. Supprimer un Taux de Change

Permet de supprimer un taux de change spécifique par son ID.

*   **Endpoint :** `DELETE /api/v1/exchange-rates/{id}`
*   **Rôles requis :** ADMIN
*   **Paramètre de chemin :**

    | Paramètre | Type | Obligatoire | Description                         |
        |-----------|------|-------------|-------------------------------------|
    | `id`      | UUID | Oui         | ID du taux de change à supprimer.   |

**Exemple de requête :**

`DELETE /api/v1/exchange-rates/c3d4e5f6-g7h8-9012-3456-789012abcdef`

**Exemple de réponse (Succès - 204 No Content) :**

(Aucun contenu dans la réponse)

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si aucun taux de change ne correspond à l'`id` fourni.

## 3. Source des Données de Taux de Change

Le système permet la saisie manuelle des taux de change via les endpoints décrits ci-dessus.
Une fonctionnalité de récupération automatique des taux depuis une API externe est envisagée (marquée comme TODO dans le code source du service), mais n'est pas activement implémentée dans la version actuelle. Si implémentée, elle permettrait de mettre à jour périodiquement les taux de manière automatisée.
