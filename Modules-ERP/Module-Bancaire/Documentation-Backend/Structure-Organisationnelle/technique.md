# Documentation Technique - Module Structure Organisationnelle

Ce document décrit l'architecture technique et les composants du module de gestion de la structure organisationnelle (`orgstructure`), qui inclut la gestion de l'entité Organisation principale et des entités Banque.

## 1. Vue d'Ensemble de l'Architecture

Le module `orgstructure` est fondamental pour définir l'entité légale utilisant le système ERP et les institutions bancaires avec lesquelles elle interagit. Il fournit les briques de base pour d'autres modules comme la gestion des comptes (`account`).

L'architecture est standard pour une application Spring Boot :

*   **Contrôleurs (Controllers) :** Gèrent les requêtes API REST pour l'organisation et les banques.
*   **Services :** Implémentent la logique métier et coordonnent les opérations.
*   **Entités (Entities) :** Modèles JPA représentant l'organisation, les banques, les adresses et les contacts.
*   **DTOs (Data Transfer Objects) :** Objets pour les échanges de données via l'API, validés par `jakarta.validation`.
*   **Mappers :** Convertissent les DTOs en Entités et vice-versa (MapStruct).
*   **Référentiels (Repositories) :** Interfaces Spring Data JPA pour l'accès à la base de données.

## 2. Composants Détaillés

### 2.1. Contrôleurs (Package : `com.cochepa.erp.banking.orgstructure.controller`)

#### 2.1.1. `OrganizationController`

