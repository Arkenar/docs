# Documentation Technique - Module de Gestion des OPCVM (Fonds d'Investissement)

Ce document décrit l'architecture technique et les composants du module de gestion des détentions d'Organismes de Placement Collectif en Valeurs Mobilières (OPCVM).

## 1. Vue d'Ensemble de l'Architecture

Le module `opcvm` permet de gérer les informations relatives aux fonds d'investissement détenus, potentiellement rattachés à des comptes bancaires existants dans le système.

L'architecture du module est la suivante :

*   **Contrôleur (Controller) :** Expose les API REST pour interagir avec les données des détentions d'OPCVM.
*   **Service :** Contient la logique métier pour la création, la lecture, la mise à jour et la suppression (CRUD) des détentions d'OPCVM.
*   **Entité (Entity) :** Modèle JPA représentant une ligne de détention d'OPCVM dans la base de données.
*   **DTO (Data Transfer Object) :** Objet utilisé pour transférer les données entre le client et le serveur via l'API.
*   **Mapper :** Assure la conversion entre les entités JPA et les DTOs (utilisation de MapStruct).
*   **Référentiel (Repository) :** Interface Spring Data JPA pour l'accès aux données des OPCVM en base.

## 2. Composants Détaillés

### 2.1. Contrôleur (Package : `com.cochepa.erp.banking.opcvm.controller`)

#### 2.1.1. `OpcvmHoldingController`

*   **Rôle :** Gère les requêtes HTTP pour les opérations CRUD sur les détentions d'OPCVM.
*   **Endpoints Principaux :**
    *   `GET /api/v1/opcvm-holdings` : Liste toutes les détentions d'OPCVM.
        *   Paramètre optionnel : `accountId` (UUID) pour filtrer par compte de rattachement.
        *   Autorisation : `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   Appelle : `OpcvmHoldingService.getAllOpcvmHoldings()`.
    *   `GET /api/v1/opcvm-holdings/{holdingId}` : Récupère une détention d'OPCVM par son ID.
        *   Autorisation : `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   Appelle : `OpcvmHoldingService.getOpcvmHoldingById()`.
    *   `POST /api/v1/opcvm-holdings` : Crée une nouvelle détention d'OPCVM.
        *   Corps de la requête : `@Valid OpcvmHoldingDto`.
        *   Autorisation : `ADMIN, DATA_PROCESSOR`.
        *   Appelle : `OpcvmHoldingService.createOpcvmHolding()`.
        *   Gestion des erreurs : `EntityNotFoundException` (pour `accountId` non trouvé) -> 400 Bad Request; `IllegalStateException` (ex: doublon ISIN/compte) -> 409 Conflict.
    *   `PUT /api/v1/opcvm-holdings/{holdingId}` : Met à jour une détention d'OPCVM existante.
        *   Corps de la requête : `@Valid OpcvmHoldingDto`.
        *   Autorisation : `ADMIN, DATA_PROCESSOR`.
        *   Appelle : `OpcvmHoldingService.updateOpcvmHolding()`.
        *   Gestion des erreurs similaire à la création.
    *   `DELETE /api/v1/opcvm-holdings/{holdingId}` : Supprime une détention d'OPCVM.
        *   Autorisation : `ADMIN`.
        *   Appelle : `OpcvmHoldingService.deleteOpcvmHolding()`.

### 2.2. Service (Package : `com.cochepa.erp.banking.opcvm.service`)

#### 2.2.1. `OpcvmHoldingService`

*   **Rôle :** Implémente la logique métier pour la gestion des détentions d'OPCVM.
*   **Dépendances :**
    *   `OpcvmHoldingRepository` : Pour l'accès aux données des OPCVM.
    *   `AccountRepository` : Pour valider et lier l'entité `AccountEntity` si un `accountId` est fourni.
    *   `OpcvmHoldingMapper` : Pour la conversion entre Entité et DTO.
