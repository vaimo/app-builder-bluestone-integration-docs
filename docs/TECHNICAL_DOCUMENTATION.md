## Overview

The Bluestone Integration is a comprehensive event-driven system that synchronizes product, category, and attribute data from Bluestone PIM to Adobe Commerce. This integration uses Adobe I/O Events and App Builder runtime actions to ensure reliable, scalable data synchronization with advanced validation, retry mechanisms, and concurrency control.

## Integration Flow

![Architecture Diagram](architecture.png)

### High-Level Process

The integration follows a multi-stage process designed for reliability and data consistency:

1. **Sync Trigger**: Bluestone completes a sync operation and sends a webhook notification
2. **Event Processing**: App Builder ingestion webhook validates and publishes sync events
3. **Difference Detection**: Sync consumer fetches differences and publishes entity-specific events
4. **Data Enrichment**: Publisher fetches full entity data from Bluestone and enriches events
5. **Entity Processing**: Specialized consumers validate and process attributes, categories, and products individually
6. **Asset Synchronization**: Product processing triggers media synchronization events for asset management

### Detailed Flow

#### 1. Webhook Ingestion (`actions-src/ingestion/action/webhook/`)

**When**: Bluestone completes a sync operation  
**What happens**:

-   Bluestone sends a signed webhook to the App Builder ingestion endpoint
-   Webhook signature is verified using `BLUESTONE_PRIMARY_SECRET`
-   Request data is validated and structured
-   `SYNC_DONE` event is published to Adobe I/O Events

**Key Components**:

-   **Security**: HMAC SHA256 signature verification
-   **Validation**: Request structure and required field validation
-   **Event Publishing**: Publishes to Adobe I/O Events for downstream processing

#### 2. Sync Processing (`actions-src/sync/action/consumer/`)

**When**: `SYNC_DONE` event is received from I/O Events  
**What happens**:

-   Validates sync event structure
-   Fetches sync details and entity differences from Bluestone API
-   Processes three entity types in parallel: attributes, categories, and products
-   Publishes separate events for each changed entity

**Key Operations**:

```typescript
const [attributeDifferences, categoryDifferences, productDifferences] = await Promise.all([
    getSyncDifferencesForAttributes(bluestoneClient, event.syncId),
    getSyncDifferencesForCategories(bluestoneClient, event.syncId),
    getSyncDifferencesForProducts(bluestoneClient, event.syncId),
]);
```

#### 3. Data Enrichment (`actions-src/sync/action/publisher/`)

**When**: Entity synchronization events are received  
**What happens**:

-   Fetches complete entity data from Bluestone API based on entity type
-   Generates MD5 hash for change detection and duplicate prevention
-   Creates enriched `EntityDataReadyEvent` with full entity payload
-   Publishes to appropriate consumer based on entity type

**Hash Generation**:

```typescript
hash: entityData ? Md5.hashStr(JSON.stringify(entityData)) : "";
```

#### 4. Entity-Specific Processing

##### Attributes (`actions-src/attribute/action/consumer/`)

**Processing**: Immediate synchronization with Adobe Commerce  
**Validation Chain**:

1. **Removal Validator**: Checks if attribute is marked for deletion
2. **Hash Validator**: Compares entity hash to detect actual changes

**What happens**:

-   Maps Bluestone attributes to Adobe Commerce format using strategy pattern
-   Creates/updates/removes attributes directly via Adobe Commerce REST API
-   Assigns attributes to configured attribute group (`COMMERCE_ATTRIBUTE_GROUP_ID`)
-   Stores mapping between Bluestone and Commerce attribute IDs

**Attribute Type Mapping**:

-   **`boolean`** - Maps to `backend_type: "int"`, `frontend_input: "boolean"`
-   **`decimal`** - Maps to `backend_type: "decimal"`, `frontend_input: "text"`
-   **`integer`** - Maps to `backend_type: "int"`, `frontend_input: "text"`
-   **`single_select`** - Maps to `backend_type: "int"`, `frontend_input: "select"`
-   **`multi_select`** - Maps to `backend_type: "int"`, `frontend_input: "multiselect"`
-   **`date`** - Maps to `backend_type: "datetime"`, `frontend_input: "date"`
-   **`time`** - Maps to `backend_type: "datetime"`, `frontend_input: "text"`
-   **`date_time`** - Maps to `backend_type: "datetime"`, `frontend_input: "datetime"`
-   **`text`** - Maps to `backend_type: "varchar"`, `frontend_input: "text"`
-   **`formatted_text`** - Maps to `backend_type: "varchar"`, `frontend_input: "text"`

##### Categories (`actions-src/category/action/consumer/`)

**Processing**: Immediate synchronization with dependency validation  
**Validation Chain**:

