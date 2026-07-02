# Sprint 1

## 📌 Global Conventions

- **Architectural Note:**This system only supports**GET**and **POST **methods; **PUT **and **DELETE **are intentionally excluded.
- **Base URL:** `/api` (e.g., `http://localhost:8080/lims/api`)
- **Headers:** `Content-Type: application/json` is required for all `POST` requests.
- **Authentication (Sprint 1 Bypass):** No JWT validation for this sprint. For any database fields requiring user tracking (e.g., `created_by`, `updated_by`), the backend will **hardcode** the value as `"SYS_ADMIN_001"`.
- **Global Response Wrapper:** All backend responses, regardless of success or failure, will be wrapped in the following standard `RespResult<T>` structure:

```json

{
  "responseCode": 200,  // Integer HTTP or Custom Business Code
  "message": "Success", // String message for debugging or UI toasts
  "data": { ... }       // Generic payload <T> (Object, Array, or null)
}
```

## 🧭 API Endpoints

### 1. Create Test Type

- **Description:** Saves a new test type form along with its parameters. The backend must open a database transaction to insert into both `Test_Type` and `Parameter` tables.
- **Method:** `POST`
- **URL:** `/api/test-types`
- **Request Body (JSON):**

```json
{
  "typeName": "ICP",
  "requiredVolume": 500.0,
  "description": "Inductively Coupled Plasma analysis for metal detection",
  "bgColor": "Purple",
  "iconColor": "Purple",
  "borderColor": "Purple",
  "isActive": 1,
  "parameters": [
    {
      "parameterName": "Li DM",
      "unit": "mg/L"
    },
    {
      "parameterName": "Be DM",
      "unit": "mg/L"
    }
  ]
}
```

- **Success Response** (`200 OK`):

```json
{
  "responseCode": 200,
  "message": "Test Type created successfully",
  "data": {
    "testTypeId": "TT-001"
  }
}
```

### 2. Get Test Type List

- **Description:** Fetches all existing test types ALONG WITH their associated parameters to display as detailed cards on the UI homepage.
- **Architect Note (For Backend):** ⚠️ **CRITICAL:** Do NOT use the N+1 query approach (executing a separate SQL query inside a `for` loop for each test type). You MUST execute a single `LEFT JOIN` SQL query to fetch data from both tables simultaneously. Then, in your Java Servlet/DAO, iterate through the flat JDBC `ResultSet` and group (fold) the duplicated `Test_Type` rows into a single `TestTypeDTO` that contains a `List<ParameterDTO>`.
- **Method:** `GET`
- **URL:** `/api/test-types`
- **Request Body:** None
- **Success Response (`HTTP 200 OK`):**

```json
{
  "responseCode": 200,
  "message": "Data retrieved successfully",
  "data": [
    {
      "testTypeId": "TT-001",
      "typeName": "ICP",
      "requiredVolume": 500.0,
      "description": "Inductively Coupled Plasma analysis for metal detection",
      "bgColor": "Purple",
      "iconColor": "Purple",
      "borderColor": "Purple",
      "isActive": 1,
      "parameters": [
        {
          "parameterId": 101,
          "parameterName": "Li DM",
          "unit": "mg/L"
        },
        {
          "parameterId": 102,
          "parameterName": "Be DM",
          "unit": "mg/L"
        }
      ]
    },
    {
      "testTypeId": "TT-002",
      "typeName": "IC",
      "requiredVolume": 100.0,
      "description": "Ion Chromatography for anion analysis",
      "bgColor": "Blue",
      "iconColor": "Blue",
      "borderColor": "Blue",
      "isActive": 1,
      "parameters": [
        {
          "parameterId": 201,
          "parameterName": "F",
          "unit": "ppm"
        },
        {
          "parameterId": 202,
          "parameterName": "Cl",
          "unit": "ppm"
        }
      ]
    }
  ]
}
```

### 3. Get Test Type Detail

- **Description:** Fetches full details of a single test type (including all its parameters) to populate the Edit form when a user clicks "Edit".
- **Architect Note** (For Frontend): Why fetch again instead of using the existing data from the List view?
- 1. **Data Freshness:** Ensures the user is editing the most up-to-date record, preventing data conflicts if another admin modified it recently.
- 2. **Deep Linking / Page Refresh:** Allows the Edit page to be accessed directly via URL or refreshed without crashing, as it does not rely on the parent list component's memory state.
- **Method:** `GET`
- **URL:** `/api/test-types/{id}` _(e.g., `/api/test-types/TT-001`)_
- **Request Body:** None
- **Success Response (`HTTP 200 OK`):**

```json
{
  "responseCode": 200,
  "message": "Data retrieved successfully",
  "data": {
    "testTypeId": "TT-001",
    "typeName": "ICP",
    "requiredVolume": 50.0,
    "description": "Inductively Coupled Plasma analysis for metal detection",
    "bgColor": "Green",
    "iconColor": "Purple",
    "borderColor": "Purple",
    "isActive": 1,
    "parameters": [
      {
        "parameterId": 101,
        "parameterName": "Li DM",
        "unit": "mg/L"
      },
      {
        "parameterId": 102,
        "parameterName": "Be DM",
        "unit": "mg/L"
      }
    ]
  }
}
```

### 4. Update Test Type

- **Description:** Overwrites an existing test type and its parameter list.
- **Architect Note (For Backend):** Use the **"Delete & Re-insert"** strategy inside a single JDBC transaction:
  1. `UPDATE` the main `Test_Type` row using the `{id}` from the URL.
  2. `DELETE` all existing rows in the `Parameter` table where `test_type_id = {id}`.
  3. Loop through the incoming `parameters` JSON array and `INSERT` them as brand-new records.
- **Method:** `POST`
- **URL:** `/api/test-types/{id}` _(e.g., `/api/test-types/TT-001`)_
- **Request Body (JSON):**
  ```
  {
    "typeName": "ICP-Modified",
    "requiredVolume": 600.00,
    "description": "Updated description text...",
    "bgColor": "Red",
    "iconColor": "Red",
    "borderColor": "Red",
    "isActive": 1,
    "parameters": [
      {
        "parameterName": "Li DM",
        "unit": "ppm"
      },
      {
        "parameterName": "New Element",
        "unit": "mg/L"
      }
    ]
  }
  ```
- **Success Response** (`HTTP 200 OK`):

```json
{
  "responseCode": 200,
  "message": "Test Type updated successfully",
  "data": null
}
```