*   **Méthodes Clés :**
    *   `getAllOpcvmHoldings(UUID accountId)` : Récupère toutes les détentions, ou filtre par `accountId` si fourni.
        *   Transactionnel en lecture seule (`@Transactional(readOnly = true)`).
    *   `getOpcvmHoldingById(UUID holdingId)` : Récupère une détention par son ID.
        *   Transactionnel en lecture seule.
    *   `createOpcvmHolding(OpcvmHoldingDto opcvmHoldingDto)` :
        *   Convertit le DTO en entité.
        *   Si `accountId` est fourni, récupère et associe l'`AccountEntity` correspondante. Lance `EntityNotFoundException` si le compte n'existe pas.
        *   Vérifie l'unicité :
            *   Si `isinCode` est fourni et `accountId` aussi, vérifie qu'il n'existe pas d'autre OPCVM avec le même ISIN pour ce compte (`opcvmHoldingRepository.findByAccountIdAndIsinCode()`).
            *   Si `isinCode` est fourni mais `accountId` est nul, vérifie qu'il n'existe pas d'autre OPCVM "générique" (sans compte) avec le même ISIN.
            *   Lance `IllegalStateException` en cas de doublon.
        *   Sauvegarde la nouvelle entité (`opcvmHoldingRepository.save()`). L'entité calcule `totalValue` via `@PrePersist`.
        *   Retourne le DTO de l'entité sauvegardée.
        *   Transactionnel (`@Transactional`).
    *   `updateOpcvmHolding(UUID holdingId, OpcvmHoldingDto opcvmHoldingDto)` :
        *   Récupère l'entité existante par `holdingId`.
        *   Met à jour l'association avec `AccountEntity` si `accountId` change ou est fourni.
        *   Vérifie l'unicité de l'ISIN si l'ISIN ou le compte associé a changé, similaire à la logique de création, en s'assurant de ne pas comparer l'entité à elle-même.
        *   Applique les modifications du DTO à l'entité via `opcvmHoldingMapper.updateEntityFromDto()`.
        *   Sauvegarde l'entité mise à jour. `totalValue` est recalculé via `@PreUpdate`.
        *   Retourne le DTO de l'entité sauvegardée.
        *   Transactionnel.
    *   `deleteOpcvmHolding(UUID holdingId)` :
        *   Vérifie l'existence de la détention.
        *   Si elle existe, la supprime.
        *   Transactionnel.

### 2.3. Entité (Package : `com.cochepa.erp.banking.opcvm.entity`)

#### 2.3.1. `OpcvmHoldingEntity`

*   **Rôle :** Représente une ligne de détention d'OPCVM dans la table `opcvm_holdings`.
*   **Annotations JPA :**
    *   `@Entity`, `@Table(name = "opcvm_holdings", ...)`
    *   `@UniqueConstraint(columnNames = {"account_id", "isin_code"})` : Assure qu'un ISIN est unique pour un compte donné. Si `account_id` peut être nul, cette contrainte ne s'applique qu'aux lignes où `account_id` est non nul. La logique de service gère l'unicité pour les ISIN "génériques" (où `account_id` est nul).
    *   `@EntityListeners(AuditingEntityListener.class)` : Pour les champs d'audit `createdAt` et `updatedAt`.
*   **Attributs Principaux :**
    *   `id` (UUID, PK, `@GeneratedValue(strategy = GenerationType.UUID)`)
    *   `account` (ManyToOne vers `AccountEntity`, `FetchType.LAZY`) : Le compte bancaire de rattachement (optionnel, peut être nul).
    *   `name` (String, `nullable = false, length = 255`) : Nom du fonds.
    *   `isinCode` (String, `length = 12`) : Code ISIN du fonds.
    *   `quantity` (BigDecimal, `nullable = false, precision = 19, scale = 6`) : Nombre de parts détenues.
    *   `unitPrice` (BigDecimal, `precision = 19, scale = 6`) : Prix unitaire de la part (valeur liquidative).
    *   `totalValue` (BigDecimal, `precision = 19, scale = 4`) : Valeur totale de la détention (quantité * prix unitaire).
    *   `currency` (String, `nullable = false, length = 3`) : Devise de valorisation.
    *   `valuationDate` (LocalDate, `nullable = false`) : Date de la valorisation.
    *   `createdAt` (OffsetDateTime, `@CreatedDate`)
    *   `updatedAt` (OffsetDateTime, `@LastModifiedDate`)
*   **Callbacks de Cycle de Vie JPA :**
    *   `@PrePersist`, `@PreUpdate` sur la méthode `calculateTotalValue()` : Calcule automatiquement `totalValue` avant chaque sauvegarde (création ou mise à jour) si `quantity` et `unitPrice` sont non nuls.

### 2.4. DTO (Package : `com.cochepa.erp.banking.opcvm.dto`)

#### 2.4.1. `OpcvmHoldingDto`