1. **Removal Validator**: Checks if category is marked for deletion
2. **Hash Validator**: Compares entity hash to detect changes
3. **Parent Category Validator**: Ensures parent categories exist in Adobe Commerce

**What happens**:

-   Validates parent category existence before processing
-   Maps Bluestone categories to Adobe Commerce format
-   Creates/updates/removes categories directly via Adobe Commerce REST API
-   Handles hierarchical category relationships

**Dependency Handling**:
If parent category is missing, publishes event to request parent category sync:

```typescript
await publishEntityToSynchronizeEvent({
    syncId: params.data.syncId,
    context: params.data.context,
    entityId: missingParentCategoryId,
    entityType: SyncEntityType.CATEGORY,
    diffType: SyncDifferenceType.ADD,
});
```

##### Products (`actions-src/product/action/`)

**Processing**: Two-stage validation and individual event-driven processing

**Stage 1: Validation** (`actions-src/product/action/validate/`)  
**Validation Chain**:

1. **Removal Validator**: Checks if product is marked for deletion
2. **Hash Validator**: Compares entity hash to detect changes
3. **Attribute Validator**: Ensures all required attributes exist in Adobe Commerce
4. **Attribute Option Validator**: Validates select attribute options exist
5. **Category Validator**: Ensures all assigned categories exist in Adobe Commerce
6. **Configurable Product Validator**: For variant groups, validates child products exist

**What happens after validation**:

-   If validation passes, publishes `PRODUCT_PROCESS` event to Adobe I/O Events
-   If validation fails, publishes dependency sync events and returns 599 for retry

**Stage 2: Processing** (`actions-src/product/action/process/`)  
**When**: `PRODUCT_PROCESS` event is received from I/O Events  
**What happens**:

-   Processes products individually, one by one (no batching)
-   Maps products using builder pattern for attributes, categories, and general properties
-   Creates/updates/removes product directly via Adobe Commerce REST API
-   Executes post-processing for configurable product setup and media synchronization
-   Stores mapping between Bluestone and Commerce product IDs

**Product Type Handling**:

-   **Simple Products**: Direct mapping and storage with configurable attribute tracking
-   **Configurable Products**: Sets up variant relationships and configurable product options
-   **Variant Products**: Links to parent configurable products with variant-specific data

##### Assets (`actions-src/asset/action/consumer/`)

**Processing**: Individual media synchronization triggered by product processing  
**Validation Chain**:

1. **Removal Validator**: Checks if asset is marked for deletion
2. **Hash Validator**: Compares entity hash to detect changes
3. **Asset Validator**: Validates media URL, content type, and required fields

**What happens**:

-   Downloads media from Bluestone download URI
-   Converts media to base64 format for Adobe Commerce
-   Creates/updates/removes product media via Adobe Commerce media gallery API
-   Supports image types (JPEG, PNG, GIF, BMP, WebP, SVG)
-   Video support planned for future implementation
-   Stores mapping between Bluestone media and Commerce asset IDs

**Media Processing Flow**:

```typescript
// Triggered during product post-processing
const mediaDiff = this.calculateMediaDifferences(bluestoneProduct, commerceProduct, assetMappings);
await this.publishMediaEvents(event, mediaDiff);
```

## Key Integration Mechanisms

### 1. Hash-Based Change Detection

**Purpose**: Prevents processing of unchanged entities  
**Implementation**: MD5 hash of entity JSON stored in Adobe I/O State  
**Benefits**:

-   Reduces unnecessary API calls to Adobe Commerce
-   Prevents duplicate processing of same data
-   Provides audit trail of changes

```typescript
const hash = Md5.hashStr(JSON.stringify(entityData));
const existingHash = await getHash(entityType, entityId);
if (hash === existingHash) {
    return { type: ValidationResultType.SKIP, message: "No changes detected" };
}
```

### 2. Retry Mechanism with Error Code 599

**Purpose**: Handles validation failures gracefully with dependency resolution  
**How it works**:

-   When validation fails (e.g., missing attributes/categories), returns HTTP 599
-   Adobe I/O Events automatically retries actions that return 599
-   System publishes events to sync missing dependencies
-   Uses validation guard to prevent duplicate dependency requests during retries

**Implementation**:

```typescript
// Mark dependency events as published to prevent duplicates
await markDependencyEventsAsPublished(params.data);
await this.handleFailure(validationResult, params);
return errorResponseWithObject(HttpStatus.TO_BE_RETRIED, validationResult);
```

**Validation Guard**:

-   Stores state in Adobe I/O State with 1-minute TTL
-   Prevents publishing multiple dependency events during retry cycles
-   Key format: `dependency-events-{syncId}-{entityId}`

### 3. Concurrency Control

**Purpose**: Prevents parallel processing of the same entities  
**Implementation**: Distributed locking using Adobe I/O State  
**Features**:

