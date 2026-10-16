# HIPAA Governance Lab in Databricks

This hands-on lab uses synthetic patient data to demonstrate basic HIPAA governance concepts in Databricks.

> **Important:** Use fake data only. This lab demonstrates technical controls; completing it does not make an organization HIPAA compliant.

## What is HIPAA?

HIPAA stands for the **Health Insurance Portability and Accountability Act of 1996**. It is a United States federal law that establishes requirements for protecting certain health information and regulating how that information may be used and disclosed.

HIPAA's major components include the Privacy Rule, Security Rule, and Breach Notification Rule. The Security Rule focuses on protecting electronic protected health information (ePHI) through administrative, physical, and technical safeguards.

### What is PHI?

**Protected Health Information (PHI)** is individually identifiable health information that is created, received, maintained, or transmitted by a covered entity or business associate. Examples include a patient's name, phone number, email address, date of birth, medical record number, diagnosis, and treatment information when they can be connected to a specific person.

## Lab objectives

This lab demonstrates:

1. Creating a synthetic PHI table
2. Identifying sensitive columns
3. Masking personal identifiers
4. Restricting access through a secure view
5. Applying a row-level restriction
6. Reviewing audit logs when available

Databricks Unity Catalog supports row filters and column masks for query-time protection:

<https://docs.databricks.com/aws/en/data-governance/unity-catalog/filters-and-masks/>

## Prerequisites

- A Databricks workspace with a SQL warehouse or supported serverless compute
- Unity Catalog enabled for catalog-level governance features
- Permission to create tables and views
- Synthetic data only

If you are using Databricks Free Edition, use the catalog and schema shown by these commands:

```sql
SHOW CATALOGS;
SHOW SCHEMAS IN workspace;
```

Some Free Edition accounts may not allow catalog creation, groups, grants, or access to system audit tables. In that case, use an existing catalog and the masked-view portions of this lab.

## 1. Create synthetic patient data

Run this in a Python notebook cell:

```python
from datetime import date

data = [
    ("P001", "Maria Garcia", "555-123-4567", "maria@example.com",
     date(1985, 4, 12), "Diabetes", "Los Angeles"),
    ("P002", "James Smith", "555-987-6543", "james@example.com",
     date(1978, 11, 30), "Hypertension", "Seattle"),
    ("P003", "Aisha Khan", "555-222-3333", "aisha@example.com",
     date(1992, 7, 19), "Asthma", "Chicago")
]

schema = """
patient_id STRING,
patient_name STRING,
phone STRING,
email STRING,
date_of_birth DATE,
diagnosis STRING,
city STRING
"""

df = spark.createDataFrame(data, schema=schema)
display(df)
```

## 2. Create a governed table

On a full Unity Catalog workspace, run:

```sql
CREATE CATALOG IF NOT EXISTS hipaa_demo;
CREATE SCHEMA IF NOT EXISTS hipaa_demo.clinical;

USE CATALOG hipaa_demo;
USE SCHEMA clinical;
```

If catalog creation is not available, replace `hipaa_demo.clinical` below with an existing catalog and schema, such as `workspace.hipaa_demo`.

Save the data as a Delta table:

```python
df.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("hipaa_demo.clinical.patient_phi")
```

Check the raw table:

```sql
SELECT *
FROM hipaa_demo.clinical.patient_phi;
```

## 3. Identify PHI columns

| Column | Classification |
|---|---|
| `patient_id` | Patient identifier |
| `patient_name` | Direct identifier |
| `phone` | Direct identifier |
| `email` | Direct identifier |
| `date_of_birth` | Quasi-identifier |
| `diagnosis` | Health information |
| `city` | Location information |

This table contains PHI because it connects identifiable patients with health information.

## 4. Create a masked view for analysts

This view hides most direct identifiers while keeping the data useful for analysis:

```sql
CREATE OR REPLACE VIEW hipaa_demo.clinical.patient_analyst_view AS
SELECT
    patient_id,
    concat(substring(patient_name, 1, 1), '***') AS patient_name_masked,
    concat('***-***-', right(phone, 4)) AS phone_masked,
    concat('***@', split(email, '@')[1]) AS email_masked,
    year(date_of_birth) AS birth_year,
    diagnosis,
    city
FROM hipaa_demo.clinical.patient_phi;
```

Test the protected view:

```sql
SELECT *
FROM hipaa_demo.clinical.patient_analyst_view;
```

Expected results:

```text
P001 | M*** | ***-***-4567 | ***@example.com | 1985 | Diabetes     | Los Angeles
P002 | J*** | ***-***-6543 | ***@example.com | 1978 | Hypertension | Seattle
P003 | A*** | ***-***-3333 | ***@example.com | 1992 | Asthma       | Chicago
```

## 5. Grant access to the masked view

These commands require the appropriate Unity Catalog permissions and may not work in Free Edition:

```sql
CREATE GROUP IF NOT EXISTS hipaa_analysts;
CREATE GROUP IF NOT EXISTS hipaa_privileged_users;
```

Grant analysts access to the masked view:

```sql
GRANT USE CATALOG
ON CATALOG hipaa_demo
TO `hipaa_analysts`;

GRANT USE SCHEMA
ON SCHEMA hipaa_demo.clinical
TO `hipaa_analysts`;

GRANT SELECT
ON VIEW hipaa_demo.clinical.patient_analyst_view
TO `hipaa_analysts`;
```

Do not grant analysts direct access to the raw PHI table:

```sql
REVOKE SELECT
ON TABLE hipaa_demo.clinical.patient_phi
FROM `hipaa_analysts`;
```

Privileged users may receive access to the original table:

```sql
GRANT SELECT
ON TABLE hipaa_demo.clinical.patient_phi
TO `hipaa_privileged_users`;
```

## 6. Add a row-level restriction

Suppose analysts should see only patients from Los Angeles:

```sql
CREATE OR REPLACE VIEW hipaa_demo.clinical.la_patient_analyst_view AS
SELECT *
FROM hipaa_demo.clinical.patient_analyst_view
WHERE city = 'Los Angeles';
```

Test the restricted view:

```sql
SELECT *
FROM hipaa_demo.clinical.la_patient_analyst_view;
```

This is a simple example of row-level security. Production implementations can use Unity Catalog row filters and column masks:

<https://docs.databricks.com/aws/en/data-governance/unity-catalog/filters-and-masks/>

## 7. Query audit logs

If system tables are enabled and you have permission, query Databricks audit activity:

```sql
SELECT
    event_time,
    user_identity.email AS user_email,
    action_name,
    request_params,
    source_ip_address
FROM system.access.audit
WHERE event_date >= current_date() - INTERVAL 7 DAYS
ORDER BY event_time DESC;
```

Databricks stores account audit activity in `system.access.audit`. Access may require an account administrator or metastore administrator to grant permission:

<https://docs.databricks.com/aws/en/admin/system-tables/audit-logs>

## Practice tasks

1. Add a `medical_record_number` column and mask it.
2. Replace exact dates of birth with only the birth year.
3. Create a view that hides `diagnosis`.
4. Create a view containing only `patient_id`, `diagnosis`, and `city`.
5. Add a rule allowing analysts to see only one city.
6. Compare the raw table with the masked view.
7. Add a data dictionary describing each column's classification and permitted users.

## HIPAA governance reminder

Technical masking and access controls are only part of HIPAA governance. A real compliance program also requires policies, workforce training, identity and access management, encryption, risk assessments, incident response, breach procedures, documentation, and applicable Business Associate Agreements.

Do not upload real patient information to this learning exercise.
