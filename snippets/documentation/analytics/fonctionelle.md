# Documentation Fonctionnelle - Module d'Analytique Financière

Ce document détaille les fonctionnalités du module d'analytique financière, permettant d'obtenir des rapports et des aperçus sur les données bancaires.

## 1. Historique des Soldes de Compte

Permet de visualiser l'évolution du solde d'un compte bancaire sur une période donnée.

*   **Endpoint :** `GET /api/v1/analytics/accounts/{accountId}/balance-history`
*   **Rôles requis :** ADMIN, USER, VIEWER, DATA_PROCESSOR
*   **Paramètres de requête :**

    | Paramètre     | Type        | Obligatoire | Description                                                                 | Défaut         |
        |---------------|-------------|-------------|-----------------------------------------------------------------------------|----------------|
    | `accountId`   | UUID        | Oui         | ID du compte bancaire pour lequel générer l'historique.                     |                |
    | `fromDate`    | LocalDate   | Oui         | Date de début de la période (format ISO : AAAA-MM-JJ).                      |                |
    | `toDate`      | LocalDate   | Oui         | Date de fin de la période (format ISO : AAAA-MM-JJ).                        |                |
    | `granularity` | String      | Non         | Granularité de l'historique : `daily` (quotidienne), `monthly` (mensuelle). | `daily`        |

**Exemple de requête :**

`GET /api/v1/analytics/accounts/c292f69f-9c96-4b71-962c-f6be7878063d/balance-history?fromDate=2023-01-01&toDate=2023-01-31&granularity=daily`

**Exemple de réponse (Succès - 200 OK) :**

```json
[
  {
    "date": "2023-01-01",
    "balance": 1000.00
  },
  {
    "date": "2023-01-02",
    "balance": 1050.50
  },
  // ... autres points de l'historique
  {
    "date": "2023-01-31",
    "balance": 1250.75
  }
]
```

**Note :** L'implémentation actuelle de cette fonctionnalité est un placeholder. Elle retournera une liste vide. Une logique plus robuste basée sur les transactions confirmées est nécessaire pour un calcul précis de l'historique.

## 2. Rapport des Flux de Trésorerie (Cash Flow)

Génère un rapport analysant les entrées et sorties de fonds sur une période donnée, potentiellement pour plusieurs comptes ou banques, avec options de regroupement et de ventilation.

*   **Endpoint :** `GET /api/v1/analytics/cashflow`
*   **Rôles requis :** ADMIN, USER, VIEWER, DATA_PROCESSOR
*   **Paramètres de requête :**

    | Paramètre       | Type          | Obligatoire | Description                                                                                                | Défaut     |
        |-----------------|---------------|-------------|------------------------------------------------------------------------------------------------------------|------------|
    | `accountIds`    | List<UUID>    | Non         | Liste des ID de comptes à inclure dans le rapport. Si omis, tous les comptes pertinents peuvent être inclus. |            |
    | `bankIds`       | List<UUID>    | Non         | Liste des ID de banques à inclure.                                                                         |            |
    | `fromDate`      | LocalDate     | Oui         | Date de début de la période (format ISO : AAAA-MM-JJ).                                                       |            |
    | `toDate`        | LocalDate     | Oui         | Date de fin de la période (format ISO : AAAA-MM-JJ).                                                         |            |
    | `groupByPeriod` | String        | Non         | Période de regroupement des flux : `monthly` (mensuel), `quarterly` (trimestriel), `yearly` (annuel).       | `monthly`  |
    | `breakdownBy`   | String        | Non         | Critère de ventilation des flux : `account` (par compte), `bank` (par banque).                               |            |

**Exemple de requête (flux mensuels pour des comptes spécifiques, ventilés par compte) :**

`GET /api/v1/analytics/cashflow?accountIds=c292f69f-9c96-4b71-962c-f6be7878063d,d330a70a-1d83-4b87-8de6-02ab9487912e&fromDate=2023-01-01&toDate=2023-03-31&groupByPeriod=monthly&breakdownBy=account`

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "periodType": "monthly", // Ou la valeur de groupByPeriod
  "period": "2023-01",     // Exemple si groupByPeriod est monthly, pourrait être une structure plus complexe
  "totalInflows": 5000.00,
  "totalOutflows": 3500.00,
  "netChange": 1500.00,
  "breakdown": {
    "c292f69f-9c96-4b71-962c-f6be7878063d": {
      "groupId": "c292f69f-9c96-4b71-962c-f6be7878063d",
      "groupName": "Compte Courant Principal", // Nom du compte
      "inflows": 3000.00,
      "outflows": 1500.00,
      "netChange": 1500.00
    },
    "d330a70a-1d83-4b87-8de6-02ab9487912e": {
      "groupId": "d330a70a-1d83-4b87-8de6-02ab9487912e",
      "groupName": "Compte d'Épargne", // Nom du compte
      "inflows": 2000.00,
      "outflows": 2000.00,
      "netChange": 0.00
    }
  }
}
```

**Note :** L'implémentation actuelle de cette fonctionnalité est un placeholder. Elle retournera des valeurs par défaut (ZERO pour les montants, map vide pour `breakdown`). Une logique d'agrégation complexe basée sur les transactions confirmées est requise.

## 3. Solde Consolidé

Calcule le solde total de tous les comptes bancaires, converti dans une devise cible unique, à une date donnée. Utilise les taux de change disponibles pour la conversion.

*   **Endpoint :** `GET /api/v1/analytics/consolidated-balance`
*   **Rôles requis :** ADMIN, USER, VIEWER, DATA_PROCESSOR
*   **Paramètres de requête :**

    | Paramètre        | Type        | Obligatoire | Description                                                                 | Défaut      |
        |------------------|-------------|-------------|-----------------------------------------------------------------------------|-------------|
    | `targetCurrency` | String      | Oui         | Code de la devise cible pour la consolidation (ex: EUR, USD).               |             |
    | `date`           | LocalDate   | Non         | Date pour laquelle effectuer la consolidation (format ISO : AAAA-MM-JJ).    | Date du jour |

**Exemple de requête (solde consolidé en EUR à la date du 2023-10-27) :**

`GET /api/v1/analytics/consolidated-balance?targetCurrency=EUR&date=2023-10-27`

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "targetCurrency": "EUR",
  "totalBalanceInTargetCurrency": 150325.75,
  "asOfDate": "2023-10-27",
  "balancesInOriginalCurrency": {
    "EUR": 120000.00,
    "USD": 25000.00,  // Supposant un taux de change USD -> EUR
    "GBP": 5000.00    // Supposant un taux de change GBP -> EUR
  }
}
```

**Comportement en cas de taux de change manquants :**
Si un taux de change nécessaire pour la conversion d'une devise d'un compte vers la devise cible n'est pas trouvé pour la date spécifiée, le solde de ce compte ne sera pas inclus dans le `totalBalanceInTargetCurrency`. Un avertissement sera logué côté serveur. Le champ `balancesInOriginalCurrency` listera toujours les soldes dans leurs devises d'origine.

Si aucun taux de change n'est trouvé pour aucun des comptes nécessitant une conversion, le `totalBalanceInTargetCurrency` ne reflétera que la somme des soldes des comptes déjà dans la devise cible.