-   5-minute default lock TTL
-   Automatic lock expiration handling
-   Retry mechanism with backoff (5 attempts, 1-second delay)
-   Graceful lock release in finally blocks

**Lock Key Format**: `lock-{entityType}-{entityId}`

```typescript
const lockId = await EntityLockManager.acquireLockWithRetry(
    event.entityType,
    event.entityId,
    300 // 5 minutes TTL
);

if (!lockId) {
    return errorResponseWithObject(HttpStatus.TO_BE_RETRIED, {
        message: "Entity is being processed by another instance",
        reason: "Concurrent processing prevention",
    });
}
```

### 4. Persistent Mapping Storage

**Purpose**: Maintains relationships between Bluestone and Adobe Commerce entities  
**Storage**: Adobe I/O Files (persistent storage)  
**Structure**: JSON files organized by entity type  
**Benefits**:

-   Enables entity updates and relationship management
-   Supports configurable product variant linking
-   Provides data consistency across sync operations

**Mapping Structure**:

```typescript
{
    bluestoneId: "bluestone-entity-id",
    commerceId: 123,
    commerceCode: "entity-code",
    data: [] // Additional mapping data (e.g., attribute options)
}
```

### 5. Event-Driven Architecture

**Components**:

-   **Adobe I/O Events**: Event backbone for all inter-action communication
-   **Provider System**: Separates internal and external event sources
-   **Event Types**: Specialized events for each entity type and operation

**Event Flow**:

```
Webhook → SYNC_DONE → EntityToSynchronize → EntityDataReady → Processing
```

## Configuration

The integration requires several environment variables for proper operation:

### Required Environment Variables

#### 1. Adobe Commerce Attribute Group Assignment

```bash
COMMERCE_ATTRIBUTE_GROUP_ID=1305
```

**Purpose**: Specifies which Adobe Commerce attribute group newly created attributes should be assigned to  
**Important Limitation**: All attributes are assigned to the default attribute set (ID: 4) and this specific group  
**Usage**: Used in `syncAttribute()` when creating new attributes

#### 2. Bluestone Configurable Attribute Group

```bash
BLUESTONE_CONFIGURABLE_ATTRIBUTE_GROUP_ID=684bde0f29b93b012e765ff3
```

**Purpose**: Identifies which Bluestone attribute group contains configurable product attributes  
**Usage**:

-   Filters out configurable attributes from variant group products during attribute building
-   Identifies configurable attributes for simple product storage strategy
-   Used in validation to determine product configurability

#### 3. Store Code Mapping

```bash
ADOBE_COMMERCE_MAPPING_LANGUAGES='[
    {"commerceId": 0, "commerceCode": "all", "externalId": "en"},
    {"commerceId": 6, "commerceCode": "german_store_view", "externalId": "l3682"}
]'
```

**Purpose**: Maps Bluestone context IDs to Adobe Commerce store views  
**Format**: JSON array of mapping objects  
**Properties**:

-   `commerceId`: Adobe Commerce store ID (0 for default/all stores)
-   `commerceCode`: Adobe Commerce store code
-   `externalId`: Bluestone context identifier

**Usage**:

-   Determines correct store view for attribute labels and values
-   Enables multi-store/multi-language product synchronization
-   Used in attribute mapping and product localization

## Full Synchronization Feature

### Overview

The full synchronization feature enables comprehensive entity synchronization from Bluestone PIM to Adobe Commerce, allowing you to trigger synchronization for all entities of a specific type (attributes, categories, or products). This feature internally uses pagination to efficiently handle large datasets while providing a simple, clean API interface.

### How It Works

The full sync action (`actions-src/full-sync/trigger/`) provides a web-exposed endpoint that supports two modes of operation:

#### Mode 1: Full Entity Synchronization (All Entities)

When `entityIds` parameter is not provided or empty:

1. **Paginated Entity Fetching**: Internally queries Bluestone Public API with pagination to retrieve entities efficiently
2. **Automatic Page Processing**: Processes all pages of entities (100 items per page)
3. **Generates Sync Events**: Creates `EntityToSynchronizeEvent` for each entity found
4. **Leverages Existing Pipeline**: Uses the same event-driven processing pipeline as regular syncs

#### Mode 2: Selective Entity Synchronization (Specific Entities)

When `entityIds` parameter is provided with specific entity IDs:

1. **Direct Event Publishing**: Bypasses Bluestone API fetching entirely
2. **Targeted Synchronization**: Creates `EntityToSynchronizeEvent` only for the specified entity IDs
3. **Performance Optimization**: Faster processing as it skips API calls to fetch entity lists
4. **Leverages Existing Pipeline**: Uses the same event-driven processing pipeline as regular syncs

### API Endpoints

**Endpoint Pattern**: `POST /full-sync/trigger`

**Supported Entity Types**:

