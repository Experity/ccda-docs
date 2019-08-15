# ccda-docs

## Table of Contents

- [Domains](#domains)
- Routes
  - [Health](#health)
  - [CCD](#ccd)
    - [CCD Authorization](#ccd-authorization)
    - Ccd Routes
      - [GET](#get-ccd)
      - [POST](#post-ccd)
      - [OPTIONS](#options-ccd)
      - Sections and Template-Names
        - [CCD Sections](#ccd-sections)
        - [CCD Template Name](#ccd-template-name)

---

### Domains

| Environment | Domain                            |
| ----------- | --------------------------------- |
| Staging     | `https://records.docutaptest.com` |
| Production  | `https://records.docutap.com`     |

---

## Routes

### Health

- `/health` - Returns 200 when the container is healthy.

- **Query Parameters**:

  - **:site-id** - requied; String; The database to retrieve data from (Ex: **BS001**).
  - **:patient-id** - required; Integer; A patient's ID **(PID)**.
  - **:visit-id** - required; Integer; A patient's visit ID **(VID)**.
  - **mock** - optional; String ("true" is the only valid input); If true, returns a mocked payload. Useful for debugging purposes.

---

### CCD

#### CCD Authorization

- All routes use a **Bearer** token for authorization (auth).

---

### GET CCD

- **Resource (DEPRECATED -- use POST resource)**: `GET /api/:version/sites/:site-id/patients/:patient-id/ccd?visit=:visit-id&user=:username[&sections=<see_below>][&template-name=<see_below>][&mock=true]`

- **URL Parameters (Required)**:

  - **:version** - String; `{v1 or v2}` Signifies which service to generate a CCD.
  - **:site-id** - String; The database to retrieve data from (Ex: **BS001**).
  - **:patient-id** - String; A patient's ID (**PID**).

- **Query Parameters**:

  - **visit** - required; Integer; A patient's visit ID (**VID**).
  - **user** - required; String; A username for logging purposes.
  - **sections** - optional; String (comma delimited); See [this](#ccd-sections) for notes on valid sections values.
  - **template-name** - optional; See [this](#ccd-template-name) for notes on valid template-name values.
  - **mock** - optional; String ("true" is the only valid input); If true, returns a mocked payload. Useful for debugging purposes.

- **Response**:

  - Status code: `200`
  - Data: `XML`

- **Notes**:

1. The version parameter **v2** will work better than **v1**.
1. All parameter key/values should be separated by dashes, if necessary. Ex: `&user=api-user`.
1. An example of a sections string: `sections=goals,vitals,social-history`
1. Example Route `https://records.docutaptest.com/api/v2/sites/DT061/patients/121378/ccd?visit=82593&user=api-user`

---

### POST CCD

- **Resource**: `POST /api/:version/sites/:site-id/patients/:patient-id/ccd`

- **POST CCD URL Parameters (Required)**

  - **:version** - String; `{v1 or v2}` Signifies which service to generate a CCD.
  - **:site-id** - String; The database to retrieve data from (Ex: **BS001**).
  - **:patient-id** - String; A patient's ID (**PID**).

- **POST CCD Body Parameters (in JSON)**

  - **visit** - required; Integer; A patient's visit ID (**VID**).
  - **user** - required; String; A username for logging purposes.
  - **sections** - optional; Array[Strings]; See [this](#ccd-sections) for notes on valid **sections** values.
  - **template-name** - optional; See [this](#ccd-template-name) for notes on valid **template-name** values.
  - **security**¹ - optional; Object.
    - **code** - optional; String (single character); `N`, `R`, or `V` (for **Normal**, **Restricted**, and **Very Restricted**). Defaults to an `N` if no `code` field is supplied.
    - **text** - optional; String. Defaults to `""` if none supplied.
  - **mock** - optional; String ("true" is only valid input); If true, returns a mocked payload. Useful for debugging purposes.

- **JSON body example**:

  ```json
  {
    "visit": 147777,
    "user": "api-user",
    "sections": ["goals", "med-list", "social-history", "vitals"],
    "template-name": "clinical-summary",
    "security": {
      "code": "N",
      "text": "Some example text placeholder"
    },
    "mock": "true"
  }
  ```

- **Response**:

  - Status code: `200`
  - Data: `XML`

- **Notes**:

1. The security section will always be attempted to be generated _at the very end of the CCD document_. This section is special since it isn't explicitly listed in the `sections` string array. Instead, if a `security object` is provided (and the `text` field has data), then the security section will be generated.
1. The version parameter **v2** will work better than **v1**.
1. All parameter key/values should be _separated by dashes_, if necessary.
   - Ex: `user=api-user`

- **Example Route**: `https://records.docutaptest.com/api/v2/sites/DT061/patients/121378/ccd`

---

### OPTIONS CCD

- **Resource**: `OPTIONS /v2/sites/:site/patients/:patient/ccd`
- **Alias**: `OPTIONS /v2/ccd`

- **URL Parameters (Required)**:

  - **:site-id** - String; The database to retrieve data from (Ex: **BS001**).
  - **:patient-id** - String; A patient's ID (**PID**).

- **Response**:

  - Status Code: `200`
  - Data: `JSON`
  - Example (Use the `OPTIONS` route for the most up-to-date example):

    ```json
    {
      "sections": [
        "cognitive-status",
        "encounter-diagnosis",
        ...
      ],
      "template-name": ["care-plan", "clinical-summary", "transition-of-care/referral"]
    }
    ```

- **Notes**:

1. The **alias** will _mimic_ the behavior of the **resource**, except that you no longer have to specify a `site-id` and a `pid`.

---

### CCD Sections

To get the most up-to-date `sections` listing, use the [OPTIONS CCD Route](#options-ccd).

- **Notes**:

1. The parameter values are **case insensitive**.
1. If no `sections` parameter is passed to the route, **ALL** sections will be generated, if possible.
1. If you specify certain sections, **be warned** that you may generate valid XML code _but it might not be a valid CCD document_!
1. Regarding the `interventions` section, if no `medications` or `procedures` arrays are included in the JSON payload, the `text` field will be placed into the `instruction` subsection.
1. The `Plan of Treatment` section in the CCD doesn't have the name `plan-of-treatment` mapped to it when passing a list of `sections` via the body parameter. Instead, if any combination of the following section names are sent in the `sections` body parameter, the `Plan of Treatment` section will be generated _exactly once_. Otherwise, the `Plan of Treatment` section won't be generated.

   1. `future-appointments`
   1. `future-scheduled-tests`
   1. `pending-diagnostic-tests`
   1. `patient-education`

1. The `lab-tests-and-results` (`lab-results`) section will only generate if **atleast one of the following JSON fields are provided** to the API:

   1. `pending_tests`
   1. `future_lab`
   1. `future_xray`
   1. `misc_procedures`

1. See [MR-2015](https://docutap.atlassian.net/browse/MR-2015) for notes/discussions.

---

### CCD Template Name

To get the most up-to-date `template-name` listing, use the [OPTIONS CCD Route](#options-ccd).

- **Notes**:

1. The parameter values are **case insensitive**.
1. If no `template-name` is provided, the default will be `clinical-summary`.
1. The route only accepts one template name. If multiple are given, the default template will be returned.
1. See [MR-2015](https://docutap.atlassian.net/browse/MR-2015) for notes/discussions.

---

### CCD Search Route

- **Resource**: `POST /api/:version/sites/:site-id/patients/:patient-id/ccd/search`

Inputs are the same as the [CCD Post route](#post-ccd) except for two new body params:

- **Additional Body Parameters**:

  - **start-date** - optional; string; MM/DD/YYYY format (Ex: 01/23/2019).
  - **end-date** - optional; string; MM/DD/YYYY format (Ex: 01/23/2019).

- **Response**:

  - Status code: `200`
  - Data: `JSON`
  - Example:

    ```jsonc
    {
      "results": [
        {
          "date": "05/31/2019",
          "visitId": "1234",
          "status": "Completed",
          "ccd": "<?xml version=\"1.0\" ?>\n<ClinicalDocument…" // raw CCD data as a string
        }
      ]
    }
    ```

- **Notes**:

1. If `start-date` or `end-date` are used, **you must provide both** `start-date` and `end-date`
