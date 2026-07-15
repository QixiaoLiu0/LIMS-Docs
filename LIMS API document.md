# Sprint 1

## 📌 Global Conventions

- **Architectural Note:** This system only supports **GET** and **POST** methods; **PUT** and **DELETE** are intentionally excluded.
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
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). (`ADMIN`, `SUPER_ADMIN`).
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
      "unit": "mg/L",
      "limit": "<1.0"
    },
    {
      "parameterName": "Be DM",
      "unit": "mg/L",
      "limit": "<2.0"
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
    "testTypeId": "1"
  }
}
```

### 2. Get Test Type List

- **Description:** Fetches all existing test types ALONG WITH their associated parameters to display as detailed cards on the UI homepage.
- **Architect Note (For Backend):** ⚠️ **CRITICAL:** Do NOT use the N+1 query approach (executing a separate SQL query inside a `for` loop for each test type). You MUST execute a single `LEFT JOIN` SQL query to fetch data from both tables simultaneously. Then, in your Java Servlet/DAO, iterate through the flat JDBC `ResultSet` and group (fold) the duplicated `Test_Type` rows into a single `TestTypeDTO` that contains a `List<ParameterDTO>`.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). (`ADMIN`, `SUPER_ADMIN`).
- **Method:** `GET`
- **URL:** `/api/test-types`
- **Request Body:** Non
- **Success Response (`HTTP 200 OK`):**

```json
{
  "responseCode": 200,
  "message": "Data retrieved successfully",
  "data": [
    {
      "testTypeId": "1",
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
          "unit": "mg/L",
          "limit": "<1.0"
        },
        {
          "parameterId": 102,
          "parameterName": "Be DM",
          "unit": "mg/L"
          "limit": "<2.0"
        }
      ]
    },
    {
      "testTypeId": "2",
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
          "limit": "<30"
        },
        {
          "parameterId": 202,
          "parameterName": "Cl",
          "unit": "ppm",
          "limit": "<50"
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
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). (`ADMIN`, `SUPER_ADMIN`).
- **Method:** `GET`
- **URL:** `/api/test-types/{id}` _(e.g., `/api/test-types/1`)_
- **Request Body:** None
- **Success Response (`HTTP 200 OK`):**

```json
{
  "responseCode": 200,
  "message": "Data retrieved successfully",
  "data": {
    "testTypeId": "1",
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
        "unit": "mg/L",
        "limit": "<1.0"
      },
      {
        "parameterId": 102,
        "parameterName": "Be DM",
        "unit": "mg/L",
        "limit": "<2.0"
      }
    ]
  }
}
```

### 4. Update Test Type

- **Description:** Updates an existing test type and performs a precise, differential update (Diffing) on its parameter list to preserve historical data integrity (Foreign Key constraints).
- **Note (For Backend & Frontend):** \* **Frontend:** To update an existing parameter, you **MUST** include its original `parameterId`. To add a brand-new parameter, omit the `parameterId` (or pass `null`). If you remove a parameter from this array, the backend will treat it as a deletion request.
  - **Backend:** Use the "Diffing Algorithm" inside a single JDBC transaction:
    1. `UPDATE` the main `Test_Type` row using the `{id}` from the URL.
    2. Loop through the incoming `parameters` JSON array.
    3. If `parameterId` exists **$\rightarrow$** `UPDATE` existing parameter.
    4. If `parameterId` is null/missing **$\rightarrow$** `INSERT` new parameter.
    5. If an existing DB parameter is missing from the JSON **$\rightarrow$** `DELETE` it (catch FK exceptions gracefully).
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). (`ADMIN`, `SUPER_ADMIN`).
- **Method:** **POST**
- **URL:** `/api/test-types/{id}` _(e.g., `/api/test-types/1`)_
- **Request Body (JSON):**

