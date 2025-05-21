# Documentation Fonctionnelle - Module de Comptes Bancaires

Ce document détaille les fonctionnalités relatives à la gestion des comptes bancaires au sein du système ERP Banking.

## 1. Gestion des Comptes Bancaires

### 1.1. Création d'un Compte Bancaire

Permet de créer un nouveau compte bancaire dans le système.

*   **Endpoint :** `POST /api/v1/accounts`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR

**Exemple de requête :**

```json
{
  "bankId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "accountNumber": "FR7630004000031234567890143",
  "iban": "FR7630004000031234567890143",
  "currency": "EUR",
  "openingDate": "2023-01-15",
  "initialBalance": 1000.00,
  "friendlyName": "Compte Courant Principal"
}
```

**Exemple de réponse (Succès - 201 Created) :**

```json
{
  "id": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "bankId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "bank": { // Objet BankDto simplifié
    "id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
    "name": "Nom de la Banque" 
    // ... autres champs de BankDto
  },
  "accountNumber": "FR7630004000031234567890143",
  "iban": "FR7630004000031234567890143",
  "currency": "EUR",
  "openingDate": "2023-01-15",
  "initialBalance": 1000.00,
  "currentBalance": 1000.00,
  "status": "OUVERT", // Ou un statut initial par défaut
  "friendlyName": "Compte Courant Principal",
  "lastStatementImportDate": null,
  "createdAt": "2023-10-27T10:00:00Z",
  "updatedAt": "2023-10-27T10:00:00Z"
}
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` : Si les données fournies sont invalides (ex: `bankId` manquant, format de date incorrect) ou si un compte avec le même numéro existe déjà pour cette banque.
    ```json
    {
      "status": 400,
      "error": "Bad Request",
      "message": "Un compte avec le numéro 'FR7630004000031234567890143' existe déjà pour la banque 'Nom de la Banque'."
      // ou "Banque non trouvée avec l'ID: a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"
    }
    ```

### 1.2. Récupération des Informations d'un Compte Bancaire

Permet de récupérer les détails d'un compte bancaire spécifique.

*   **Endpoint :** `GET /api/v1/accounts/{accountId}`
*   **Rôles requis :** ADMIN, USER, DATA_PROCESSOR, VIEWER

**Exemple de requête :**

