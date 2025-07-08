# Debug Guide

This guide explains how to use the debug endpoint to inspect and manage hashes and mappings in the Bluestone PIM to Adobe Commerce integration. The debug endpoint provides tools to understand the internal state of the integration and troubleshoot synchronization issues.

## Overview

The debug endpoint (`/api/v1/web/debug-internal/consumer`) allows you to:

-   **Inspect Mappings**: View the persistent mappings between Bluestone and Adobe Commerce entities
-   **Inspect Hashes**: View the temporary hash values used for change detection
-   **Remove Data**: Clear mappings or hashes for troubleshooting
-   **Save Mappings**: Manually create or update entity mappings

## Authentication

All debug requests require Adobe authentication. Include your authentication headers when making requests to the debug endpoint.

## Available Operations

### 1. Get Mappings

Retrieve persistent mappings between Bluestone and Adobe Commerce entities.

#### Get Category Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-category-mappings"
    }
  }'
```

**Response:**

```json
{
    "status": "success",
    "message": "category-mappings",
    "data": [
        {
            "bluestoneId": "category-123",
            "commerceId": 15,
            "commerceCode": "default",
            "data": []
        },
        {
            "bluestoneId": "category-456",
            "commerceId": 16,
            "commerceCode": "german_store_view",
            "data": []
        }
    ]
}
```

#### Get Product Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-product-mappings"
    }
  }'
```

#### Get Attribute Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-attribute-mappings"
    }
  }'
```

#### Get Asset Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-asset-mappings"
    }
  }'
```

### 2. Get Hashes

Retrieve temporary hash values used for change detection. Hashes are stored in the temporary state and are used to determine if an entity has changed since the last synchronization.

#### Get Product Hashes

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-product-hashes"
    }
  }'
```

**Response:**

```json
{
    "status": "success",
    "message": "product-hashes",
    "data": [
        ["hash-product-123", "a1b2c3d4e5f6g7h8i9j0"],
        ["hash-product-456", "b2c3d4e5f6g7h8i9j0k1"]
    ]
}
```

#### Get Category Hashes

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-category-hashes"
    }
  }'
```

#### Get Attribute Hashes

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-attribute-hashes"
    }
  }'
```

### 3. Remove Data

Clear mappings or hashes for troubleshooting purposes. This is useful when you need to force re-synchronization of entities.

#### Remove Product Hashes

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-product-hashes"
    }
  }'
```

**Response:**

```json
{
    "status": "success",
    "message": "product-hashes",
    "data": {
        "message": "Hashes removed"
    }
}
```

#### Remove Category Hashes

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-category-hashes"
    }
  }'
```

#### Remove Attribute Hashes

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-attribute-hashes"
    }
  }'
```

#### Remove Mappings

Remove persistent mappings for specific entity types:

```bash
# Remove product mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-product-mappings"
    }
  }'

# Remove category mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-category-mappings"
    }
  }'

# Remove attribute mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-attribute-mappings"
    }
  }'

# Remove asset mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-asset-mappings"
    }
  }'
```

### 4. Save Mappings

Manually create or update entity mappings. This is useful for testing or fixing mapping issues.

#### Save Product Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "save-product-mappings",
      "mappings": [
        {
          "bluestoneId": "product-123",
          "commerceId": 25,
          "commerceCode": "default",
          "data": []
        },
        {
          "bluestoneId": "product-456",
          "commerceId": 26,
          "commerceCode": "german_store_view",
          "data": []
        }
      ]
    }
  }'
```

#### Save Category Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "save-category-mappings",
      "mappings": [
        {
          "bluestoneId": "category-123",
          "commerceId": 15,
          "commerceCode": "default",
          "data": []
        }
      ]
    }
  }'
```

#### Save Attribute Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "save-attribute-mappings",
      "mappings": [
        {
          "bluestoneId": "attribute-123",
          "commerceId": 35,
          "commerceCode": "default",
          "data": []
        }
      ]
    }
  }'
```

#### Save Asset Mappings

```bash
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "save-asset-mappings",
      "mappings": [
        {
          "bluestoneId": "asset-123",
          "commerceId": 45,
          "commerceCode": "default",
          "data": []
        }
      ]
    }
  }'
```

## Understanding the Data

### Mapping Structure

Each mapping contains:

-   **`bluestoneId`**: The unique identifier from Bluestone PIM
-   **`commerceId`**: The corresponding Adobe Commerce entity ID
-   **`commerceCode`**: The Adobe Commerce store view code (optional)
-   **`data`**: Additional metadata stored with the mapping (optional)

### Hash Structure

Hashes are stored as key-value pairs where:

-   **Key**: Format `hash-{entityType}-{entityId}` (e.g., `hash-product-123`)
-   **Value**: MD5 hash of the entity data used for change detection

## Common Debug Scenarios

### 1. Force Re-synchronization

If an entity is not updating properly, you can force re-synchronization by removing its hash:

```bash
# Remove product hashes to force re-sync
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-product-hashes"
    }
  }'
```

### 2. Fix Mapping Issues

If mappings are incorrect, you can remove and recreate them:

```bash
# Remove existing mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "remove-product-mappings"
    }
  }'

# Then trigger a full sync to recreate mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/full-sync/trigger" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "entityType": "product",
      "context": "default"
    }
  }'
```

### 3. Verify Synchronization State

Check if entities are properly mapped and hashed:

```bash
# Check product mappings
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-product-mappings"
    }
  }'

# Check product hashes
curl -X POST "https://your-app-builder-endpoint/api/v1/web/debug-internal/consumer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "data": {
      "type": "get-product-hashes"
    }
  }'
```

## Error Handling

The debug endpoint returns appropriate error responses:

-   **400 Bad Request**: Invalid request type or missing required parameters
-   **500 Internal Server Error**: Server-side errors during processing

Example error response:

```json
{
    "status": "error",
    "message": "Invalid type",
    "code": 400
}
```

## Best Practices

1. **Backup Before Removal**: Always check existing mappings before removing them
2. **Test in Development**: Use debug operations in development environment first
3. **Monitor Logs**: Check App Builder runtime logs for additional debugging information
4. **Use Specific Operations**: Target specific entity types rather than removing all data
5. **Verify Changes**: Always verify the results of debug operations

## Integration with Full Sync

The debug endpoint works well with the [Full Sync Operations](../README.md#full-sync-operations) feature:

1. Use debug to inspect current state
2. Remove problematic hashes or mappings
3. Use full sync to recreate data
4. Verify results with debug operations

This combination provides a powerful toolkit for troubleshooting synchronization issues in the Bluestone integration.