```JSON
{
  "testTypeId": 1,
  "typeName": "ICP-Modified",
  "requiredVolume": 600.00,
  "description": "Updated description text with differential parameters...",
  "bgColor": "Red",
  "iconColor": "Red",
  "borderColor": "Red",
  "isActive": 1,
  "parameters": [
    {
      "parameterId": 101,
      "parameterName": "Li DM",
      "unit": "ppm",
      "limit": "<1.5"
    },
    {
      "parameterName": "New Element",
      "unit": "mg/L",
      "limit": "<2.0"
    }
  ]
}
```

(💡 **JSON Note:** Notice the first parameter includes `"parameterId": 101`, signaling an **UPDATE** to an existing row. The second parameter omits `parameterId`, signaling a brand-new **INSERT** . Any ID not sent back will be **DELETED** .)

- **Success Response ( `HTTP 200 OK` ):**

```JSON
{
  "responseCode": 200,
  "message": "Test Type updated successfully",
  "data": null
}
```

# Sprint 2

## User Login

- **Description:** Validates user credentials against the database. Upon successful authentication, generates and issues a cryptographically signed JSON Web Token (JWT) encapsulated with the user's identity and system role.
- **Note (For Backend):** This specific endpoint **MUST** be added to the `WHITELIST` inside `JwtAuthFilter` to bypass authentication checks. Password verification should handle incoming plain text for initial testing, but must pivot to secure hashing (e.g., BCrypt or SHA-256) upon final database integration.
- **Method:** `POST`
- **URL:** `/api/auth/login`
- **Request Body (JSON):**

```JSON
{
  "email": "admin@test.com",
  "password": "secure_password_123"
}
```

- **Success Response ( HTTP 200 OK ):**

```JSON
{
  "responseCode": 200,
  "message": "Login successful",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEsImVtYWlsIjoiY2hyaXMubGl1QHNhaXQuY2EiLCJyb2xlIjoiQURNSU4ifQ...",
    "userId": 1,
    "email": "chris.liu@sait.ca",
    "role": "SUPER_ADMIN"
  }
}
```

