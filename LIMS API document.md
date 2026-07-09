# Sprint 1

## 📌 Global Conventions

- **Architectural Note: This system only supports **GET** and **POST** methods; **PUT** and **DELETE\*\* are intentionally excluded.
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
- **Architect Note (For Backend & Frontend):** \* **Frontend:** To update an existing parameter, you **MUST** include its original `parameterId`. To add a brand-new parameter, omit the `parameterId` (or pass `null`). If you remove a parameter from this array, the backend will treat it as a deletion request.
  - **Backend:** Use the "Diffing Algorithm" inside a single JDBC transaction:
    1. `UPDATE` the main `Test_Type` row using the `{id}` from the URL.
    2. Loop through the incoming `parameters` JSON array.
    3. If `parameterId` exists **$\rightarrow$** `UPDATE` existing parameter.
    4. If `parameterId` is null/missing **$\rightarrow$** `INSERT` new parameter.
    5. If an existing DB parameter is missing from the JSON **$\rightarrow$** `DELETE` it (catch FK exceptions gracefully).
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