`GET /api/v1/accounts/c292f69f-9c96-4b71-962c-f6be7878063d`

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "bankId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "bank": {
    "id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
    "name": "Nom de la Banque"
  },
  "accountNumber": "FR7630004000031234567890143",
  "iban": "FR7630004000031234567890143",
  "currency": "EUR",
  "openingDate": "2023-01-15",
  "initialBalance": 1000.00,
  "currentBalance": 1250.50,
  "status": "OUVERT",
  "friendlyName": "Compte Courant Principal",
  "lastStatementImportDate": "2023-10-26T15:30:00Z",
  "createdAt": "2023-10-27T10:00:00Z",
  "updatedAt": "2023-10-27T10:05:00Z"
}
```

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si aucun compte ne correspond à l'`accountId` fourni.

### 1.3. Listage des Comptes Bancaires

Permet de récupérer une liste de tous les comptes bancaires, avec des options de filtrage.

*   **Endpoint :** `GET /api/v1/accounts`
*   **Rôles requis :** ADMIN, USER, DATA_PROCESSOR, VIEWER
*   **Paramètres de requête optionnels :**
    *   `bankId` (UUID) : Pour filtrer les comptes par ID de banque.
    *   `organizationId` (UUID) : Pour filtrer les comptes par ID d'organisation (via la banque).

**Exemple de requête (tous les comptes) :**

`GET /api/v1/accounts`

**Exemple de requête (filtré par `bankId`) :**

`GET /api/v1/accounts?bankId=a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11`

**Exemple de réponse (Succès - 200 OK) :**

```json
[
  {
    "id": "c292f69f-9c96-4b71-962c-f6be7878063d",
    // ... autres champs du compte
  },
  {
    "id": "d330a70a-1d83-4b87-8de6-02ab9487912e",
    // ... autres champs du compte
  }
]
```

### 1.4. Mise à Jour d'un Compte Bancaire

Permet de modifier les informations d'un compte bancaire existant.

*   **Endpoint :** `PUT /api/v1/accounts/{accountId}`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR

**Exemple de requête :**

`PUT /api/v1/accounts/c292f69f-9c96-4b71-962c-f6be7878063d`

```json
{
  "bankId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11", // Peut être le même ou un autre ID de banque
  "accountNumber": "FR7630004000031234567890143", // Attention si changement, vérifier unicité
  "iban": "FR7630004000031234567890143",
  "currency": "EUR",
  "openingDate": "2023-01-15", // Généralement non modifiable
  "initialBalance": 1000.00, // Généralement non modifiable
  "status": "ACTIF", // Exemple de changement de statut
  "friendlyName": "Compte Courant Principal (Modifié)"
}
```

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "bankId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "bank": {
    "id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
    "name": "Nom de la Banque"
  },
  "accountNumber": "FR7630004000031234567890143",
  "iban": "FR7630004000031234567890143",
  "currency": "EUR",
  "openingDate": "2023-01-15",
  "initialBalance": 1000.00,
  "currentBalance": 1250.50, // Le solde courant n'est pas modifié par cet endpoint
  "status": "ACTIF",
  "friendlyName": "Compte Courant Principal (Modifié)",
  "lastStatementImportDate": "2023-10-26T15:30:00Z",
  "createdAt": "2023-10-27T10:00:00Z",
  "updatedAt": "2023-10-27T11:00:00Z"
}
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` : Si les données fournies sont invalides ou si la mise à jour crée un conflit (ex: numéro de compte dupliqué pour la même banque).
*   `404 Not Found` : Si aucun compte ne correspond à l'`accountId` fourni.

### 1.5. Suppression d'un Compte Bancaire

Permet de supprimer un compte bancaire. (Attention : vérifier les implications, comme la présence de relevés liés).

*   **Endpoint :** `DELETE /api/v1/accounts/{accountId}`
*   **Rôles requis :** ADMIN

**Exemple de requête :**

`DELETE /api/v1/accounts/c292f69f-9c96-4b71-962c-f6be7878063d`

**Exemple de réponse (Succès - 204 No Content) :**

(Aucun contenu dans la réponse)

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si aucun compte ne correspond à l'`accountId` fourni.

## 2. Consultation des Soldes

Le solde actuel (`currentBalance`) et le solde initial (`initialBalance`) sont disponibles lors de la récupération des informations d'un compte bancaire (voir section 1.2). Il n'y a pas d'endpoint dédié uniquement au solde.

## 3. Relevés de Compte (Transactions)

### 3.1. Récupération des Relevés d'un Compte

Permet de lister les relevés (transactions) d'un compte bancaire spécifique, avec pagination.

*   **Endpoint :** `GET /api/v1/accounts/{accountId}/statements`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR, USER, VIEWER
*   **Paramètres de requête optionnels :**
    *   `status` (String) : Pour filtrer les relevés par statut (ex: `PENDING_REVIEW`, `CONFIRMED`).
    *   `page` (int) : Numéro de la page (défaut: 0).
    *   `size` (int) : Nombre d'éléments par page (défaut: 20).
    *   `sort` (String) : Propriété de tri (ex: `operationDate,desc`).

**Exemple de requête (relevés confirmés, page 0, 10 éléments par page) :**