(💡 **Frontend Integration Note:** The client-side application is responsible for intercepting this successful response, extracting the `token` string, and persisting it securely within local storage (`localStorage` or `sessionStorage`). For all subsequent non-whitelist API calls, the frontend must attach this token inside the HTTP header formatted exactly as `Authorization: Bearer \_.)

## Modify Password

- **Description:** Modifies the password of the currently authenticated user.
- **Architect Note (Security Defense):** This endpoint is highly critical and **MUST NOT** be added to the filter whitelist. To prevent ID-tampering and Horizontal Privilege Escalation vulnerabilities, **the JSON request payload deliberately omits the user's ID** .
  Once the request successfully passes through the `JwtAuthFilter`, the downstream `Controller` and `Service` layers must fetch the active user's identity directly from the thread-bound context execution pool via `UserContext.getUserId()`.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header).(All authenticated roles are permitted.)
- **Method:** **POST**
- **URL:** `/api/auth/password`
- **Request Body (JSON):**

```JSON
{
  "oldPassword": "secure_password_123",
  "newPassword": "brand_new_secure_password_456"
}
```

- **Success Response ( `HTTP 200 OK` ):**

```JSON
{
  "responseCode": 200,
  "message": "Password updated successfully. Please log in again with your new credentials.",
  "data": null
}
```

## User Logout

**ArchitectNote:** We don't actually need a logout endpoint on the backend. You can just handle it entirely on the frontend by clearing the token from your `localStorage` or `sessionStorage` and redirecting the user back to the login page. That will fully close the loop for logging out.

# Sprint 3

## 1. Get Active Test Types (Lookup)

- **Description:** Retrieves a lightweight dictionary list of all currently active test types. This endpoint is designed specifically to populate frontend tag selectors during COC creation or sample test assignment. It performs a single-table query on `Test_Type` (`WHERE is_active = 1`) and strictly avoids joining underlying parameters to prevent over-fetching.
- **Authentication:** Required (Valid JWT in `Authorization Bearer <token>`). All authenticated roles are permitted.
- **Method:** `GET`
- **URL:** `/api/lookup/test-types`
- **Request Body (JSON):** _(None required for this GET request)_
- **Success Response:**
 ```JSON
  {
    "responseCode": 200,
    "message": "Success",
    "data": [
      {
        "testTypeId": 1,
        "typeName": "ICP",
        "description": "Inductively Coupled Plasma",
        "requiredVolume": 300.00,
        "bgColor": "#F0F0F0",
        "iconColor": "#333333",
        "borderColor": "#CCCCCC"
      },
      {
        "testTypeId": 2,
        "typeName": "IC",
        "description": "Ion Chromatography",
        "requiredVolume": 250.00,
        "bgColor": "#E8F4F8",
        "iconColor": "#005577",
        "borderColor": "#BBDDFF"
      },
      {
        "testTypeId": 3,
        "typeName": "Alkalinity",
        "description": "Total Alkalinity Test",
        "requiredVolume": 100.00,
        "bgColor": "#E6F4EA",
        "iconColor": "#117733",
        "borderColor": "#AADDCC"
      }
    ]
  }
  ```

## 2. Get Sample Types (Lookup)

- **Description:** Retrieves a dictionary list of all available sample categories (e.g., Water, Soil, Gas, Oil). This endpoint queries the Sample_Type table and is used to populate the sample type dropdown menus during COC and sample creation.
* **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
* **Method:** `GET`
* **URL:** `/api/lookup/sample-types`
* **Request Body (JSON):** _(None required for this GET request)_
* **Success Response:**
 ```JSON
  {
    "responseCode": 200,
    "message": "Success",
    "data": [
      {
        "sampleTypeId": 1,
        "sampleTypeName": "Water"
      },
      {
        "sampleTypeId": 2,
        "sampleTypeName": "Soil"
      },
      {
        "sampleTypeId": 3,
        "sampleTypeName": "Oil"
      },
      {
        "sampleTypeId": 4,
        "sampleTypeName": "Gas"
      }
    ]
  }
  ```
## 3. Delete COC (Cascade Physical Deletion)
- **Description:** Permanently deletes a Chain of Custody (COC) record based on the provided `cocId` in the path. To prevent foreign key constraint violations, the backend service explicitly executes a reverse-order cascading physical deletion: it first removes all associated `Result` placeholders **(the pre-populated `value=null` rows generated during COC/Sample creation)**, then Test assignments, then `Sample` records, and finally the `COC` root record. This action is irreversible.
- **Note:** Front-end must send an empty JSON object {} as the request body to avoid backend JSON parsing exceptions.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `POST`
- **URL:** `/api/cocs/{cocId}/delete`
- **Request Body (JSON):** 
```JSON
  {}
```
- **Success Response:**
 ```JSON
  {
    "responseCode": 200,
    "message": "COC and all related tree data permanently deleted.",
    "data": null
  }
  ```
## 4. Create COC (Hierarchical Aggregate Creation)
- **Description:** Creates a single Chain of Custody (COC) record in one atomic database transaction. To support flexible laboratory workflows, this endpoint accepts varying depths of nested arrays, resulting in three distinct creation states:
1. **Empty COC:**
   - **Action:** Populates only the `COC` root table.
   - **JSON Structure:** The payload contains COC-level fields, and the samples array is explicitly empty (i.e., `"samples": []`).

2. **Semi-empty COC:**
   - **Action:** Populates the `COC` and `Sample` tables.
   - **JSON Structure:** The payload contains COC fields and one or more sample objects, but the test assignments are explicitly empty (i.e., `"testTypeIds": []`).

3. **Fully populated COC:**
   - **Action:** Populates the `COC`, `Sample`, and `Test` tables. Additionally, it triggers the automatic parameter blueprint fetch and pre-populates the `Result` table with placeholder rows (`value = null`).
   - **JSON Structure:** The payload contains COC fields, sample objects, and populated test assignments (e.g., `"testTypeIds": [1, 2]`).
- **Security & Audit Note:** The `created_by_user_id` required for creating the `COC` root record (and the subsequent `Result` table placeholders) is securely extracted from the backend's ThreadLocal JWT context. The frontend must not pass any user ID in the payload.
- **Business Status Rule:** Any `COC`, `Sample`, or `Test` record generated during this transaction will automatically have its `status` initialized to `'In-Progress'` by the backend.

- **Data Contract & Fault Tolerance Rule:**
Due to strict ERD NOT NULL constraints and to ensure backend DTO fault tolerance, the frontend MUST pass all fields shown in the example payload below. If a field is currently unused or empty in the UI (e.g., `reportToName`, `reportToEmail`), it must be passed as an empty string `""`. Do not omit the key and do not pass `null`.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `POST`
- **URL:** `/api/cocs`
- **Request Body (JSON):**
``` JSON
{
  "cocNumber": "COC-2026-001",
  "projectName": "Calgary Water Quality Test",
  "reportToName1": "",
  "reportToEmail1": "",
  "reportToName2": "",
  "reportToEmail2": "",
  "dateRequired": "2026-07-20 17:00:00",
  "isRush": 0,
  "dateForRush": "",
  "receivedBy": "",
  "receivedTime": "",
  "relinquishedBy": "John",
  "relinquishedTime": "2026-07-14 11:00:00",
  "numberOfContainers": 10,
  "specialInstructions": "Keep refrigerated below 4°C.",
  "samples": [
    {
      "sampleTypeId": 1,
      "sampleClientId": "S_DW_01",
      "sampledTime": "2026-07-14 10:30:00",
      "samplingPoint": "Inlet Pipe A",
      "matrix": "Drinking Water",
      "numberOfContainers": 2,
      "remarks": "",
      "initialVolume": 500.00,
      "remainingVolume": 500.00,
      "isFiltered": 0,
      "isPreserved": 1,
      "isFilteredAndPreserved": 0,
      "testTypeIds": [1, 2] 
    },
    {
      "sampleTypeId": 1,
      "sampleClientId": "S_DW_02",
      "sampledTime": "2026-07-14 11:00:00",
      "samplingPoint": "Outlet Pipe B",
      "matrix": "Drinking Water",
      "numberOfContainers": 1,
      "remarks": "Slightly cloudy",
      "initialVolume": 250.00,
      "remainingVolume": 250.00,
      "isFiltered": 1,
      "isPreserved": 0,
      "isFilteredAndPreserved": 0,
      "testTypeIds": [] 
    }
  ]
}
```
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "COC created successfully.",
  "data": {
    "cocId": "c8f8b9e2-5a12-4c8d-b9f0-e7a8c9b0d1e2"
  }
}
```
## 5. Append Sample to COC
- **Description:** Appends a single Sample physical record to an existing Chain of Custody (COC) identified by the cocId in the path. This endpoint is strictly designed for `sample` entity creation only, it inherently prohibits any cascading insertion into the `Test` or `Result` tables. 
- **Data Contract & Fault Tolerance Rule:**
The request payload accepts a single JSON object (not an array). Due to strict ERD NOT NULL constraints, the frontend **MUST** pass all fields shown in the example payload below. Unused fields must be submitted as an empty string `""` to ensure DTO fallback safety. Do not pass null.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Business Status Rule:** The newly created `Sample` record will automatically have its `status` initialized to `'In-Progress'` by the backend.
- **Method:** `POST`
- **URL:** `/api/cocs/{cocId}/samples`
- **Request Body (JSON):**
``` json
{
  "sampleTypeId": 1,
  "sampleClientId": "S_DW_03",
  "sampledTime": "2026-07-14 12:00:00",
  "samplingPoint": "Inlet Pipe C",
  "matrix": "Drinking Water",
  "numberOfContainers": 1,
  "remarks": "",
  "initialVolume": 500.00,
  "remainingVolume": 500.00,
  "isFiltered": 0,
  "isPreserved": 0,
  "isFilteredAndPreserved": 0
}
```
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "Sample added successfully.",
  "data": {
    "sampleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  }
}
```

## 6. Append Test to Sample (Dynamic Run & Pre-population)
- **Description:** Assigns a new test task to an existing sample (identified by sampleId in the path). This endpoint handles two heavy backend operations within a single database transaction:
  - **Dynamic Run Calculation:** Before inserting into the `Test` table, the backend queries historical records for the same `sampleId` and `testTypeId` to automatically calculate and assign the run_number (e.g., if it's the first time, it assigns 1).
  - **Placeholder Pre-population:** Immediately after creating the `Test` record, the backend fetches all parameter blueprints mapped to this `testTypeId` and cascade-inserts placeholder rows (`value = null`) into the Result table.
  - **Security & Audit Note:** The `created_by_user_id` required for the `Result` table generation is securely extracted from the backend's ThreadLocal JWT context. The frontend must not pass any user ID in the payload.
- **Business Status Rule:** The newly created `Test` record will automatically have its `status` initialized to `'In-Progress'` by the backend.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `POST`
- **URL:** `/api/samples/{sampleId}/tests`
- **Request Body (JSON):**
``` json
{
  "testTypeId": 1
}
```
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "Test assigned successfully with Result table placeholders generated.",
  "data": {
    "testId": "a9b8c7d6-e5f4-3c2b-1a09-876543210fed",
    "runNumber": 1
  }
}
```

## 7. Delete Sample (Cascade Physical Deletion)
- **Description:** Permanently deletes a Sample physical record and its entire downstream hierarchy based on the provided sampleId in the path. To prevent foreign key constraint violations, the backend service must execute a strict reverse-order cascading deletion within a single database transaction:
  - Identifies all `test_id`s associated with the `sampleId`.
  - Deletes all related data and placeholders from the `Result` table.
  - Deletes the assigned tasks from the `Test` table.
  - Finally, deletes the root `Sample` record itself.
  
  **Note:** This action is irreversible. The frontend must send an empty JSON object `{}` as the request body to satisfy `POST` request JSON parsing requirements.

- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `POST`
- **URL:** `/api/samples/{sampleId}/delete`
- **Request Body (JSON):**
``` json
  {}
```
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "Sample and all associated downstream tests/results permanently deleted.",
  "data": null
}
```
## 8. Delete Test Task (Cascade Physical Deletion)
- **Description:** Permanently deletes a single test task (`Tes`t record) identified by the `testId` in the path. To prevent foreign key constraint violations, the backend service must execute a cascading deletion within a single database transaction: first, physically delete all associated rows in the `Result` table (the pre-populated parameter placeholders mapped to this `test_id`), and then delete the target record in the `Test` table.
- **Note:** This action is irreversible. The frontend must send an empty JSON object {} as the request body to satisfy backend JSON parsing requirements.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `POST`
- **URL:** `/api/tests/{testId}/delete`
- **Request Body (JSON):**
``` json
  {}
