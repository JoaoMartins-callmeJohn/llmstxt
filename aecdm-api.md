# AEC Data Model API - llms.txt

## Overview

The AEC Data Model API is a GraphQL-based API from Autodesk Platform Services (APS) that provides direct cloud access to granular design data from AEC (Architecture, Engineering, Construction) models without requiring desktop application plugins. The API democratizes access to BIM data by breaking down monolithic files (.rvt, .dwg) into granular, object-level data accessible via the cloud.

**Key Features:**
- GraphQL interface for precise data queries
- Read-only access to Revit elements and properties (as of 2024)
- Cloud-based access to granular design data
- No desktop plugins required
- Supports data from Revit 2024+ models
- Available to Autodesk Docs subscribers in US and EU regions

## Authentication

The AEC Data Model API uses OAuth 2.0 three-legged authentication flow:

1. **Register APS Application**: Create an application in Autodesk Platform Services to get Client ID and Client Secret
2. **OAuth Flow**: Implement three-legged OAuth to obtain access tokens
3. **Access Token**: Include bearer token in Authorization header for all API requests

### Required Headers:
```
Authorization: Bearer {access_token}
Content-Type: application/json
Region: US  # or EU, depending on your account region
```

### Prerequisites:
- Autodesk Platform Services (APS) account
- Autodesk Docs subscription
- AEC Data Model feature activated in Account Admin settings
- Revit 2024+ models uploaded to Autodesk Docs
- Account region must be US or EU (Australia not supported)

## API Endpoint

**GraphQL Endpoint:** Single endpoint for all queries
- **URL Pattern:** `https://developer.api.autodesk.com/aecdatamodel/graphql`
- **Method:** POST
- **Content-Type:** application/json

## Core API Constructs

### 1. Hub
- Top-level container representing an organization or account
- Contains multiple projects
- Has geographic location (US/EU)

### 2. Project  
- Container for design files and folders within a hub
- Corresponds to Autodesk Construction Cloud projects

### 3. Folder
- Organizational structure within projects
- Contains files and sub-folders

### 4. ElementGroup
- Represents a specific version of a design file (e.g., Revit model)
- Contains elements and their properties
- Sometimes referred to as "Model" or "Design"

### 5. Element
- Individual building components (walls, doors, windows, etc.)
- Has properties, references, and geometric data
- Represents the granular data within models

## Available Queries

### Hub & Project Queries
- `hubs` - Get all accessible hubs
- `hub(id: ID!)` - Get specific hub by ID
- `projects` - Get all projects
- `project(id: ID!)` - Get specific project
- `elementGroupsByHub(hubId: ID!)` - Get element groups in a hub
- `elementGroupsByProject(projectId: ID!)` - Get element groups in a project

### Folder Queries
- `folder(id: ID!)` - Get specific folder
- `foldersByProject(projectId: ID!)` - Get folders in a project
- `foldersByFolder(folderId: ID!)` - Get sub-folders
- `elementGroupsByFolder(folderId: ID!)` - Get element groups in folder
- `elementGroupsByFolderAndSubFolders(folderId: ID!)` - Get element groups recursively

### ElementGroup Queries
- `elementGroupAtTip(id: ID!)` - Get latest version of element group
- `elementGroupByVersionNumber(id: ID!, versionNumber: Int!)` - Get specific version
- `elementGroupExtractionStatus(id: ID!)` - Check data extraction status
- `elementGroupExtractionStatusAtTip(id: ID!)` - Check extraction status of latest version

### Element Queries
- `elementAtTip(id: ID!)` - Get specific element from latest version
- `elementsByHub(hubId: ID!)` - Get elements across hub
- `elementsByProject(projectId: ID!)` - Get elements across project
- `elementsByFolder(folderId: ID!)` - Get elements in folder
- `elementsByElementGroup(elementGroupId: ID!)` - Get elements in element group
- `elementsByElementGroupAtVersion(elementGroupId: ID!, versionNumber: Int!)` - Get elements from specific version

### Property Queries
- `distinctPropertyValuesInElementGroupById(elementGroupId: ID!)` - Get unique property values
- `distinctPropertyValuesInElementGroupByName(elementGroupId: ID!)` - Get unique values by name
- `propertyDefinitionsByElementGroup(elementGroupId: ID!)` - Get property schema

## Common Query Patterns

### Basic Hub and Project Query
```graphql
query GetHubs {
  hubs {
    results {
      id
      name
      region
    }
  }
}

query GetProjects($hubId: ID!) {
  elementGroupsByHub(hubId: $hubId) {
    results {
      id
      name
      properties {
        results {
          name
          value
        }
      }
    }
  }
}
```

### Elements by Category with Filtering
```graphql
query GetElementsFromCategory($elementGroupId: ID!, $propertyFilter: String!) {
  elementsByElementGroup(
    elementGroupId: $elementGroupId, 
    filter: {query: $propertyFilter},
    pagination: {limit: 500}
  ) {
    pagination {
      cursor
      hasNext
    }
    results {
      id
      name
      properties {
        results {
          name
          value
          definition {
            units {
              name
            }
          }
        }
      }
    }
  }
}
```