-   `attribute` - Synchronizes product attributes using `GET /attributes?pageNo=0&itemsOnPage=100`
-   `category` - Synchronizes categories using `GET /categories/scan?pageNo=0&itemsOnPage=100`
-   `product` - Synchronizes products using `POST /products/list?pageNo=0&itemsOnPage=100`

**Request Parameters**:

-   `entityType` (required): Type of entities to synchronize (`attribute`, `category`, or `product`)
-   `context` (optional): Bluestone context/language (defaults to "default")
-   `syncId` (optional): Custom sync identifier (auto-generated if not provided)
-   `entityIds` (optional): Array of specific entity IDs to synchronize. When provided, bypasses Bluestone API fetching and directly synchronizes the specified entities

**Response Format**:

For full synchronization (all entities):

```json
{
    "message": "Full sync processed successfully for {entityType}",
    "entityType": "attribute|category|product",
    "context": "production",
    "syncId": "full-sync-uuid",
    "entitiesPublished": 100
}
```

For selective synchronization (specific entities):

```json
{
    "message": "Full sync processed successfully for {entityType}",
    "entityType": "attribute|category|product",
    "context": "production",
    "syncId": "full-sync-uuid",
    "entitiesPublished": 3
}
```

**Note**: When using `entityIds`, the `entitiesPublished` count reflects the number of specific entities requested, not the total count from Bluestone API.

### Use Cases and Benefits

#### Full Synchronization Mode

**Best for**:

-   Initial data migration from Bluestone to Adobe Commerce
-   Periodic full data reconciliation
-   Disaster recovery scenarios
-   When you need to ensure all entities are synchronized

**Benefits**:

-   Ensures complete data consistency
-   Automatically handles pagination
-   Discovers all available entities from Bluestone

#### Selective Synchronization Mode

**Best for**:

-   Targeted entity updates after specific changes
-   Performance-critical scenarios with large catalogs
-   Debugging specific entity synchronization issues
-   Re-synchronizing entities that previously failed

**Benefits**:

-   Faster execution (no API calls to fetch entity lists)
-   Reduced resource usage
-   More precise control over synchronization scope
-   Lower impact on Bluestone API rate limits

### Usage Examples

#### Full Synchronization (All Entities)

Synchronize all products from Bluestone:

```bash
curl -X POST "https://your-app-builder-namespace.adobeioruntime.net/api/v1/web/full-sync/trigger" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entityType": "product",
    "context": "en"
  }'
```

Synchronize all attributes with a custom sync ID:

```bash
curl -X POST "https://your-app-builder-namespace.adobeioruntime.net/api/v1/web/full-sync/trigger" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entityType": "attribute",
    "context": "en",
    "syncId": "custom-sync-123"
  }'
```

#### Selective Synchronization (Specific Entities)

Synchronize specific products by their IDs:

```bash
curl -X POST "https://your-app-builder-namespace.adobeioruntime.net/api/v1/web/full-sync/trigger" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entityType": "product",
    "context": "en",
    "entityIds": ["prod-123", "prod-456", "prod-789"]
  }'
```

Synchronize specific categories:

```bash
curl -X POST "https://your-app-builder-namespace.adobeioruntime.net/api/v1/web/full-sync/trigger" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entityType": "category",
    "context": "production",
    "entityIds": ["cat-001", "cat-002"],
    "syncId": "selective-cat-sync"
  }'
```

### Bluestone API Integration Details

The full sync feature integrates with specific Bluestone API endpoints:

**Attributes**:

-   **Endpoint**: `GET /attributes`
-   **Query Parameters**: `pageNo=0&itemsOnPage=100`
-   **Context**: Sent as header parameter

**Categories**:

-   **Endpoint**: `GET /categories/scan`
-   **Query Parameters**: `pageNo=0&itemsOnPage=100`
-   **Context**: Sent as header parameter

**Products**:

-   **Endpoint**: `POST /products/list`
-   **Query Parameters**: `pageNo=0&itemsOnPage=100`
-   **Body**: Empty object `{}`
-   **Context**: Sent as header parameter

All endpoints return structured responses with `totalCount` and `results[]` fields.

### Response Format

**Success Response**:

```json
{
    "statusCode": 200,
    "body": {
        "message": "Full sync triggered successfully for attribute",
        "entityType": "attribute",
        "context": "production",
        "syncId": "full-sync-550e8400-e29b-41d4-a716-446655440000",
        "entitiesPublished": 100,
        "totalCount": 2500
    }
}
```

**Error Responses**:

```json
{
    "statusCode": 400,
    "body": {
        "error": "Invalid entity type. Must be one of: attribute, category, product, asset"
    }
}
```

```json
{
    "statusCode": 500,
    "body": {
        "error": "Unsupported entity type: asset"
    }
}
```
