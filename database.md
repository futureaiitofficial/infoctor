## **Table of Contents**

1. [Introduction](#1-introduction)
2. [Assumptions and Considerations](#2-assumptions-and-considerations)
3. [Database Schema Creation Scripts](#3-database-schema-creation-scripts)
   - 3.1 [Tenant Management](#31-tenant-management)
   - 3.2 [User Management and RBAC](#32-user-management-and-rbac)
   - 3.3 [Reference Tables for Code Sets](#33-reference-tables-for-code-sets)
   - 3.4 [Patient Management](#34-patient-management)
   - 3.5 [Appointment Scheduling](#35-appointment-scheduling)
   - 3.6 [Clinical Documentation (FHIR Resources)](#36-clinical-documentation-fhir-resources)
   - 3.7 [Medication and ePrescribing](#37-medication-and-eprescribing)
   - 3.8 [Lab Orders and Results](#38-lab-orders-and-results)
   - 3.9 [Consent Management](#39-consent-management)
   - 3.10 [Billing and Revenue Cycle Management](#310-billing-and-revenue-cycle-management)
   - 3.11 [Audit Logging and Provenance](#311-audit-logging-and-provenance)
4. [Notes on Loading Code Sets](#4-notes-on-loading-code-sets)
5. [Conclusion](#5-conclusion)

---

## **1. Introduction**

The following SQL scripts will create the necessary tables, relationships, and constraints to fulfill all the requirements for the Infoctor EHR system. This includes multi-tenancy support, user management with role-based access control (RBAC), integration of ICD-10 and other clinical coding systems, alignment with FHIR resources, and compliance with healthcare regulations.

---

## **2. Assumptions and Considerations**

- **Database Management System:** PostgreSQL 13 or higher.
- **UUIDs:** Used as primary keys for scalability and uniqueness across tenants.
- **JSONB Fields:** Used for flexible data storage where appropriate.
- **Compliance:** Designed to meet HIPAA, GDPR, and other regulatory requirements.
- **ICD-10 and Other Codes:** Reference tables will be created, but actual code data must be loaded from official sources.
- **FHIR Compliance:** Database structures align with FHIR resource definitions.

---

## **3. Database Schema Creation Scripts**

### **3.1 Tenant Management**

```sql
-- Create the tenants table
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    subscription_plan VARCHAR(50),
    status VARCHAR(50) CHECK (status IN ('active', 'suspended', 'terminated')) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.2 User Management and RBAC**

```sql
-- Create the roles table
CREATE TABLE roles (
    role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create the permissions table
CREATE TABLE permissions (
    permission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create the role_permissions table (Many-to-Many)
CREATE TABLE role_permissions (
    role_id UUID REFERENCES roles(role_id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(permission_id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Create the users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    username VARCHAR(150) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    phone_number VARCHAR(20),
    role_id UUID REFERENCES roles(role_id) ON DELETE SET NULL,
    status VARCHAR(50) CHECK (status IN ('active', 'inactive')) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, username)
);
```

---

### **3.3 Reference Tables for Code Sets**

#### **ICD-10 Codes**

```sql
-- Create the icd10_codes table
CREATE TABLE icd10_codes (
    code VARCHAR(10) PRIMARY KEY,
    description TEXT NOT NULL,
    chapter VARCHAR(255),
    block VARCHAR(255),
    is_billable BOOLEAN DEFAULT TRUE,
    effective_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **CPT Codes**

```sql
-- Create the cpt_codes table
CREATE TABLE cpt_codes (
    code VARCHAR(10) PRIMARY KEY,
    description TEXT NOT NULL,
    category VARCHAR(255),
    effective_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **SNOMED CT Concepts**

```sql
-- Create the snomed_ct_concepts table
CREATE TABLE snomed_ct_concepts (
    concept_id BIGINT PRIMARY KEY,
    fully_specified_name TEXT NOT NULL,
    preferred_term TEXT,
    active BOOLEAN,
    effective_time DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **LOINC Codes**

```sql
-- Create the loinc_codes table
CREATE TABLE loinc_codes (
    loinc_num VARCHAR(10) PRIMARY KEY,
    component TEXT,
    property TEXT,
    time_aspct TEXT,
    system TEXT,
    scale_typ TEXT,
    method_typ TEXT,
    class VARCHAR(100),
    source VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **RxNorm Codes**

```sql
-- Create the rxnorm_codes table
CREATE TABLE rxnorm_codes (
    rxcui VARCHAR(15) PRIMARY KEY,
    name TEXT NOT NULL,
    tty VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.4 Patient Management**

```sql
-- Create the patients table
CREATE TABLE patients (
    patient_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    identifiers JSONB, -- e.g., [{"system": "Aadhar", "value": "123456789012"}]
    active BOOLEAN DEFAULT TRUE,
    name JSONB NOT NULL, -- FHIR HumanName structure
    telecom JSONB, -- Contact points (phone, email)
    gender VARCHAR(20) CHECK (gender IN ('male', 'female', 'other', 'unknown')),
    birth_date DATE,
    address JSONB, -- FHIR Address structure
    marital_status VARCHAR(50),
    photo BYTEA,
    communication JSONB, -- Languages and preferences
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index for faster search by patient name
CREATE INDEX idx_patients_name ON patients USING gin ((name) gin_trgm_ops);
```

#### **Patient Allergies (AllergyIntolerance Resource)**

```sql
-- Create the allergy_intolerances table
CREATE TABLE allergy_intolerances (
    allergy_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    clinical_status VARCHAR(50),
    verification_status VARCHAR(50),
    type VARCHAR(50),
    category VARCHAR(50),
    criticality VARCHAR(50),
    code JSONB, -- CodeableConcept (e.g., SNOMED CT code)
    reaction JSONB, -- List of reactions
    recorder_id UUID REFERENCES users(user_id),
    recorded_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_occurrence TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **Patient Problems (Condition Resource)**

```sql
-- Create the conditions table
CREATE TABLE conditions (
    condition_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    clinical_status VARCHAR(50),
    verification_status VARCHAR(50),
    category JSONB, -- List of categories
    severity JSONB, -- CodeableConcept
    code JSONB, -- CodeableConcept (e.g., ICD-10 code)
    body_site JSONB, -- List of body sites
    onset_date DATE,
    abatement_date DATE,
    recorded_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    recorder_id UUID REFERENCES users(user_id),
    asserter_id UUID REFERENCES users(user_id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.5 Appointment Scheduling (Appointment Resource)**

```sql
-- Create the appointments table
CREATE TABLE appointments (
    appointment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('proposed', 'pending', 'booked', 'arrived', 'fulfilled', 'cancelled', 'noshow', 'entered-in-error', 'checked-in', 'waitlist')) NOT NULL,
    service_category JSONB, -- CodeableConcept
    service_type JSONB, -- CodeableConcept
    specialty JSONB, -- CodeableConcept
    appointment_type JSONB, -- CodeableConcept
    start TIMESTAMP WITH TIME ZONE NOT NULL,
    end TIMESTAMP WITH TIME ZONE NOT NULL,
    participant JSONB NOT NULL, -- List of participants (patient, practitioner)
    reason_code JSONB, -- List of CodeableConcepts
    reason_reference JSONB, -- References to other resources
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index for quick lookup of appointments
CREATE INDEX idx_appointments_start_end ON appointments (start, end);
```

---

### **3.6 Clinical Documentation (FHIR Resources)**

#### **Observations**

```sql
-- Create the observations table
CREATE TABLE observations (
    observation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('registered', 'preliminary', 'final', 'amended', 'corrected', 'cancelled', 'entered-in-error', 'unknown')) NOT NULL,
    category JSONB, -- List of CodeableConcepts
    code JSONB NOT NULL, -- CodeableConcept (e.g., LOINC code)
    subject_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    effective_datetime TIMESTAMP WITH TIME ZONE,
    issued TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    performer JSONB, -- List of practitioners
    value_quantity JSONB, -- Quantity data type
    value_codeable_concept JSONB,
    interpretation JSONB, -- CodeableConcept
    note JSONB, -- List of annotations
    body_site JSONB, -- CodeableConcept
    method JSONB, -- CodeableConcept
    device JSONB, -- Reference to a device
    reference_range JSONB, -- List of reference ranges
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **Clinical Notes (Composition Resource)**

```sql
-- Create the compositions table
CREATE TABLE compositions (
    composition_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('preliminary', 'final', 'amended', 'entered-in-error')) NOT NULL,
    type JSONB NOT NULL, -- CodeableConcept
    category JSONB, -- List of CodeableConcepts
    subject_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    author JSONB NOT NULL, -- List of practitioners
    title VARCHAR(255),
    confidentiality VARCHAR(50),
    attester JSONB, -- List of attesters
    custodian JSONB, -- Organization
    sections JSONB, -- Structured content
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.7 Medication and ePrescribing (MedicationRequest and Medication Resources)**

#### **Medications**

```sql
-- Create the medications table
CREATE TABLE medications (
    medication_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    code JSONB NOT NULL, -- CodeableConcept (e.g., RxNorm code)
    status VARCHAR(50) CHECK (status IN ('active', 'inactive', 'entered-in-error')) DEFAULT 'active',
    manufacturer JSONB, -- Reference to organization
    form JSONB, -- CodeableConcept
    amount JSONB, -- Ratio
    ingredient JSONB, -- List of ingredients
    batch JSONB, -- Batch details
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **Medication Requests (Prescriptions)**

```sql
-- Create the medication_requests table
CREATE TABLE medication_requests (
    medication_request_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('active', 'on-hold', 'cancelled', 'completed', 'entered-in-error', 'stopped', 'draft', 'unknown')) NOT NULL,
    intent VARCHAR(50) CHECK (intent IN ('proposal', 'plan', 'order', 'original-order', 'reflex-order', 'filler-order', 'instance-order', 'option')) NOT NULL,
    medication_id UUID REFERENCES medications(medication_id),
    subject_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    authored_on TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    requester_id UUID REFERENCES users(user_id),
    dosage_instruction JSONB, -- List of dosage instructions
    dispense_request JSONB, -- Details about dispensing
    substitution JSONB, -- Details about substitution
    prior_prescription_id UUID, -- Reference to previous prescription
    detected_issue JSONB, -- List of detected issues
    event_history JSONB, -- List of events
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.8 Lab Orders and Results (ServiceRequest and DiagnosticReport Resources)**

#### **Service Requests (Lab Orders)**

```sql
-- Create the service_requests table
CREATE TABLE service_requests (
    service_request_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('draft', 'active', 'on-hold', 'revoked', 'completed', 'entered-in-error', 'unknown')) NOT NULL,
    intent VARCHAR(50) CHECK (intent IN ('proposal', 'plan', 'order', 'original-order', 'reflex-order', 'filler-order', 'instance-order', 'option')) NOT NULL,
    category JSONB, -- List of CodeableConcepts
    code JSONB NOT NULL, -- CodeableConcept (e.g., LOINC code)
    subject_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    authored_on TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    requester_id UUID REFERENCES users(user_id),
    performer JSONB, -- List of practitioners or organizations
    reason_code JSONB, -- List of CodeableConcepts
    supporting_info JSONB, -- References to other resources
    specimen JSONB, -- List of specimens
    note JSONB, -- List of annotations
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **Diagnostic Reports (Lab Results)**

```sql
-- Create the diagnostic_reports table
CREATE TABLE diagnostic_reports (
    diagnostic_report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('registered', 'partial', 'preliminary', 'final', 'amended', 'corrected', 'appended', 'cancelled', 'entered-in-error', 'unknown')) NOT NULL,
    category JSONB, -- List of CodeableConcepts
    code JSONB NOT NULL, -- CodeableConcept
    subject_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    effective_datetime TIMESTAMP WITH TIME ZONE,
    issued TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    performer JSONB, -- List of practitioners or organizations
    results_interpreter JSONB, -- List of practitioners
    specimen JSONB, -- List of specimens
    result JSONB, -- References to observations
    conclusion TEXT,
    conclusion_code JSONB, -- List of CodeableConcepts
    presented_form JSONB, -- Attachments
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.9 Consent Management (Consent Resource)**

```sql
-- Create the consents table
CREATE TABLE consents (
    consent_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('draft', 'proposed', 'active', 'rejected', 'inactive', 'entered-in-error')) NOT NULL,
    scope VARCHAR(50), -- CodeableConcept
    category JSONB, -- List of CodeableConcepts
    patient_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    date_time TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    performer JSONB, -- List of practitioners
    organization JSONB, -- List of organizations
    policy JSONB, -- List of policies
    provision JSONB, -- Rules for data access
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.10 Billing and Revenue Cycle Management**

#### **Billing Records**

```sql
-- Create the billing_records table
CREATE TABLE billing_records (
    billing_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    patient_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    service_date DATE NOT NULL,
    total_charges NUMERIC(12,2) DEFAULT 0.00,
    total_payments NUMERIC(12,2) DEFAULT 0.00,
    total_adjustments NUMERIC(12,2) DEFAULT 0.00,
    balance NUMERIC(12,2) GENERATED ALWAYS AS (total_charges - total_payments - total_adjustments) STORED,
    status VARCHAR(50) CHECK (status IN ('pending', 'paid', 'overdue', 'written-off')) DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create the billing_items table
CREATE TABLE billing_items (
    billing_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_id UUID REFERENCES billing_records(billing_id) ON DELETE CASCADE,
    code VARCHAR(10), -- Reference to code tables
    code_type VARCHAR(10) CHECK (code_type IN ('ICD-10', 'CPT', 'HCPCS')),
    description TEXT,
    charge_amount NUMERIC(12,2) DEFAULT 0.00,
    quantity INTEGER DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **3.11 Audit Logging and Provenance**

#### **Audit Logs (AuditEvent Resource)**

```sql
-- Create the audit_events table
CREATE TABLE audit_events (
    audit_event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    type VARCHAR(50), -- CodeableConcept
    subtype JSONB, -- List of CodeableConcepts
    action VARCHAR(10) CHECK (action IN ('C', 'R', 'U', 'D', 'E')), -- Create, Read, Update, Delete, Execute
    recorded TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    outcome VARCHAR(50), -- Success, minor failure, serious failure, major failure
    outcome_desc TEXT,
    agent JSONB NOT NULL, -- User details
    source JSONB NOT NULL, -- System details
    entity JSONB, -- Details of the entity involved
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **Provenance**

```sql
-- Create the provenance table
CREATE TABLE provenance (
    provenance_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target JSONB NOT NULL, -- References to resources
    recorded TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    agent JSONB NOT NULL, -- User details
    activity JSONB, -- CodeableConcept
    signature JSONB, -- Digital signature
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## **4. Notes on Loading Code Sets**

- **ICD-10 Codes:** The `icd10_codes` table should be populated with the official ICD-10 code set, which can be obtained from authorized sources like the World Health Organization (WHO) or national health agencies.
- **CPT Codes:** Obtain the CPT code set from the American Medical Association (AMA) or authorized distributors.
- **SNOMED CT Concepts:** SNOMED CT is distributed by SNOMED International and may require a license.
- **LOINC Codes:** LOINC codes are available from the Regenstrief Institute.
- **RxNorm Codes:** RxNorm data can be downloaded from the U.S. National Library of Medicine.

**Loading Data Example:**

```sql
-- Example of inserting an ICD-10 code
INSERT INTO icd10_codes (code, description, chapter, block, is_billable, effective_date)
VALUES ('A00', 'Cholera', 'I. Certain infectious and parasitic diseases', 'A00-A09', TRUE, '2021-10-01');
```

*Note: Bulk loading methods (e.g., COPY command, ETL tools) should be used for large datasets.*

---

## **5. Conclusion**

The provided SQL scripts create a comprehensive database schema for the Infoctor EHR system, addressing all the requirements, including:

- **Multi-Tenancy Support:** Ensuring data isolation between tenants.
- **User Management and RBAC:** Secure access control mechanisms.
- **Integration of Code Sets:** Tables for ICD-10, CPT, SNOMED CT, LOINC, and RxNorm codes.
- **FHIR Compliance:** Aligning database structures with FHIR resource definitions for interoperability.
- **Clinical Documentation:** Support for detailed clinical data capture.
- **Consent Management:** Recording patient consents in compliance with regulations.
- **Audit Logging and Provenance:** Maintaining detailed logs for security and compliance.
- **Compliance and Security Measures:** Designed to meet HIPAA, GDPR, and other regulatory standards.

---

**Next Steps:**

1. **Implement the Database:**
   - Execute the SQL scripts in a PostgreSQL environment.
2. **Load Code Sets:**
   - Populate the reference tables with official code data.
3. **Develop API Layer:**
   - Build FHIR-compliant APIs to interface with the database.
4. **Implement Business Logic:**
   - Develop application layers to handle workflows and data processing.
5. **Testing:**
   - Perform thorough testing, including unit, integration, and security testing.
6. **Deployment:**
   - Deploy the system in a secure, compliant infrastructure.

**Please note** that this schema serves as a foundation. Additional indexes, constraints, or optimizations may be necessary based on specific use cases and performance considerations.

---
