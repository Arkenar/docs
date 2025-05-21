# Documentation Technique - Module des Taux de Change

Ce document décrit l'architecture technique et les composants du module de gestion des taux de change.

## 1. Vue d'Ensemble de l'Architecture

Le module `exchangerate` est responsable de la gestion et de la fourniture des taux de change entre différentes devises. Il est conçu pour être utilisé par d'autres modules, notamment le module d'analytique pour la consolidation des soldes.

L'architecture est structurée comme suit :

*   **Contrôleur (Controller) :** Gère les requêtes API REST pour la consultation et la manipulation des taux de change.
*   **Service :** Contient la logique métier, y compris la récupération, la création, la mise à jour et la suppression des taux.
*   **Entité (Entity) :** Représente un taux de change dans la base de données (JPA).
*   **DTO (Data Transfer Object) :** Structure de données pour les échanges via l'API.
*   **Mapper :** Assure la conversion entre l'entité et le DTO (MapStruct).
*   **Référentiel (Repository) :** Interface avec la base de données pour les opérations CRUD (Spring Data JPA).

## 2. Composants Détaillés

### 2.1. Contrôleur (Package : `com.cochepa.erp.banking.exchangerate.controller`)

#### 2.1.1. `ExchangeRateController`

*   **Rôle :** Expose les endpoints API REST pour gérer les taux de change.
*   **Endpoints :**
    *   `GET /api/v1/exchange-rates`
        *   **Objectif :** Obtenir un taux de change spécifique pour une paire de devises à une date donnée.
        *   **Paramètres :** `fromCurrency` (String), `toCurrency` (String), `date` (LocalDate).
        *   **Autorisation :** `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   **Appelle :** `ExchangeRateService.getExchangeRate()`.
        *   **Réponse :** `ResponseEntity<ExchangeRateDto>` ou 404 si non trouvé.
    *   `GET /api/v1/exchange-rates/by-date`
        *   **Objectif :** Obtenir tous les taux de change pour une date donnée.
        *   **Paramètres :** `date` (LocalDate).
        *   **Autorisation :** `ADMIN, USER, DATA_PROCESSOR, VIEWER`.
        *   **Appelle :** `ExchangeRateService.getExchangeRatesByDate()`.
        *   **Réponse :** `List<ExchangeRateDto>`.
    *   `POST /api/v1/exchange-rates`
        *   **Objectif :** Créer ou mettre à jour un taux de change (idempotent).
        *   **Corps de la requête :** `@Valid ExchangeRateDto`.
        *   **Autorisation :** `ADMIN, DATA_PROCESSOR`.
        *   **Appelle :** `ExchangeRateService.createOrUpdateExchangeRate()`.
        *   **Réponse :** `ResponseEntity<ExchangeRateDto>`. Gestion des exceptions avec `ResponseStatusException` pour les erreurs serveur.
    *   `POST /api/v1/exchange-rates/batch`
        *   **Objectif :** Créer ou mettre à jour plusieurs taux de change en lot.
        *   **Corps de la requête :** `@Valid List<ExchangeRateDto>`.
        *   **Autorisation :** `ADMIN, DATA_PROCESSOR`.
        *   **Appelle :** `ExchangeRateService.createOrUpdateExchangeRates()`.
        *   **Réponse :** `ResponseEntity<List<ExchangeRateDto>>`.
    *   `DELETE /api/v1/exchange-rates/{id}`
        *   **Objectif :** Supprimer un taux de change par son ID.
        *   **Paramètre de chemin :** `id` (UUID).
        *   **Autorisation :** `ADMIN`.
        *   **Appelle :** `ExchangeRateService.deleteExchangeRate()`.
        *   **Réponse :** `ResponseEntity<Void>` (204 No Content ou 404 Not Found).

### 2.2. Service (Package : `com.cochepa.erp.banking.exchangerate.service`)

#### 2.2.1. `ExchangeRateService`

*   **Rôle :** Implémente la logique métier pour la gestion des taux de change.
*   **Dépendances :**
    *   `ExchangeRateRepository` : Pour l'accès aux données.
    *   `ExchangeRateMapper` : Pour la conversion Entité/DTO.
    *   `RestTemplate` (commenté) : Prévu pour une éventuelle récupération de taux depuis une API externe.
*   **Méthodes Clés :**
    *   `getExchangeRate(String fromCurrency, String toCurrency, LocalDate date)`
        *   Normalise les codes de devise en majuscules.
        *   Appelle `exchangeRateRepository.findByFromCurrencyAndToCurrencyAndDate()`.
        *   Mappe le résultat en DTO.
        *   Transactionnel en lecture seule (`@Transactional(readOnly = true)`).
    *   `getExchangeRatesByDate(LocalDate date)`
        *   Appelle `exchangeRateRepository.findByDate()`.
        *   Mappe la liste des résultats en DTOs.
        *   Transactionnel en lecture seule.
    *   `createOrUpdateExchangeRate(ExchangeRateDto exchangeRateDto)`
        *   Vérifie l'existence d'un taux pour la combinaison devise source/cible et date via `exchangeRateRepository.findByFromCurrencyAndToCurrencyAndDate()`.
        *   Si existant, met à jour l'entité avec les nouvelles valeurs du DTO (`exchangeRateMapper.updateEntityFromDto()`).
        *   Sinon, crée une nouvelle entité à partir du DTO (`exchangeRateMapper.toEntity()`).
        *   Les codes de devise sont normalisés en majuscules par le mapper.
        *   Sauvegarde l'entité via `exchangeRateRepository.save()`.
        *   Retourne le DTO de l'entité sauvegardée.
        *   Transactionnel (`@Transactional`).
    *   `createOrUpdateExchangeRates(List<ExchangeRateDto> exchangeRateDtos)`
        *   Itère sur la liste de DTOs et appelle `createOrUpdateExchangeRate()` pour chacun.
        *   Collecte les résultats.
        *   Transactionnel.
    *   `deleteExchangeRate(UUID id)`
        *   Vérifie l'existence par ID avec `exchangeRateRepository.existsById()`.
        *   Si existe, supprime via `exchangeRateRepository.deleteById()`.
        *   Transactionnel.
    *   `fetchExternalExchangeRates()` (commenté)
        *   Prévu pour être une méthode planifiée (`@Scheduled`) qui récupèrerait les taux d'une API externe et les stockerait localement en utilisant `createOrUpdateExchangeRate()`.

### 2.3. Entité (Package : `com.cochepa.erp.banking.exchangerate.entity`)

#### 2.3.1. `ExchangeRateEntity`

*   **Rôle :** Représente un taux de change dans la table `exchange_rates`.
*   **Annotations JPA :**
    *   `@Entity`, `@Table(name = "exchange_rates", ...)`
    *   `@UniqueConstraint(columnNames = {"from_currency", "to_currency", "date"})` : Assure qu'il n'y a qu'un seul taux par paire de devises pour une date donnée.
    *   `@EntityListeners(AuditingEntityListener.class)` : Pour la gestion automatique des champs `createdAt` et `updatedAt`.
*   **Attributs Principaux :**
    *   `id` (UUID, PK, `@GeneratedValue(strategy = GenerationType.UUID)`)
    *   `fromCurrency` (String, `nullable = false, length = 3`) : Devise source (ex: USD).
    *   `toCurrency` (String, `nullable = false, length = 3`) : Devise cible (ex: EUR).
    *   `rate` (BigDecimal, `nullable = false, precision = 19, scale = 8`) : Le taux de change.
    *   `date` (LocalDate, `nullable = false`) : Date d'application du taux.
    *   `createdAt` (OffsetDateTime, `@CreatedDate`) : Date de création, gérée par l'audit JPA.
    *   `updatedAt` (OffsetDateTime, `@LastModifiedDate`) : Date de dernière modification, gérée par l'audit JPA.
*   **Constructeur :** `@Builder` de Lombok est disponible.

### 2.4. DTO (Package : `com.cochepa.erp.banking.exchangerate.dto`)

#### 2.4.1. `ExchangeRateDto`

*   **Rôle :** Objet Record utilisé pour transférer les données des taux de change via l'API.
*   **Champs :**
    *   `id` (UUID, optionnel pour la création)
    *   `fromCurrency` (String)
    *   `toCurrency` (String)
    *   `rate` (BigDecimal)
    *   `date` (LocalDate)
*   **Validation :** Utilise les annotations de `jakarta.validation.constraints` :
    *   `@NotBlank`, `@Size(min = 3, max = 3)` pour `fromCurrency` et `toCurrency`.
    *   `@NotNull`, `@DecimalMin(value = "0.0", inclusive = false)` pour `rate` (assure un taux positif).
    *   `@NotNull` pour `date`.

### 2.5. Mapper (Package : `com.cochepa.erp.banking.exchangerate.mapper`)

#### 2.5.1. `ExchangeRateMapper`

*   **Rôle :** Convertit `ExchangeRateEntity` en `ExchangeRateDto` et vice-versa, en utilisant MapStruct.
*   **Dépendances :** `DateTimeMapper` (probablement pour gérer les types `OffsetDateTime` si présents dans les DTOs, bien qu'ils soient ignorés ici).
*   **Méthodes :**
    *   `toEntity(ExchangeRateDto dto)` : Mappe DTO vers Entité. `createdAt` et `updatedAt` sont ignorés car gérés par l'audit JPA.
    *   `toDto(ExchangeRateEntity entity)` : Mappe Entité vers DTO.
    *   `updateEntityFromDto(ExchangeRateDto dto, @MappingTarget ExchangeRateEntity entity)` : Met à jour une entité existante à partir d'un DTO. `id`, `createdAt`, `updatedAt` sont ignorés.
    *   `normalizeCurrencyCodesEntity(ExchangeRateDto dto, @MappingTarget ExchangeRateEntity entity)` (`@AfterMapping`) : Hook appelé après le mapping `toEntity`. S'assure que `fromCurrency` et `toCurrency` dans l'entité sont stockés en majuscules.

### 2.6. Référentiel (Package : `com.cochepa.erp.banking.exchangerate.repository`)

#### 2.6.1. `ExchangeRateRepository`

*   **Interface :** `JpaRepository<ExchangeRateEntity, UUID>`
*   **Méthodes de Requête Personnalisées :**
    *   `findByFromCurrencyAndToCurrencyAndDate(String fromCurrency, String toCurrency, LocalDate date)` : Récupère un taux optionnel pour une paire de devises et une date.
    *   `findByFromCurrencyAndToCurrencyOrderByDateDesc(String fromCurrency, String toCurrency)` : (Non utilisé directement par les services actuels, mais pourrait servir à trouver le taux le plus récent pour une paire).
    *   `findByDate(LocalDate date)` : Récupère tous les taux pour une date donnée.

## 3. Source des Données et Gestion

*   **Stockage Interne :** Les taux de change sont stockés dans la table `exchange_rates` de la base de données de l'application.
*   **Alimentation :**
    *   Manuelle : Via les endpoints `POST /api/v1/exchange-rates` et `POST /api/v1/exchange-rates/batch`.
    *   Automatisée (envisagée) : Le code du `ExchangeRateService` contient une section commentée (`fetchExternalExchangeRates`) qui suggère une future intégration avec une API externe pour récupérer et mettre à jour les taux de manière périodique. Cette fonctionnalité n'est pas active.
*   **Unicité :** La contrainte d'unicité en base de données (`from_currency`, `to_currency`, `date`) garantit l'intégrité des données et évite les doublons pour un même taux à une même date.
*   **Normalisation :** Les codes de devise sont normalisés en majuscules avant d'être persistés, assurant la cohérence des recherches.

Cette documentation technique offre un aperçu du fonctionnement interne du module des taux de change.
