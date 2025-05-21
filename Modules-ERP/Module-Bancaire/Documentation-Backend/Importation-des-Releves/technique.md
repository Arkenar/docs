# Documentation Technique - Module d'Importation de Relevés Bancaires

Ce document décrit l'architecture technique et les composants du module `statementimport`, responsable de l'importation et du traitement des fichiers de relevés bancaires.

## 1. Vue d'Ensemble de l'Architecture

Le module `statementimport` est conçu pour permettre une importation flexible des relevés bancaires en s'appuyant sur des modèles (templates) configurables. Il interagit principalement avec le module `account` pour stocker les transactions parsées.

L'architecture du module comprend :

*   **Contrôleurs (Controllers) :**
    *   `ImportTemplateController` : Gère les API REST pour les opérations CRUD sur les modèles d'importation.
    *   `StatementImportController` : Gère l'API REST pour le téléversement et le traitement des fichiers de relevés.
*   **Services :**
    *   `ImportTemplateService` : Contient la logique métier pour la gestion des modèles d'importation.
    *   `StatementProcessingService` : Contient la logique de parsing des fichiers de relevés en utilisant les modèles.
*   **Entités (Entities) :**
    *   `ImportTemplateEntity` : Modèle JPA pour un modèle d'importation.
    *   `TemplateFieldMappingEntity` : Modèle JPA pour le mapping d'un champ spécifique au sein d'un modèle.
*   **DTOs (Data Transfer Objects) :** Utilisés pour la communication API (création/lecture de modèles, réponse de téléversement, données parsées).
*   **Mappers :** Convertissent les Entités en DTOs et vice-versa (MapStruct).
*   **Référentiels (Repositories) :** Interfaces Spring Data JPA pour l'accès aux données des modèles.
*   **Domaine (Domain) :**
    *   `SourceType` (Enum) : Définit les types de fichiers sources supportés (EXCEL, CSV, etc.).

## 2. Composants Détaillés

### 2.1. Contrôleurs

#### 2.1.1. `ImportTemplateController` (Package : `...controller`)

*   **Rôle :** Expose les API REST pour la gestion des modèles d'importation.
*   **Endpoints :**
    *   `GET /api/v1/import-templates` : Liste les modèles (filtre optionnel `bankId`). Autorisation : `ADMIN, DATA_PROCESSOR`.
    *   `GET /api/v1/import-templates/{templateId}` : Récupère un modèle par ID. Autorisation : `ADMIN, DATA_PROCESSOR`.
    *   `POST /api/v1/import-templates` : Crée un modèle. Autorisation : `ADMIN`.
    *   `PUT /api/v1/import-templates/{templateId}` : Met à jour un modèle. Autorisation : `ADMIN`.
    *   `DELETE /api/v1/import-templates/{templateId}` : Supprime un modèle. Autorisation : `ADMIN`.
*   **Gestion des Erreurs :** Utilise `ResponseStatusException` pour les erreurs (ex: `EntityNotFoundException` -> 400, autres erreurs -> 500).

#### 2.1.2. `StatementImportController` (Package : `...controller`)

*   **Rôle :** Expose l'API REST pour le téléversement des fichiers de relevés.
*   **Endpoint :**
    *   `POST /api/v1/accounts/{accountId}/statements/upload` : Téléverse un fichier de relevé.
        *   Autorisation : `ADMIN, DATA_PROCESSOR`.
        *   **Paramètres :** `accountId` (UUID), `file` (MultipartFile), `templateId` (UUID).
        *   **Logique :**
            1.  Valide que le fichier n'est pas vide.
            2.  Récupère `AccountEntity` et `ImportTemplateEntity` via leurs dépôts. Lance `EntityNotFoundException` si non trouvés.
            3.  Vérifie que le modèle est applicable à la banque du compte.
            4.  Appelle `StatementProcessingService.parseStatement()` pour parser le fichier.
            5.  Si le parsing échoue ou ne retourne aucune transaction, lance `ResponseStatusException` (400).
            6.  Génère un `importJobId` (UUID).
            7.  Appelle `AccountStatementStorageService.storeParsedStatement()` (du module `account`) pour sauvegarder les transactions parsées.
            8.  Retourne `FileUploadResponseDto` avec un message de succès et le nombre de transactions.
        *   **Gestion des Erreurs :** `EntityNotFoundException` (compte, modèle), `ResponseStatusException` pour diverses erreurs (fichier vide, parsing, etc.).