*   **Rôle :** Gère les requêtes HTTP pour l'entité `OrganizationEntity` (supposée unique ou principale).
*   **Endpoints :**
    *   `GET /api/v1/organization` : Récupère l'organisation principale.
        *   Autorisation : `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   Appelle `OrganizationService.getPrimaryOrganization()`.
    *   `PUT /api/v1/organization` : Met à jour l'organisation principale. Peut la créer si elle n'existe pas (logique idempotente partielle).
        *   Autorisation : `ADMIN`.
        *   Appelle `OrganizationService.updatePrimaryOrganization()` ou `OrganizationService.createOrganization()`.
    *   `POST /api/v1/organization` : Crée explicitement l'organisation principale si elle n'existe pas.
        *   Autorisation : `ADMIN`.
        *   Appelle `OrganizationService.createOrganization()`. Retourne 409 Conflict si déjà existante.

#### 2.1.2. `BankController`

*   **Rôle :** Gère les requêtes HTTP pour les entités `BankEntity`.
*   **Endpoints :**
    *   `GET /api/v1/banks` : Liste toutes les banques.
        *   Autorisation : `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   Appelle `BankService.getAllBanks()`.
    *   `GET /api/v1/banks/{bankId}` : Récupère une banque par son ID.
        *   Autorisation : `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   Appelle `BankService.getBankById()`.
    *   `POST /api/v1/banks` : Crée une nouvelle banque.
        *   Autorisation : `ADMIN, DATA_PROCESSOR`.
        *   Appelle `BankService.createBank()`. Gestion des `EntityNotFoundException` (ex: organisation parente non trouvée).
    *   `PUT /api/v1/banks/{bankId}` : Met à jour une banque existante.
        *   Autorisation : `ADMIN, DATA_PROCESSOR`.
        *   Appelle `BankService.updateBank()`.
    *   `DELETE /api/v1/banks/{bankId}` : Supprime une banque.
        *   Autorisation : `ADMIN`.
        *   Appelle `BankService.deleteBank()`.

### 2.2. Services (Package : `com.cochepa.erp.banking.orgstructure.service`)

#### 2.2.1. `OrganizationService`

*   **Rôle :** Implémente la logique métier pour l'entité `OrganizationEntity`.
*   **Dépendances :** `OrganizationRepository`, `AddressRepository`, `OrganizationMapper`, `AddressMapper`.
*   **Méthodes Clés :**
    *   `getPrimaryOrganization()` : Récupère la première (et supposément unique) organisation.
    *   `createOrganization(OrganizationDto)` :
        *   Convertit DTO en entité.
        *   Gère la création ou la liaison d'une `AddressEntity` pour `primaryAddress`.
        *   Sauvegarde l'entité `OrganizationEntity`.
    *   `updatePrimaryOrganization(OrganizationDto)` :
        *   Récupère l'organisation existante.
        *   Met à jour ses champs et son adresse (création ou mise à jour de `AddressEntity` associée).
        *   Sauvegarde les modifications.

#### 2.2.2. `BankService`

*   **Rôle :** Implémente la logique métier pour les entités `BankEntity`.
*   **Dépendances :** `BankRepository`, `OrganizationRepository`, `AddressRepository`, `ContactRepository`, `BankMapper`, `AddressMapper`, `ContactMapper`.
*   **Méthodes Clés :**
    *   `getAllBanks()`, `getBankById(UUID)` : Fonctions de lecture standard.
    *   `createBank(BankDto)` :
        *   Valide l'existence de l'`OrganizationEntity` parente.
        *   Convertit DTO en `BankEntity`.
        *   Gère la création/liaison de `AddressEntity` pour `primaryAddress`.
        *   Gère la création/liaison des `ContactEntity` : si un contact a un ID, il est récupéré ; sinon, il est cherché par email ou créé.
        *   Sauvegarde l'entité `BankEntity`.
    *   `updateBank(UUID bankId, BankDto)` :
        *   Récupère la `BankEntity` existante.
        *   Met à jour les champs simples.
        *   Gère le changement d'`OrganizationEntity` parente.
        *   Met à jour `primaryAddress` (création, mise à jour, ou dé-liaison).
        *   Met à jour l'ensemble des `ContactEntity` associés (logique de `clear` puis `addAll` ; une logique de différentiel serait plus fine).
        *   Sauvegarde les modifications.
    *   `deleteBank(UUID bankId)` : Supprime la banque si elle existe.

### 2.3. Entités (Package : `com.cochepa.erp.banking.orgstructure.entity`)

Toutes les entités utilisent `@EntityListeners(AuditingEntityListener.class)` pour les champs `createdAt` et `updatedAt` (type `OffsetDateTime`).

#### 2.3.1. `OrganizationEntity`

*   **Table :** `organizations`
*   **Champs Principaux :**
    *   `id` (UUID, PK)
    *   `name` (String, `nullable = false, unique = true`)
    *   `registrationNumber`, `vatNumber` (String)
    *   `defaultCurrency` (String, length = 3)
    *   `primaryAddress` (OneToOne vers `AddressEntity`, `CascadeType.PERSIST, MERGE, REFRESH`) : Adresse principale.
*   **Relations :** Liée aux `BankEntity` (une organisation a plusieurs banques).

#### 2.3.2. `BankEntity`

*   **Table :** `banks`
*   **Contraintes :** `UniqueConstraint` sur `organization_id` et `name`.
*   **Champs Principaux :**
    *   `id` (UUID, PK)
    *   `organization` (ManyToOne vers `OrganizationEntity`, `nullable = false`) : Organisation parente.
    *   `name` (String, `nullable = false`)
    *   `swiftCode`, `website` (String)
    *   `primaryAddress` (OneToOne vers `AddressEntity`, `CascadeType.PERSIST, MERGE, REFRESH`)
    *   `contacts` (ManyToMany vers `ContactEntity`, `CascadeType.PERSIST, MERGE`) : Géré via une table de jointure `bank_contacts`.
*   **Relations :** Liée aux `AccountEntity` (une banque a plusieurs comptes).

#### 2.3.3. `AddressEntity`

*   **Table :** `addresses`
*   **Champs Principaux :** `id` (UUID, PK), `line1`, `line2`, `city`, `stateProvince`, `postalCode`, `country` (String).
*   **Relations :** Utilisée par `OrganizationEntity` et `BankEntity`.

#### 2.3.4. `ContactEntity`

*   **Table :** `contacts`
*   **Champs Principaux :** `id` (UUID, PK), `firstName`, `lastName`, `email` (unique), `phone`, `role` (String).
*   **Relations :** `ManyToMany` avec `BankEntity` (un contact peut être lié à plusieurs banques, une banque peut avoir plusieurs contacts).

### 2.4. DTOs (Package : `com.cochepa.erp.banking.orgstructure.dto`)

*   `OrganizationDto`, `BankDto`, `AddressDto`, `ContactDto` : Records Java utilisés pour la communication API.
*   **Validation :** Utilisent des annotations de `jakarta.validation.constraints` (ex: `@NotBlank`, `@Size`, `@Email`, `@Valid` pour les DTOs imbriqués).
*   **Structure :**
    *   `BankDto` contient un `organizationId` (UUID) pour la liaison, une `AddressDto` pour `primaryAddress`, et un `Set<ContactDto>` pour les contacts.
    *   `OrganizationDto` contient une `AddressDto` pour `primaryAddress`.

### 2.5. Mappers (Package : `com.cochepa.erp.banking.orgstructure.mapper`)

Utilisent MapStruct (`@Mapper(componentModel = "spring")`) pour la conversion Entité/DTO.

*   **`OrganizationMapper`** (utilise `AddressMapper`)
*   **`BankMapper`** (utilise `AddressMapper`, `ContactMapper`)
*   **`AddressMapper`**
*   **`ContactMapper`**
*   **Logique d'Ignore :**
    *   Les champs d'audit (`createdAt`, `updatedAt`) et les `id` (lors de `updateEntityFromDto`) sont généralement ignorés.
    *   Les relations complexes (ex: `BankEntity.organization`, `BankEntity.contacts`) sont souvent ignorées dans le mapper car leur gestion (création, liaison) est effectuée dans la couche Service. Par exemple, `BankMapper.toEntity()` ignore `organization`, qui est ensuite défini par `BankService` après avoir récupéré l'`OrganizationEntity`.

### 2.6. Référentiels (Package : `com.cochepa.erp.banking.orgstructure.repository`)

Interfaces Spring Data JPA étendant `JpaRepository`.

*   **`OrganizationRepository`** : `findByName(String name)`.
*   **`BankRepository`** : `findByOrganizationIdAndName(UUID, String)`, `findByOrganizationId(UUID)`.
*   **`AddressRepository`** : Pas de méthodes personnalisées.
*   **`ContactRepository`** : `findByEmail(String email)`.

## 3. Flux de Données et Logique de Service (Exemple : Création d'une Banque)

1.  **Requête API :** `POST /api/v1/banks` avec `BankDto` (incluant `organizationId`, et potentiellement de nouvelles `AddressDto` et `ContactDto` sans ID).
2.  **`BankController.createBank(bankDto)`** est appelé.
3.  Le contrôleur délègue à **`BankService.createBank(bankDto)`**.
4.  **`BankService`** :
    *   Récupère l'`OrganizationEntity` via `organizationRepository.findById(bankDto.organizationId())`. Lance `EntityNotFoundException` si non trouvée.
    *   Appelle `bankMapper.toEntity(bankDto)` pour une conversion initiale. L'entité `BankEntity` résultante a son champ `organization` à `null` à ce stade (car ignoré par le mapper).
    *   Définit `bankEntity.setOrganization(organization)`.
    *   **Gestion de l'Adresse :**
        *   Si `bankDto.primaryAddress()` est fourni :
            *   Si `bankDto.primaryAddress().id()` est nul, convertit `AddressDto` en `AddressEntity` (`addressMapper.toEntity()`), la sauvegarde (`addressRepository.save()`), puis l'assigne à `bankEntity.setPrimaryAddress()`.
            *   Si `bankDto.primaryAddress().id()` est non nul, tente de récupérer l'adresse. Si non trouvée, la crée. (La logique actuelle dans `BankService.createBank` est plus simple : elle crée si l'ID est nul, sinon elle suppose qu'elle sera liée si l'ID existe, mais ne la récupère pas explicitement pour la création de banque).
    *   **Gestion des Contacts :**
        *   Itère sur `bankDto.contacts()`:
            *   Pour chaque `ContactDto` : Si `id` est non nul, récupère via `contactRepository.findById()`.
            *   Si `id` est nul, tente de trouver par email via `contactRepository.findByEmail()`. Si non trouvé, convertit en `ContactEntity` (`contactMapper.toEntity()`) et sauvegarde (`contactRepository.save()`).
            *   Ajoute le `ContactEntity` (existant ou nouveau) à un `Set<ContactEntity>`.
        *   Assigne cet ensemble à `bankEntity.setContacts()`.
    *   Appelle `bankRepository.save(bankEntity)`. Les relations de cascade (`CascadeType.PERSIST, MERGE` sur `primaryAddress` et `contacts` dans `BankEntity`) peuvent jouer un rôle ici si les entités associées ne sont pas encore persistées.
    *   Convertit l'entité `BankEntity` sauvegardée en `BankDto` via `bankMapper.toDto()` et la retourne.
5.  **`BankController`** retourne le DTO avec un statut 201 Created.

## 4. Points Techniques Notables

*   **Gestion des Adresses/Contacts :** La création et la mise à jour des adresses et contacts sont entrelacées avec celles des banques/organisations. Les services contiennent la logique pour soit créer de nouvelles entités Adresse/Contact, soit lier des entités existantes, en fonction de la présence d'ID dans les DTOs.
*   **Unicité :** Des contraintes d'unicité sont définies au niveau des entités (`OrganizationEntity.name`, `BankEntity.organization_id+name`).
*   **Transactions :** Les méthodes de service sont typiquement annotées `@Transactional`. Les opérations de lecture seule avec `@Transactional(readOnly = true)`.
*   **Cascade JPA :** L'utilisation de `CascadeType.PERSIST` et `CascadeType.MERGE` sur les relations (ex: `BankEntity.primaryAddress`) signifie que si une nouvelle `AddressEntity` est associée à une `BankEntity` et que la `BankEntity` est sauvegardée, l'`AddressEntity` sera également persistée.

Cette documentation technique offre un aperçu du fonctionnement interne du module de structure organisationnelle.
