---

# **Complete Updated Database Schema for Infoctor EHR System**

## **Table of Contents**

1. [Introduction](#1-introduction)
2. [Assumptions and Considerations](#2-assumptions-and-considerations)
3. [Database Schema Overview](#3-database-schema-overview)
4. [Database Creation Scripts](#4-database-creation-scripts)
   - 4.1 [Tenant Management](#41-tenant-management)
   - 4.2 [User Management and RBAC](#42-user-management-and-rbac)
   - 4.3 [Reference Tables for Coding Systems](#43-reference-tables-for-coding-systems)
     - 4.3.1 [ICD-10 Codes](#431-icd-10-codes)
     - 4.3.2 [CPT Codes](#432-cpt-codes)
     - 4.3.3 [SNOMED CT Concepts](#433-snomed-ct-concepts)
     - 4.3.4 [LOINC Codes](#434-loinc-codes)
     - 4.3.5 [RxNorm Codes](#435-rxnorm-codes)
   - 4.4 [Patient Management](#44-patient-management)
   - 4.5 [Appointment Scheduling](#45-appointment-scheduling)
   - 4.6 [Clinical Documentation](#46-clinical-documentation)
     - 4.6.1 [Observations](#461-observations)
     - 4.6.2 [Conditions](#462-conditions)
     - 4.6.3 [Procedures](#463-procedures)
     - 4.6.4 [Clinical Notes (Compositions)](#464-clinical-notes-compositions)
   - 4.7 [Medication and ePrescribing](#47-medication-and-eprescribing)
     - 4.7.1 [Medications](#471-medications)
     - 4.7.2 [Medication Requests](#472-medication-requests)
   - 4.8 [Lab Orders and Results](#48-lab-orders-and-results)
     - 4.8.1 [Service Requests (Lab Orders)](#481-service-requests-lab-orders)
     - 4.8.2 [Diagnostic Reports (Lab Results)](#482-diagnostic-reports-lab-results)
   - 4.9 [Consent Management](#49-consent-management)
   - 4.10 [Billing and Revenue Cycle Management](#410-billing-and-revenue-cycle-management)
   - 4.11 [Audit Logging and Provenance](#411-audit-logging-and-provenance)
   - 4.12 [FAX Integration](#412-fax-integration)
   - 4.13 [Secure Messaging](#413-secure-messaging)
5. [Compliance and Security Measures](#5-compliance-and-security-measures)
6. [Data Loading Notes](#6-data-loading-notes)
7. [Conclusion](#7-conclusion)
8. [Next Steps](#8-next-steps)

---

## **1. Introduction**

This updated database schema is designed to comprehensively support the Infoctor EHR system, incorporating all required features and ensuring compliance with healthcare regulations and standards. The schema includes:

- **Multi-Tenancy Support**: Data isolation between tenants.
- **User Management with RBAC**: Secure access control.
- **Integration of Coding Systems**: ICD-10, CPT, SNOMED CT, LOINC, RxNorm.
- **FAX Integration**: Sending and receiving faxes within the EHR.
- **Secure Messaging**: HIPAA-compliant communication between users.
- **FHIR Compliance**: Alignment with FHIR resources for interoperability.
- **Audit Logging and Provenance**: Detailed logs for security and compliance.

---

## **2. Assumptions and Considerations**

- **Database Management System**: PostgreSQL 13 or higher.
- **UUIDs**: Used as primary keys for uniqueness.
- **JSONB Fields**: Employed for flexible data structures.
- **Compliance**: Designed to meet HIPAA, GDPR, and other regulations.
- **Coding Systems**: Reference tables are created; actual code data must be loaded from official sources.
- **FAX and Secure Messaging**: Integrated with compliance and security in mind.

---

## **3. Database Schema Overview**

The database is organized into several key areas:

1. **Tenant and User Management**: Handling multi-tenancy and user roles.
2. **Reference Data**: Storing standard coding systems.
3. **Clinical Data**: Managing patient records, encounters, observations, etc.
4. **Administrative Data**: Appointments, billing, consents.
5. **Communication**: FAX and secure messaging functionalities.
6. **Audit and Compliance**: Logging and provenance tracking.

---

## **4. Database Creation Scripts**

### **4.1 Tenant Management**

```sql
-- Enable UUID generation extension
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Tenants table
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

### **4.2 User Management and RBAC**

```sql
-- Roles table
CREATE TABLE roles (
    role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Permissions table
CREATE TABLE permissions (
    permission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Role-Permissions mapping
CREATE TABLE role_permissions (
    role_id UUID REFERENCES roles(role_id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(permission_id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Users table
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

### **4.3 Reference Tables for Coding Systems**

#### **4.3.1 ICD-10 Codes**

```sql
-- ICD-10 Codes table
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

#### **4.3.2 CPT Codes**

```sql
-- CPT Codes table
CREATE TABLE cpt_codes (
    code VARCHAR(10) PRIMARY KEY,
    description TEXT NOT NULL,
    category VARCHAR(255),
    effective_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **4.3.3 SNOMED CT Concepts**

```sql
-- SNOMED CT Concepts table
CREATE TABLE snomed_concepts (
    concept_id BIGINT PRIMARY KEY,
    fully_specified_name TEXT NOT NULL,
    preferred_term TEXT,
    active BOOLEAN,
    effective_time DATE,
    module_id BIGINT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- SNOMED CT Relationships table
CREATE TABLE snomed_relationships (
    relationship_id BIGINT PRIMARY KEY,
    source_id BIGINT REFERENCES snomed_concepts(concept_id),
    destination_id BIGINT REFERENCES snomed_concepts(concept_id),
    relationship_group INTEGER,
    type_id BIGINT,
    active BOOLEAN,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **4.3.4 LOINC Codes**

```sql
-- LOINC Codes table
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

#### **4.3.5 RxNorm Codes**

```sql
-- RxNorm Codes table
CREATE TABLE rxnorm_codes (
    rxcui VARCHAR(15) PRIMARY KEY,
    name TEXT NOT NULL,
    tty VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **4.4 Patient Management**

```sql
-- Patients table
CREATE TABLE patients (
    patient_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    identifiers JSONB, -- E.g., [{"system": "Aadhar", "value": "123456789012"}]
    active BOOLEAN DEFAULT TRUE,
    name JSONB NOT NULL, -- FHIR HumanName structure
    telecom JSONB, -- Contact points
    gender VARCHAR(20) CHECK (gender IN ('male', 'female', 'other', 'unknown')),
    birth_date DATE,
    address JSONB, -- FHIR Address structure
    marital_status VARCHAR(50),
    photo BYTEA,
    communication JSONB, -- Languages and preferences
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index on patient names
CREATE INDEX idx_patients_name ON patients USING gin ((name) gin_trgm_ops);
```

---

### **4.5 Appointment Scheduling**

```sql
-- Appointments table
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
    participant JSONB NOT NULL, -- List of participants
    reason_code JSONB, -- List of CodeableConcepts
    reason_reference JSONB, -- References
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index on appointment times
CREATE INDEX idx_appointments_start_end ON appointments (start, end);
```

---

### **4.6 Clinical Documentation**

#### **4.6.1 Observations**

```sql
-- Observations table
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

#### **4.6.2 Conditions**

```sql
-- Conditions table
CREATE TABLE conditions (
    condition_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    patient_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    clinical_status VARCHAR(50),
    verification_status VARCHAR(50),
    category JSONB, -- List of CodeableConcepts
    severity JSONB, -- CodeableConcept
    code JSONB NOT NULL, -- CodeableConcept (e.g., SNOMED CT code)
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

#### **4.6.3 Procedures**

```sql
-- Procedures table
CREATE TABLE procedures (
    procedure_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status VARCHAR(50) CHECK (status IN ('preparation', 'in-progress', 'not-done', 'on-hold', 'stopped', 'completed', 'entered-in-error', 'unknown')) NOT NULL,
    category JSONB, -- CodeableConcept
    code JSONB NOT NULL, -- CodeableConcept (e.g., CPT code)
    subject_id UUID REFERENCES patients(patient_id) ON DELETE CASCADE,
    encounter_id UUID, -- Reference to an encounter
    performed_datetime TIMESTAMP WITH TIME ZONE,
    performer JSONB, -- List of performers
    reason_code JSONB, -- List of CodeableConcepts
    body_site JSONB, -- List of body sites
    outcome JSONB, -- CodeableConcept
    report JSONB, -- References to diagnostic reports
    complication JSONB, -- List of CodeableConcepts
    follow_up JSONB, -- List of CodeableConcepts
    note JSONB, -- List of annotations
    focal_device JSONB, -- List of devices
    used_reference JSONB, -- References to resources
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **4.6.4 Clinical Notes (Compositions)**

```sql
-- Compositions table
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

### **4.7 Medication and ePrescribing**

#### **4.7.1 Medications**

```sql
-- Medications table
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

#### **4.7.2 Medication Requests**

```sql
-- Medication Requests table
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

### **4.8 Lab Orders and Results**

#### **4.8.1 Service Requests (Lab Orders)**

```sql
-- Service Requests table
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

#### **4.8.2 Diagnostic Reports (Lab Results)**

```sql
-- Diagnostic Reports table
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

### **4.9 Consent Management**

```sql
-- Consents table
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

### **4.10 Billing and Revenue Cycle Management**

```sql
-- Billing Records table
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

-- Billing Items table
CREATE TABLE billing_items (
    billing_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_id UUID REFERENCES billing_records(billing_id) ON DELETE CASCADE,
    code VARCHAR(10), -- References code tables
    code_type VARCHAR(10) CHECK (code_type IN ('ICD-10', 'CPT', 'HCPCS')),
    description TEXT,
    charge_amount NUMERIC(12,2) DEFAULT 0.00,
    quantity INTEGER DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **4.11 Audit Logging and Provenance**

#### **Audit Events**

```sql
-- Audit Events table
CREATE TABLE audit_events (
    audit_event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    type VARCHAR(50), -- CodeableConcept
    subtype JSONB, -- List of CodeableConcepts
    action VARCHAR(10) CHECK (action IN ('C', 'R', 'U', 'D', 'E')), -- Create, Read, Update, Delete, Execute
    recorded TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    outcome VARCHAR(50), -- Success, minor failure, etc.
    outcome_desc TEXT,
    agent JSONB NOT NULL, -- User details
    source JSONB NOT NULL, -- System details
    entity JSONB, -- Entity involved
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **Provenance**

```sql
-- Provenance table
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

### **4.12 FAX Integration**

#### **Documents**

```sql
-- Documents table
CREATE TABLE documents (
    document_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    patient_id UUID REFERENCES patients(patient_id) ON DELETE SET NULL,
    file_name VARCHAR(255),
    file_type VARCHAR(100),
    file_size INTEGER,
    content BYTEA, -- Alternatively, store file paths
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### **FAX Queue**

```sql
-- FAX Queue table
CREATE TABLE fax_queue (
    fax_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    document_id UUID REFERENCES documents(document_id) ON DELETE CASCADE,
    recipient_number VARCHAR(20),
    status VARCHAR(50) CHECK (status IN ('pending', 'sent', 'failed')) DEFAULT 'pending',
    attempts INTEGER DEFAULT 0,
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### **4.13 Secure Messaging**

```sql
-- Messages table
CREATE TABLE messages (
    message_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    sender_id UUID,
    recipient_id UUID,
    subject VARCHAR(255),
    body TEXT,
    attachments JSONB,
    is_read BOOLEAN DEFAULT FALSE,
    sent_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    FOREIGN KEY (sender_id) REFERENCES users(user_id) ON DELETE SET NULL,
    FOREIGN KEY (recipient_id) REFERENCES users(user_id) ON DELETE SET NULL
);
```

---

## **5. Compliance and Security Measures**

- **Encryption**: Use encryption at rest and in transit (e.g., SSL/TLS for connections).
- **Access Control**: Implement role-based access control and fine-grained permissions.
- **Audit Trails**: Maintain detailed logs of all access and actions.
- **Data Isolation**: Enforce tenant-level data isolation using schemas or row-level security.
- **Data Retention Policies**: Define policies for data archiving and deletion.
- **Regular Updates**: Keep coding systems and software up to date.
- **Disaster Recovery**: Implement robust backup and recovery procedures.
- **Business Associate Agreements**: Ensure compliance when using third-party services.

---

## **6. Data Loading Notes**

- **Official Sources**: Load code sets from official releases (e.g., WHO for ICD-10, AMA for CPT).
- **Licensing**: Be aware of licensing requirements for code sets like SNOMED CT and CPT.
- **Bulk Loading**: Use efficient methods (e.g., `COPY` command) for large datasets.
- **Data Validation**: Verify data integrity after loading.

---

## **7. Conclusion**

This comprehensive database schema integrates all required functionalities for the Infoctor EHR system, ensuring that:

- **Clinical Coding Systems** are fully integrated for interoperability and compliance.
- **FAX Integration** and **Secure Messaging** functionalities are implemented for efficient communication.
- **Compliance and Security** measures are embedded in the design.
- **FHIR Standards** are followed for resource structures to facilitate interoperability.

---

## **8. Next Steps**

1. **Implement the Schema**: Execute the SQL scripts in a PostgreSQL environment.
2. **Load Reference Data**: Populate coding system tables with official data.
3. **Develop Application Logic**: Build the application layers that interact with the database.
4. **Implement APIs**: Develop FHIR-compliant APIs for data exchange.
5. **Integrate Third-Party Services**: Set up FAX and secure messaging services.
6. **Testing**: Conduct thorough testing, including security and compliance checks.
7. **Deployment**: Deploy the system in a secure, compliant infrastructure.
8. **Training and Documentation**: Prepare user guides and training materials.

---

**Feel free to reach out if you need further assistance with implementation details, optimization strategies, or any other aspect of the database design.**