`GET /api/v1/accounts/c292f69f-9c96-4b71-962c-f6be7878063d/statements?status=CONFIRMED&page=0&size=10&sort=operationDate,desc`

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "content": [
    {
      "id": "e116a71b-1d82-4b86-8de5-02ab9487912a",
      "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
      "accountNumber": "FR7630004000031234567890143",
      "bankName": "Nom de la Banque",
      "importJobId": "f010a61a-1c80-4a85-8da4-01ab9386801a",
      "operationDate": "2023-10-26",
      "valueDate": "2023-10-26",
      "description": "Virement entrant de John Doe",
      "amount": 500.00,
      "currency": "EUR",
      "operationType": "CREDIT",
      "balanceAfter": 1750.50,
      "externalReference": "VIR00123",
      "internalStatus": "CONFIRMED",
      "confirmedByUserId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
      "confirmedByUsername": "admin@example.com",
      "confirmedAt": "2023-10-27T12:00:00Z",
      "notes": "Relevé confirmé",
      "createdAt": "2023-10-27T11:30:00Z",
      "updatedAt": "2023-10-27T12:00:00Z"
    }
    // ... autres relevés
  ],
  "pageable": {
    "sort": {
      "sorted": true,
      "unsorted": false,
      "empty": false
    },
    "offset": 0,
    "pageNumber": 0,
    "pageSize": 10,
    "paged": true,
    "unpaged": false
  },
  "last": false,
  "totalPages": 5,
  "totalElements": 48,
  "size": 10,
  "number": 0,
  "sort": {
    "sorted": true,
    "unsorted": false,
    "empty": false
  },
  "first": true,
  "numberOfElements": 10,
  "empty": false
}
```

### 3.2. Mise à Jour des Détails d'un Relevé Spécifique

Permet de modifier les détails d'un relevé avant sa confirmation. Un relevé confirmé ne peut pas être modifié directement par cet endpoint.

*   **Endpoint :** `PUT /api/v1/accounts/{accountId}/statements/{statementId}`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR

**Exemple de requête :**

`PUT /api/v1/accounts/c292f69f-9c96-4b71-962c-f6be7878063d/statements/e116a71b-1d82-4b86-8de5-02ab9487912a`

```json
{
  // accountId et statementId sont dans l'URL
  // Les champs ci-dessous sont ceux qui peuvent être modifiés
  "operationDate": "2023-10-25", // Date d'opération modifiée
  "valueDate": "2023-10-25",     // Date de valeur modifiée
  "description": "Paiement fournisseur XYZ - Corrigé",
  "amount": 250.00,
  "currency": "EUR",
  "operationType": "DEBIT", // Type d'opération peut être corrigé
  // "balanceAfter": 1500.50, // Le solde après opération n'est généralement pas fourni ici, mais recalculé
  "externalReference": "INV-2023-10-555",
  "notes": "Correction de la description et de la date."
  // Les champs comme internalStatus, confirmedBy*, etc., ne sont pas modifiés ici.
}
```

**Exemple de réponse (Succès - 200 OK) :**

```json
{
  "id": "e116a71b-1d82-4b86-8de5-02ab9487912a",
  "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
  "accountNumber": "FR7630004000031234567890143",
  "bankName": "Nom de la Banque",
  "importJobId": "f010a61a-1c80-4a85-8da4-01ab9386801a",
  "operationDate": "2023-10-25",
  "valueDate": "2023-10-25",
  "description": "Paiement fournisseur XYZ - Corrigé",
  "amount": 250.00,
  "currency": "EUR",
  "operationType": "DEBIT",
  "balanceAfter": null, // Ou la valeur recalculée si applicable
  "externalReference": "INV-2023-10-555",
  "internalStatus": "PENDING_REVIEW", // Le statut reste inchangé par cet appel
  "confirmedByUserId": null,
  "confirmedByUsername": null,
  "confirmedAt": null,
  "notes": "Correction de la description et de la date.",
  "createdAt": "2023-10-27T11:30:00Z",
  "updatedAt": "2023-10-27T12:15:00Z"
}
```

**Réponses d'erreur possibles :**

*   `404 Not Found` : Si le compte ou le relevé n'existe pas.
*   `409 Conflict` ou `IllegalStateException` (traduit en `409` ou `400` par le `ControllerAdvice`): Si le relevé est déjà confirmé.
    ```json
    {
      "status": 409,
      "error": "Conflict",
      "message": "Impossible de modifier les détails d'un relevé déjà confirmé. Changez d'abord son statut."
    }
    ```

## 4. Processus de Révision et de Mise à Jour des Relevés

### 4.1. Mise à Jour en Masse du Statut des Relevés

Permet de changer le statut de plusieurs relevés en une seule opération (ex: passer de `PENDING_REVIEW` à `CONFIRMED` ou `REJECTED`). La confirmation d'un relevé met à jour le solde du compte associé. L'annulation d'une confirmation (passer de `CONFIRMED` à un autre statut) ajuste également le solde en conséquence.

*   **Endpoint :** `POST /api/v1/accounts/{accountId}/statements/batch-review`
*   **Rôles requis :** ADMIN, DATA_PROCESSOR

**Exemple de requête :**

`POST /api/v1/accounts/c292f69f-9c96-4b71-962c-f6be7878063d/statements/batch-review`

```json
{
  "statementIds": [
    "e116a71b-1d82-4b86-8de5-02ab9487912a",
    "f227b82c-2e93-5c97-9ef6-03bc0598023b"
  ],
  "newStatus": "CONFIRMED", // Peut être CONFIRMED, REJECTED, REVIEWED, PENDING_REVIEW
  "notes": "Relevés vérifiés et confirmés par l'utilisateur."
}
```

**Exemple de réponse (Succès - 200 OK) :**

```json
[
  {
    "id": "e116a71b-1d82-4b86-8de5-02ab9487912a",
    "accountId": "c292f69f-9c96-4b71-962c-f6be7878063d",
    // ... autres champs du relevé, avec internalStatus mis à jour, confirmedBy*, confirmedAt* remplis
    "internalStatus": "CONFIRMED",
    "confirmedByUserId": "a1b2c3d4-e5f6-7890-1234-567890abcdef", // ID de l'utilisateur effectuant l'action
    "confirmedByUsername": "user@example.com", // Email/username de l'utilisateur
    "confirmedAt": "2023-10-27T14:30:00Z",
    "notes": "Relevés vérifiés et confirmés par l'utilisateur."
  },
  {
    "id": "f227b82c-2e93-5c97-9ef6-03bc0598023b",
    // ... autres champs
    "internalStatus": "CONFIRMED",
    "confirmedByUserId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "confirmedByUsername": "user@example.com",
    "confirmedAt": "2023-10-27T14:30:00Z",
    "notes": "Relevés vérifiés et confirmés par l'utilisateur."
  }
]
```

**Réponses d'erreur possibles :**

*   `400 Bad Request` : Si le `newStatus` est invalide ou si la liste `statementIds` est vide/nulle.
    ```json
    {
        "status": 400,
        "error": "Bad Request",
        "message": "Statut de mise à jour invalide: INVALID_STATUS"
    }
    ```
*   `404 Not Found` : Si l'un des `statementIds` ne correspond à aucun relevé.
    ```json
    {
        "status": 404,
        "error": "Not Found",
        "message": "Relevé non trouvé avec l'ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
    ```
*   `IllegalStateException` (traduit en `400` ou `500` par le `ControllerAdvice`): Si l'utilisateur effectuant l'action ne peut être déterminé.

## 5. Import de Relevés (Fonctionnalité Connexe)

Bien que la logique d'import et de parsing des fichiers de relevés soit gérée par un autre module (`statementimport`), le service `AccountStatementStorageService` est utilisé par ce processus pour stocker les transactions parsées en tant qu'entités `AccountStatementEntity`.

Lorsqu'un fichier de relevé est traité :
1.  Les transactions sont extraites.
2.  Pour chaque transaction, une `AccountStatementEntity` est créée avec le statut initial `PENDING_REVIEW`.
3.  La date `lastStatementImportDate` sur l'entité `AccountEntity` associée est mise à jour.

Ces relevés nouvellement importés apparaissent ensuite via l'endpoint `GET /api/v1/accounts/{accountId}/statements` (souvent filtrés par `status=PENDING_REVIEW`) pour la révision.