### 2.2. Services

#### 2.2.1. `ImportTemplateService` (Package : `...service`)

*   **Rôle :** Implémente la logique métier pour le CRUD des `ImportTemplateEntity`.
*   **Dépendances :** `ImportTemplateRepository`, `TemplateFieldMappingRepository`, `BankRepository`, `UserRepository`, `ImportTemplateMapper`, `TemplateFieldMappingMapper`.
*   **Méthodes Clés :**
    *   `getAllTemplates()`, `getTemplateById(UUID)`, `getTemplatesForBank(UUID)` : Opérations de lecture.
    *   `createTemplate(ImportTemplateDto)` :
        *   Convertit DTO en entité.
        *   Associe `BankEntity` si `bankId` est fourni.
        *   Assure la liaison bidirectionnelle entre `ImportTemplateEntity` et ses `TemplateFieldMappingEntity` enfants (via `@AfterMapping` dans le mapper ou manuellement).
        *   Sauvegarde l'entité.
    *   `updateTemplate(UUID templateId, ImportTemplateDto)` :
        *   Récupère l'entité existante.
        *   Met à jour les champs simples.
        *   Gère la mise à jour de l'association `BankEntity`.
        *   Gère la mise à jour de la collection `fieldMappings` : une approche simple est de vider la collection existante (`clear()`) et d'ajouter les nouveaux mappings du DTO. L'annotation `@OneToMany(orphanRemoval = true)` sur `ImportTemplateEntity.fieldMappings` est cruciale pour que les anciens mappings soient supprimés de la base de données. Chaque nouveau `TemplateFieldMappingEntity` doit être lié à l'entité `ImportTemplateEntity` parente.
        *   Sauvegarde l'entité.
    *   `deleteTemplate(UUID templateId)` : Supprime le modèle. La suppression en cascade (via `CascadeType.ALL` et `orphanRemoval=true`) devrait supprimer les `TemplateFieldMappingEntity` associés.

#### 2.2.2. `StatementProcessingService` (Package : `...service`)