*   **Rôle :** Objet Record utilisé pour transférer les données des détentions d'OPCVM.
*   **Champs :**
    *   `id` (UUID, optionnel pour la création)
    *   `accountId` (UUID, optionnel)
    *   `name` (String)
    *   `isinCode` (String, optionnel)
    *   `quantity` (BigDecimal)
    *   `unitPrice` (BigDecimal)
    *   `totalValue` (BigDecimal, lecture seule pour le client, calculé côté serveur)
    *   `currency` (String)
    *   `valuationDate` (LocalDate)
*   **Validation :** Utilise les annotations de `jakarta.validation.constraints` (ex: `@NotBlank`, `@NotNull`, `@Size`, `@DecimalMin`).

### 2.5. Mapper (Package : `com.cochepa.erp.banking.opcvm.mapper`)

#### 2.5.1. `OpcvmHoldingMapper`

*   **Rôle :** Convertit `OpcvmHoldingEntity` en `OpcvmHoldingDto` et vice-versa, en utilisant MapStruct.
*   **Configuration :** `@Mapper(componentModel = "spring")` pour l'intégration avec Spring.
*   **Méthodes :**
    *   `toEntity(OpcvmHoldingDto dto)` : Mappe DTO vers Entité.
        *   Ignore `account` (géré par le service), `totalValue` (calculé par l'entité), `createdAt`, `updatedAt` (audit JPA).
    *   `toDto(OpcvmHoldingEntity entity)` : Mappe Entité vers DTO.
        *   Mappe `entity.getAccount().getId()` vers `accountId` dans le DTO.
    *   `updateEntityFromDto(OpcvmHoldingDto dto, @MappingTarget OpcvmHoldingEntity entity)` : Met à jour une entité existante à partir d'un DTO.
        *   Ignore `id`, `account`, `totalValue`, `createdAt`, `updatedAt`.

### 2.6. Référentiel (Package : `com.cochepa.erp.banking.opcvm.repository`)

#### 2.6.1. `OpcvmHoldingRepository`

*   **Interface :** `JpaRepository<OpcvmHoldingEntity, UUID>`
*   **Méthodes de Requête Personnalisées :**
    *   `findByAccountId(UUID accountId)` : Récupère toutes les détentions pour un compte spécifique.
    *   `findByIsinCode(String isinCode)` : Récupère les détentions par code ISIN (peut retourner plusieurs si l'ISIN est partagé entre un OPCVM générique et un rattaché à un compte, bien que la logique de service tente d'éviter cela).
    *   `findByAccountIdAndIsinCode(UUID accountId, String isinCode)` : Récupère une détention optionnelle pour une combinaison spécifique de compte et d'ISIN (utilisé pour la vérification d'unicité).

## 3. Flux de Données (Exemple : Création d'une Détention d'OPCVM)

1.  **Requête Client :** `POST /api/v1/opcvm-holdings` avec un `OpcvmHoldingDto` dans le corps.
2.  **`OpcvmHoldingController.createOpcvmHolding(dto)`** est appelé.
3.  Le contrôleur délègue à **`OpcvmHoldingService.createOpcvmHolding(dto)`**.
4.  **`OpcvmHoldingService`** :
    *   Appelle `opcvmHoldingMapper.toEntity(dto)` pour convertir le DTO en `OpcvmHoldingEntity`.
    *   Si `dto.accountId()` est présent, appelle `accountRepository.findById()` pour récupérer l'`AccountEntity`. Si non trouvé, lance `EntityNotFoundException`. L'entité `AccountEntity` est ensuite associée à l'`OpcvmHoldingEntity`.
    *   Effectue la vérification d'unicité de l'ISIN (par compte ou globalement).
    *   Appelle `opcvmHoldingRepository.save(entity)`.
        *   Avant la persistance, la méthode `calculateTotalValue()` de `OpcvmHoldingEntity` (annotée `@PrePersist`) est exécutée pour calculer `totalValue`.
        *   Les champs `createdAt` et `updatedAt` sont également initialisés par l'audit JPA.
    *   L'entité sauvegardée est reconvertie en DTO via `opcvmHoldingMapper.toDto()`.
5.  **`OpcvmHoldingController`** retourne le DTO résultant avec un statut 201 Created.

Cette documentation technique fournit une vue d'ensemble du fonctionnement interne du module de gestion des OPCVM.