```
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "Test task and its associated result placeholders permanently deleted.",
  "data": null
}
```

## 9. Get Test Results (Placeholder Retrieval)
- **Description:** Retrieves all result rows (both populated values and empty placeholders) for a specific test task, identified by the testId in the path. Because placeholders are automatically generated during test assignment, the backend simply executes a JOIN query between the Result table and the Parameter blueprint table. This provides the frontend with both the data entry fields (resultId, value, qualifier) and the necessary display metadata (parameterName, unit, limit).
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `GET`
- **URL:** `/api/tests/{testId}/results`
- **Request Body (JSON):** _(None required for this GET request)_
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "Success",
  "data": [
    {
      "resultId": "e1f2g3h4-5678-90ab-cdef-1234567890ab",
      "parameterId": 101,
      "parameterName": "Calcium",
      "unit": "mg/L",
      "limit": "50",
      "value": null,
      "qualifier": ""
    },
    {
      "resultId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "parameterId": 102,
      "parameterName": "Magnesium",
      "unit": "mg/L",
      "limit": "30",
      "value": null,
      "qualifier": ""
    },
    {
      "resultId": "f9e8d7c6-b5a4-3210-fedc-ba0987654321",
      "parameterId": 103,
      "parameterName": "pH",
      "unit": "pH units",
      "limit": "6.5 - 8.5",
      "value": "7.2",
      "qualifier": ""
    }
  ]
}
```
## 10. Batch Save Test Results (Update Placeholders)
- **Description:** Saves or submits the laboratory results for a specific test task. Because result placeholders are already pre-populated, this endpoint performs strictly lightweight `UPDATE` operations to avoid `INSERT` overhead and row-lock contention.
Furthermore, this endpoint serves a dual purpose based on the frontend's interaction, controlled by the `isComplete` boolean flag:
  - (**Save as Draft** - `isComplete: false`): Performs only the lightweight `UPDATE` on the `Result` placeholders. It does not alter any business status.
  - (**Submit** - `isComplete: true`): Updates the results AND triggers a hierarchical status rollup within the same database transaction. It updates the current `Test` status to `'Completed'`, then checks if all sibling tests under the parent `Sample` are completed. If yes, it bubbles up to update the `Sample` status to `'Completed'`, and repeats the check for the grandparent `COC` level.

- **Security & Audit Note:** The `updated_by_user_id` is securely extracted from the backend's ThreadLocal JWT context and injected into the SQL statement. The frontend must not pass user IDs in the payload.
- **Data Contract & Fault Tolerance Rule:**
  - The frontend only needs to send an array of the results that are being modified. The qualifier field (e.g., `<`, `>`) is strictly NOT NULL in the database; if there is no qualifier, pass an empty string `""`. If a user clears a previously entered value, also pass `""` for the value field.
  - When the user clicks the **`"Save as Draft"`** button, pass `"isComplete": false`; when the user clicks the **`"Submit"`** button, pass `"isComplete": true`.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `POST`

- **URL:** `/api/tests/{testId}/results/save`

- **Request Body (JSON):**
``` json 
{
  "isComplete": true,
  "results": [
    {
      "resultId": "e1f2g3h4-5678-90ab-cdef-1234567890ab",
      "value": "45.2",
      "qualifier": ""
    },
    {
      "resultId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "value": "0.05",
      "qualifier": "<"
    },
    {
      "resultId": "f9e8d7c6-b5a4-3210-fedc-ba0987654321",
      "value": "7.2",
      "qualifier": ""
    }
  ]
}
```
- **Success Response:**
``` json
{
  "responseCode": 200,
  "message": "Test results processed successfully.",
  "data": null
}
```

## 11. Get COCs for Dashboard (Dashboard Simplified Card)
- **Description:**
Retrieves the COC list used for the homepage Dashboard display. To ensure above-the-fold loading performance, this API uses a dedicated lightweight DTO, returning only the key information required by the UI card (COC metadata, sample summary). It directly calculates the total number of test tasks (`totalTests`) and completed tasks (`completedTests`) under the document through backend **SQL aggregation**. At the current stage, all data is returned in full, without pagination.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `GET`
- **URL:** `/api/cocs`
- **Request Body:** `None`
- **Success Response (JSON):**
``` json
{
  "responseCode": 200,
  "message": "Dashboard COCs retrieved successfully.",
  "data": [
    {
      "cocId": "c8f8b9e2-5a12-4c8d-b9f0-e7a8c9b0d1e2",
      "cocNumber": "COC-2024-001",
      "projectName": "City Water Department",
      "status": "Completed",
      "receivedTime": "2024-01-15 09:30:00",
      "dateRequired": "2024-01-25 17:00:00",
      "totalTests": 40,
      "completedTests": 40,
      "samples": [
        {
          "sampleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "sampleClientId": "S_DW_01",
          "matrix": "Drinking Water"
        },
        {
          "sampleId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
          "sampleClientId": "S_DW_02",
          "matrix": "Drinking Water"
        },
        {
          "sampleId": "1b9d6bcd-bbfd-4b2d-9b5d-ab8dfbbd4bed",
          "sampleClientId": "S_DW_03",
          "matrix": "Drinking Water"
        }
      ]
    },
    {
      "cocId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "cocNumber": "COC-2024-002",
      "projectName": "ABC Manufacturing",
      "status": "In-Progress",
      "receivedTime": "2024-01-18 14:15:00",
      "dateRequired": null,
      "totalTests": 24,
      "completedTests": 15,
      "samples": [
        {
          "sampleId": "f9e8d7c6-b5a4-3210-fedc-ba0987654321",
          "sampleClientId": "S_IE_01",
          "matrix": "Industrial Effluent"
        },
        {
          "sampleId": "d1c2b3a4-0987-6543-21ef-dcba09876543",
          "sampleClientId": "S_IE_02",
          "matrix": "Industrial Effluent"
        },
        {
          "sampleId": "5e4d3c2b-1a09-8765-4321-fedcba098765",
          "sampleClientId": "S_PW_01",
          "matrix": "Process Water"
        }
      ]
    }
  ]
}
```
## 12. Get COC Details (Full Header and Sample Summary)
- **Description:** Retrieves the detailed information of a single Chain of Custody (COC) document, which is used to render the COC details page. The response body of this API is divided into two levels:

  - Root (COC Level): Returns all fields of the COC table in strict accordance with the ERD. This not only meets the rendering requirements of the current UI header but also provides maximum flexibility for the frontend to expand display fields at any time in the future, avoiding frequent modifications to the backend DTO.

  - Nested (Samples Array): Returns the list of samples attached to this document. To ensure network transmission performance, this array only extracts key business and physical fields from the `Sample` table (such as `sampleClientId`, `matrix`, `samplingPoint`, `sampledTime`, `initial/remaining volume`), and calculates the total number of test tasks (totalTests) under each sample through backend SQL aggregation.
- **Authentication:** Required (Valid JWT in `Authorization: Bearer <token>` header). All authenticated roles are permitted.
- **Method:** `GET`
- **URL:** `/api/cocs/{cocId}`
- **Request Body:** `None`
- **Success Response (JSON):**
``` json
{
  "responseCode": 200,
  "message": "COC details retrieved successfully.",
  "data": {
    "cocId": "c8f8b9e2-5a12-4c8d-b9f0-e7a8c9b0d1e2",
    "cocNumber": "COC-2024-001",
    "projectName": "City Water Department",
    "reportToName1": "John Doe",
    "reportToEmail1": "john.doe@citywater.ca",
    "reportToName2": "",
    "reportToEmail2": "",
    "dateRequired": "2024-01-22 17:00:00",
    "isRush": 0,
    "dateForRush": null,
    "receivedBy": "Sarah Johnson",
    "receivedTime": "2024-01-15 09:30:00",
    "relinquishedBy": "Mike Chen",
    "relinquishedTime": "2024-01-14 16:00:00",
    "numberOfContainers": 5,
    "specialInstructions": "Handle with care. Keep below 4°C.",
    "createdByUserId": "u1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c",
    "createdAt": "2024-01-14 16:05:00",
    "status": "Completed",
    "samples": [
      {
        "sampleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "sampleClientId": "S_DW_01",
        "matrix": "Drinking Water",
        "samplingPoint": "Inlet",
        "sampledTime": "2024-01-14 10:30:00",
        "initialVolume": 1000.00,
        "remainingVolume": 650.00,
        "status": "Completed",
        "totalTests": 3
      },
      {
        "sampleId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
        "sampleClientId": "S_DW_02",
        "matrix": "Drinking Water",
        "samplingPoint": "Inlet",
        "sampledTime": "2024-01-14 11:50:00",
        "initialVolume": 1000.00,
        "remainingVolume": 1000.00,
        "status": "In-Progress",
        "totalTests": 0
      }
    ]
  }
}
```