### Elements with Type References
```graphql
query GetElementsWithTypes($elementGroupId: ID!) {
  elementsByElementGroup(elementGroupId: $elementGroupId) {
    results {
      id
      name
      properties {
        results {
          name
          value
        }
      }
      referencedBy(name: "Type") {
        results {
          id
          name
          properties {
            results {
              name
              value
            }
          }
        }
      }
    }
  }
}
```

## Filtering Options

### Property Filters
- `property.name.category==Walls` - Elements in Walls category
- `property.name.category=contains=Doors` - Category contains "Doors"
- `'property.name.Level'=exists=true` - Elements with Level property
- `'property.name.Level'='Level 1'` - Elements on specific level

### Filter Operators
- `==` - Equals
- `!=` - Not equals  
- `=contains=` - Contains substring
- `=exists=` - Property exists
- `=startsWith=` - Starts with
- `=endsWith=` - Ends with

### Category Examples
Common categories for filtering:
- Walls
- Windows  
- Floors
- Doors
- Furniture
- Ceilings
- Electrical Equipment
- Structural Framing
- MEP equipment

## Pagination

### Default Limits
- Default page size: 50 items
- Maximum page size: 1000 items
- Use cursor-based pagination for large datasets

### Pagination Example
```graphql
query GetElementsPaginated($elementGroupId: ID!, $cursor: String) {
  elementsByElementGroup(
    elementGroupId: $elementGroupId,
    pagination: {limit: 100, cursor: $cursor}
  ) {
    pagination {
      cursor
      hasNext
    }
    results {
      id
      name
    }
  }
}
```

## Rate Limits

- **Default Rate Limit:** 6000 points per minute per application
- **Query Limit:** 1000 points per individual query
- **Point Values:** Different operations consume different point values
- Contact support for higher rate limits if needed

## Error Handling

### Common HTTP Status Codes
- `200` - Success
- `400` - Bad Request (invalid query syntax)
- `401` - Unauthorized (invalid or expired token)
- `403` - Forbidden (insufficient permissions)
- `429` - Rate limit exceeded
- `500` - Internal server error

### GraphQL Error Response
```json
{
  "errors": [
    {
      "message": "Error description",
      "locations": [{"line": 2, "column": 3}],
      "path": ["field", "name"]
    }
  ],
  "data": null
}
```

## Data Structure Examples

### Element Properties Structure
```json
{
  "id": "element-id",
  "name": "Wall-001",
  "properties": {
    "results": [
      {
        "name": "Category",
        "value": "Walls",
        "definition": {
          "units": null
        }
      },
      {
        "name": "Length",
        "value": 12.5,
        "definition": {
          "units": {
            "name": "feet"
          }
        }
      }
    ]
  }
}
```

### Element References
Elements can reference other elements (e.g., instances reference types):
```json
{
  "referencedBy": {
    "results": [
      {
        "id": "type-id",
        "name": "Basic Wall Type",
        "properties": {
          "results": [...]
        }
      }
    ]
  }
}
```

## Best Practices

### Query Optimization
1. Request only needed fields to minimize response size
2. Use filters to limit results at the server level
3. Implement pagination for large datasets
4. Cache frequently used data locally
5. Use property filters instead of client-side filtering

### Error Handling
1. Always check for GraphQL errors in responses
2. Implement retry logic for rate limit errors
3. Handle authentication token expiration
4. Validate query syntax before sending

### Performance Tips
1. Batch related queries when possible
2. Use appropriate pagination limits
3. Monitor rate limit consumption
4. Consider using persisted queries for repeated operations

## Common Use Cases

### Quality Control & Reporting
- Find elements missing required properties
- Generate quantity takeoffs and schedules
- Create compliance reports
- Identify design anomalies

### Data Analysis
- Compare element properties across versions
- Generate insights from property data
- Create custom dashboards
- Export data for external analysis

### Integration & Automation
- Sync data with external systems
- Automate repetitive tasks
- Create custom workflows
- Build specialized applications

## Limitations

- **Read-only access:** Currently supports only data retrieval, not modification
- **File format support:** Limited to Revit 2024+ models initially
- **Geometry:** Limited geometry support in current version
- **Regional availability:** US and EU regions only
- **File processing:** AEC Data Model generation required for uploaded files

## Development Tools

### AEC Data Model Explorer
- GraphiQL-based explorer for testing queries
- Integrated with Autodesk Viewer
- Step-by-step tutorials available
- Schema exploration capabilities

### Code Samples
- .NET samples available on GitHub
- Python examples with aps-toolkit
- Tutorial applications with full workflows
- Community examples and integrations

## Getting Started Checklist

1. ✅ Set up APS application with Client ID/Secret
2. ✅ Activate AEC Data Model in Account Admin settings  
3. ✅ Upload Revit 2024+ models to Autodesk Docs
4. ✅ Wait for data model generation to complete
5. ✅ Implement OAuth 2.0 authentication
6. ✅ Test queries using AEC Data Model Explorer
7. ✅ Build application with appropriate error handling
8. ✅ Implement pagination and rate limit handling

## Support Resources

- **Documentation:** Autodesk Platform Services Developer Portal
- **Tutorials:** Step-by-step AEC Data Model tutorial
- **Community:** APS developer forums and community
- **Code Samples:** GitHub repositories with example implementations
- **Support:** APS support channel for technical assistance