*   **Rôle :** Contient la logique de parsing des fichiers de relevés (actuellement Excel).
*   **Dépendances :** Aucune injection de service Spring, pure logique de traitement.
*   **Bibliothèques Utilisées :** Apache POI (`org.apache.poi`) pour la manipulation des fichiers Excel.
*   **Méthodes Clés :**
    *   `parseStatement(MultipartFile file, ImportTemplateEntity template)` : Méthode principale.
        1.  Ouvre le fichier Excel (`WorkbookFactory.create()`).
        2.  Sépare les `fieldMappings` du template en mappings d'en-tête et de corps (transactions).
        3.  Identifie la feuille Excel à utiliser via `sheetIdentifier` (défini dans les mappings d'en-tête, sinon première feuille).
        4.  Appelle `parseHeader()` pour extraire les données d'en-tête (`ParsedStatementHeaderDto`).
        5.  Appelle `parseTransactions()` pour extraire la liste des transactions (`List<ParsedTransactionDto>`).
        6.  Retourne `ParsedStatementDataDto` contenant l'en-tête, les transactions, et les erreurs de parsing.
    *   Méthodes privées auxiliaires :
        *   `getSheetIdentifier()`, `getSheet()` : Localisation de la feuille Excel.
        *   `parseHeader()` : Itère sur les mappings d'en-tête et extrait les valeurs des cellules spécifiées.
        *   `parseTransactions()` :
            *   Détermine la ligne de début des données (`dataStartRow` du template, ou heuristique).
            *   Itère sur les lignes de la feuille à partir de cette ligne.
            *   Pour chaque ligne, itère sur les mappings de corps et extrait les valeurs des colonnes spécifiées.
            *   Applique des règles de traitement (`applyProcessingRules()`) si définies (ex: remplacement de chaînes).
        *   `getCellValue(Sheet, ...)` et `getCellValue(Row, ...)` : Récupèrent la valeur d'une cellule (en-tête ou corps) en fonction de sa localisation (`sourceLocation` du mapping).
        *   `formatCellValueAsString(Cell)` : Convertit une cellule POI en chaîne de caractères, gérant les formules.
        *   `parseDate(String, String, ...)` : Parse une chaîne en `LocalDate` en utilisant le `formatPattern` du mapping ou des formats par défaut.
        *   `parseNumber(String, String, ...)` : Parse une chaîne en `BigDecimal` en utilisant le `formatPattern` ou une logique générique (gestion des virgules/points comme séparateurs décimaux).
        *   `findDataStartRow()` : Trouve la ligne de départ des transactions.
*   **Gestion des Erreurs de Parsing :** Les erreurs mineures (ex: format de date incorrect pour une cellule) sont ajoutées à une liste `parsingErrors` dans `ParsedStatementDataDto` sans arrêter le processus. Les erreurs majeures (ex: fichier illisible) peuvent lever des exceptions.

### 2.3. Entités

#### 2.3.1. `ImportTemplateEntity` (Package : `...entity`)

*   **Table :** `import_templates`
*   **Champs Principaux :**
    *   `id` (UUID, PK)
    *   `name` (String, unique)
    *   `bank` (ManyToOne vers `BankEntity`, optionnel) : Banque associée.
    *   `sourceType` (Enum `SourceType`) : Type de fichier (EXCEL, CSV).
    *   `description` (String)
    *   Champs d'audit (`createdByUser`, `updatedByUser`, `createdAt`, `updatedAt`).
    *   `fieldMappings` (OneToMany vers `TemplateFieldMappingEntity`, `CascadeType.ALL, orphanRemoval = true, FetchType.LAZY`) : Liste des mappings de champs pour ce modèle. La cascade et `orphanRemoval` sont importants pour la gestion du cycle de vie des mappings enfants.

#### 2.3.2. `TemplateFieldMappingEntity` (Package : `...entity`)

*   **Table :** `template_field_mappings`
*   **Contraintes :** `UniqueConstraint` sur `template_id` et `target_field`.
*   **Champs Principaux :**
    *   `id` (UUID, PK)
    *   `template` (ManyToOne vers `ImportTemplateEntity`, `nullable = false`) : Modèle parent.
    *   `targetField` (String) : Nom du champ de destination dans le système.
    *   `sourceLocation` (String) : Localisation dans le fichier source (cellule, nom de colonne).
    *   `dataType` (String) : Type de donnée (Date, Number, String).
    *   `formatPattern` (String) : Motif pour parser dates/nombres.
    *   `defaultValue` (String)
    *   `processingRules` (Map<String, Object>, `columnDefinition = "jsonb"`) : Règles de transformation (stockées en JSONB). Utilise `@Type(JsonType.class)` de Hypersistence Utils.
    *   `isHeaderField` (boolean) : Indicateur de champ d'en-tête.
    *   `fieldOrder` (Integer) : Ordre du champ.

### 2.4. DTOs (Package : `...dto`)

*   `ImportTemplateDto` : Pour la création/lecture des modèles. Contient une `List<TemplateFieldMappingDto>`.
*   `TemplateFieldMappingDto` : Pour les mappings de champs.
*   `FileUploadResponseDto` : Réponse après téléversement (`importJobId`, message, nombre de transactions).
*   `ParsedStatementDataDto` : Conteneur pour les données parsées (en-tête, transactions, erreurs de parsing).
*   `ParsedStatementHeaderDto` : Données d'en-tête du relevé.
*   `ParsedTransactionDto` : Une transaction individuelle parsée.

### 2.5. Mappers (Package : `...mapper`)

*   **`ImportTemplateMapper`** (utilise `TemplateFieldMappingMapper`) :
    *   Gère la conversion `ImportTemplateEntity` <=> `ImportTemplateDto`.
    *   Ignore les champs gérés par le service (ex: `bank`) ou l'audit JPA lors de `toEntity`.
    *   Mappe les informations de l'utilisateur et de la banque pour l'affichage dans le DTO.
    *   Utilise `@AfterMapping` pour assurer la liaison bidirectionnelle correcte entre `ImportTemplateEntity` et ses `TemplateFieldMappingEntity` enfants lors de la conversion `toEntity` et `updateEntityFromDto`.
*   **`TemplateFieldMappingMapper`** :
    *   Gère la conversion `TemplateFieldMappingEntity` <=> `TemplateFieldMappingDto`.
    *   Ignore la liaison `template` vers le parent dans `toEntity` car elle est gérée par `ImportTemplateMapper`.

### 2.6. Référentiels (Package : `...repository`)

*   **`ImportTemplateRepository`** : `JpaRepository<ImportTemplateEntity, UUID>`
    *   `findByName(String name)`
    *   `findByBankIdOrGeneric(UUID bankId)` : Requête JPQL pour trouver les modèles spécifiques à une banque ou les modèles génériques (où `bank_id` est nul).
    *   `findByBankIsNullOrderByNameAsc()` : Pour les modèles purement génériques.
*   **`TemplateFieldMappingRepository`** : `JpaRepository<TemplateFieldMappingEntity, UUID>`
    *   `findByTemplateId(UUID templateId)`
    *   `findByTemplateIdOrderByFieldOrderAsc(UUID templateId)`

### 2.7. Domaine (Package : `...domain`)

*   **`SourceType` (Enum)** : `EXCEL, CSV, API, MANUAL`. Définit les types de fichiers sources.

## 3. Workflow d'Importation (Focus Technique)

1.  **Requête API :** `POST /api/v1/accounts/{accountId}/statements/upload` avec `file` et `templateId`.
2.  **`StatementImportController`** :
    *   Valide les entrées, récupère `AccountEntity` et `ImportTemplateEntity`.
    *   Appelle `statementProcessingService.parseStatement(file, template)`.
3.  **`StatementProcessingService.parseStatement()`** :
    *   Utilise Apache POI pour ouvrir le fichier Excel.
    *   Exploite les `fieldMappings` du `template` pour :
        *   Identifier la feuille Excel.
        *   Extraire les données d'en-tête (ex: `accountNumber` depuis la cellule "B2").
        *   Déterminer la ligne de début des transactions.
        *   Pour chaque ligne de transaction, extraire les données de chaque colonne mappée (ex: date en colonne "A", description en "C").
        *   Convertir les chaînes extraites en types cibles (`LocalDate`, `BigDecimal`) en utilisant les `formatPattern` ou des logiques de parsing par défaut.
        *   Collecter les `ParsedTransactionDto` et `ParsedStatementHeaderDto`.
4.  **`StatementImportController`** (suite) :
    *   Reçoit `ParsedStatementDataDto`.
    *   Appelle `accountStatementStorageService.storeParsedStatement(account, parsedData, originalFilename, templateId, importJobId)`.
        *   Ce service (du module `account`) convertit `ParsedTransactionDto` en `AccountStatementEntity`.
        *   Persiste les `AccountStatementEntity` avec un statut initial (ex: `PENDING_REVIEW`).
        *   Met à jour `lastStatementImportDate` sur `AccountEntity`.
5.  Retourne `FileUploadResponseDto`.

## 4. Points Techniques Notables

*   **Parsing Excel Robuste :** Le `StatementProcessingService` contient une logique détaillée pour lire les cellules Excel, formater les valeurs, et parser les dates et nombres avec une certaine flexibilité (formats multiples, gestion des locales pour les nombres).
*   **Gestion des Erreurs de Parsing :** Les erreurs mineures de parsing sont collectées et retournées, permettant un import partiel ou une information à l'utilisateur.
*   **Modèles Flexibles :** La structure `ImportTemplateEntity` avec `TemplateFieldMappingEntity` permet une configuration fine pour différents formats de fichiers. L'utilisation de JSONB pour `processingRules` offre une extensibilité.
*   **Liaison Bidirectionnelle Mapper :** Les `@AfterMapping` dans `ImportTemplateMapper` sont importants pour s'assurer que la relation parent-enfant entre le modèle et ses mappings de champs est correctement établie des deux côtés après la conversion DTO -> Entité, ce qui est crucial pour la persistance JPA avec cascade.
*   **Dépendance Externe (Apache POI) :** Pour la lecture des fichiers Excel.
*   **Hypersistence Utils :** Pour le type JSONB (`@Type(JsonType.class)` sur `TemplateFieldMappingEntity.processingRules`).

Cette documentation technique vise à fournir une compréhension approfondie du module d'importation de relevés.
