# KYC Portal for Auto Loan Origination - Component Design

## Table of Contents
1. [Introduction](#1-introduction)
2. [Authentication Service](#2-authentication-service)
3. [Customer Service](#3-customer-service)
4. [Document Management Service](#4-document-management-service)
5. [Verification Service](#5-verification-service)
6. [Risk Assessment Service](#6-risk-assessment-service)
7. [Workflow Service](#7-workflow-service)
8. [Notification Service](#8-notification-service)
9. [Reporting Service](#9-reporting-service)
10. [Cross-Component Interactions](#10-cross-component-interactions)
11. [Data Consistency Patterns](#11-data-consistency-patterns)

## 1. Introduction

### 1.1 Purpose
This document provides detailed component designs for the KYC Portal for Auto Loan Origination. It includes domain models, API specifications, and interaction patterns for each component identified in the architecture.

### 1.2 Component Design Template
Each component section follows this standardized template:

1. **Component Overview**
   - Purpose and responsibilities
   - Key features
   - Dependencies on other components

2. **Domain Model**
   - Entity definitions
   - Attributes and relationships
   - Business rules and constraints

3. **API Specification**
   - Endpoint definitions
   - Request/response formats
   - Error handling

4. **Component Interactions**
   - Interaction with other components
   - Event publishing and subscription
   - External system integration

5. **Data Management**
   - Data ownership
   - Storage strategy
   - Caching approach

### 1.3 Design Principles
The component designs adhere to the following principles:

1. **Single Responsibility**: Each component has a well-defined and focused set of responsibilities
2. **API-First Design**: All functionality is exposed through well-defined APIs
3. **Domain-Driven Design**: Components are organized around business domains
4. **Loose Coupling**: Components interact through well-defined interfaces
5. **High Cohesion**: Related functionality is grouped together
6. **Statelessness**: Components are designed to be stateless where possible
7. **Resilience**: Components handle failures gracefully
8. **Security by Design**: Security is built into each component

## 2. Authentication Service

### 2.1 Component Overview

#### 2.1.1 Purpose and Responsibilities
The Authentication Service is responsible for user registration, authentication, session management, and access control. It serves as the entry point for user authentication and authorization across the system.

#### 2.1.2 Key Features
- User registration and account management
- Authentication using username/password and multi-factor authentication
- Integration with SSO provider
- Session management
- Role-based access control

#### 2.1.3 Dependencies
- Customer Service: For customer profile creation after registration
- Notification Service: For sending verification emails and OTPs
- External SSO Provider: For staff authentication

### 2.2 Domain Model

#### 2.2.1 User
```
User {
    id: UUID (PK)
    username: String
    email: String
    mobileNumber: String
    passwordHash: String
    salt: String
    status: Enum [ACTIVE, INACTIVE, LOCKED, PENDING_VERIFICATION]
    failedLoginAttempts: Integer
    lastLoginTimestamp: DateTime
    createdAt: DateTime
    updatedAt: DateTime
    userType: Enum [CUSTOMER, STAFF]
}
```

#### 2.2.2 Role
```
Role {
    id: UUID (PK)
    name: String
    description: String
    permissions: List<Permission>
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 2.2.3 Permission
```
Permission {
    id: UUID (PK)
    name: String
    description: String
    resource: String
    action: String
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 2.2.4 UserRole
```
UserRole {
    userId: UUID (FK to User)
    roleId: UUID (FK to Role)
    assignedAt: DateTime
    assignedBy: UUID
}
```

#### 2.2.5 Session
```
Session {
    id: UUID (PK)
    userId: UUID (FK to User)
    token: String
    ipAddress: String
    userAgent: String
    createdAt: DateTime
    expiresAt: DateTime
    lastActivityAt: DateTime
    status: Enum [ACTIVE, EXPIRED, REVOKED]
}
```

#### 2.2.6 VerificationToken
```
VerificationToken {
    id: UUID (PK)
    userId: UUID (FK to User)
    token: String
    type: Enum [EMAIL, MOBILE, PASSWORD_RESET]
    createdAt: DateTime
    expiresAt: DateTime
    usedAt: DateTime
    status: Enum [ACTIVE, USED, EXPIRED]
}
```

#### 2.2.7 Business Rules
- Passwords must meet complexity requirements (minimum 8 characters, including uppercase, lowercase, number, and special character)
- User accounts are locked after 5 consecutive failed login attempts
- Verification tokens expire after a specified period (email: 24 hours, OTP: 5 minutes)
- Sessions automatically expire after 15 minutes of inactivity
- Staff users must have at least one role assigned

### 2.3 API Specification

#### 2.3.1 User Registration and Management

##### Register User
- **Endpoint**: `POST /api/auth/register`
- **Description**: Registers a new user in the system
- **Request Body**:
  ```json
  {
    "username": "string",
    "email": "string",
    "mobileNumber": "string",
    "password": "string",
    "userType": "CUSTOMER"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "username": "string",
    "email": "string",
    "mobileNumber": "string",
    "status": "PENDING_VERIFICATION",
    "createdAt": "datetime"
  }
  ```
- **Error Responses**:
  - 400 Bad Request: Invalid input data
  - 409 Conflict: Username/email already exists

##### Verify Email
- **Endpoint**: `POST /api/auth/verify-email`
- **Description**: Verifies user's email using a token
- **Request Body**:
  ```json
  {
    "token": "string"
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "message": "Email verified successfully"
  }
  ```
- **Error Responses**:
  - 400 Bad Request: Invalid or expired token

##### Verify Mobile
- **Endpoint**: `POST /api/auth/verify-mobile`
- **Description**: Verifies user's mobile number using OTP
- **Request Body**:
  ```json
  {
    "mobileNumber": "string",
    "otp": "string"
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "message": "Mobile number verified successfully"
  }
  ```
- **Error Responses**:
  - 400 Bad Request: Invalid or expired OTP

##### Request Password Reset
- **Endpoint**: `POST /api/auth/request-password-reset`
- **Description**: Initiates password reset process
- **Request Body**:
  ```json
  {
    "email": "string"
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "message": "Password reset instructions sent"
  }
  ```

##### Reset Password
- **Endpoint**: `POST /api/auth/reset-password`
- **Description**: Resets user password using a token
- **Request Body**:
  ```json
  {
    "token": "string",
    "newPassword": "string"
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "message": "Password reset successfully"
  }
  ```
- **Error Responses**:
  - 400 Bad Request: Invalid or expired token, password doesn't meet requirements

#### 2.3.2 Authentication

##### Login
- **Endpoint**: `POST /api/auth/login`
- **Description**: Authenticates a user and creates a session
- **Request Body**:
  ```json
  {
    "username": "string",
    "password": "string"
  }
  ```
- **Response**:
  ```json
  {
    "token": "string",
    "expiresAt": "datetime",
    "user": {
      "id": "uuid",
      "username": "string",
      "email": "string",
      "userType": "string",
      "roles": ["string"]
    }
  }
  ```
- **Error Responses**:
  - 401 Unauthorized: Invalid credentials
  - 403 Forbidden: Account locked or inactive

##### Request MFA Code
- **Endpoint**: `POST /api/auth/mfa/request`
- **Description**: Requests a multi-factor authentication code
- **Request Body**:
  ```json
  {
    "userId": "uuid",
    "method": "EMAIL|SMS"
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "message": "MFA code sent"
  }
  ```

##### Verify MFA Code
- **Endpoint**: `POST /api/auth/mfa/verify`
- **Description**: Verifies a multi-factor authentication code
- **Request Body**:
  ```json
  {
    "userId": "uuid",
    "code": "string"
  }
  ```
- **Response**:
  ```json
  {
    "token": "string",
    "expiresAt": "datetime"
  }
  ```
- **Error Responses**:
  - 401 Unauthorized: Invalid code

##### Logout
- **Endpoint**: `POST /api/auth/logout`
- **Description**: Invalidates the current session
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "success": true,
    "message": "Logged out successfully"
  }
  ```

#### 2.3.3 Role and Permission Management

##### Get Roles
- **Endpoint**: `GET /api/auth/roles`
- **Description**: Retrieves all roles
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "roles": [
      {
        "id": "uuid",
        "name": "string",
        "description": "string",
        "permissions": [
          {
            "id": "uuid",
            "name": "string",
            "resource": "string",
            "action": "string"
          }
        ]
      }
    ]
  }
  ```

##### Create Role
- **Endpoint**: `POST /api/auth/roles`
- **Description**: Creates a new role
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "name": "string",
    "description": "string",
    "permissions": ["uuid"]
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "permissions": [
      {
        "id": "uuid",
        "name": "string",
        "resource": "string",
        "action": "string"
      }
    ]
  }
  ```

##### Assign Role to User
- **Endpoint**: `POST /api/auth/users/{userId}/roles`
- **Description**: Assigns roles to a user
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "roles": ["uuid"]
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "message": "Roles assigned successfully"
  }
  ```

#### 2.3.4 Session Management

##### Get Active Sessions
- **Endpoint**: `GET /api/auth/sessions`
- **Description**: Retrieves all active sessions for the current user
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "sessions": [
      {
        "id": "uuid",
        "ipAddress": "string",
        "userAgent": "string",
        "createdAt": "datetime",
        "lastActivityAt": "datetime",
        "current": true
      }
    ]
  }
  ```

##### Revoke Session
- **Endpoint**: `DELETE /api/auth/sessions/{sessionId}`
- **Description**: Revokes a specific session
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "success": true,
    "message": "Session revoked successfully"
  }
  ```

#### 2.3.5 SSO Integration

##### Initiate SSO Login
- **Endpoint**: `GET /api/auth/sso/login`
- **Description**: Initiates SSO login process
- **Query Parameters**:
  - redirectUrl: URL to redirect after successful authentication
- **Response**: Redirects to SSO provider

##### SSO Callback
- **Endpoint**: `GET /api/auth/sso/callback`
- **Description**: Handles callback from SSO provider
- **Query Parameters**:
  - code: Authorization code from SSO provider
  - state: State parameter for security
- **Response**: Redirects to the original redirectUrl with token

### 2.4 Component Interactions

#### 2.4.1 Interaction with Customer Service
- After successful registration, the Authentication Service calls the Customer Service to create a customer profile
- The Authentication Service provides user identity information to the Customer Service

#### 2.4.2 Interaction with Notification Service
- The Authentication Service uses the Notification Service to send:
  - Email verification links
  - Mobile verification OTPs
  - Password reset links
  - Multi-factor authentication codes

#### 2.4.3 Integration with SSO Provider
- The Authentication Service integrates with the external SSO provider for staff authentication
- It handles the OAuth/SAML flow with the SSO provider
- It maps SSO user attributes to internal user roles and permissions

#### 2.4.4 Event Publishing
- User Registered: Published when a new user registers
- User Verified: Published when a user completes verification
- User Locked: Published when a user account is locked due to failed login attempts
- Role Assigned: Published when roles are assigned to a user

### 2.5 Data Management

#### 2.5.1 Data Ownership
The Authentication Service owns:
- User accounts and credentials
- Roles and permissions
- Sessions
- Verification tokens

#### 2.5.2 Storage Strategy
- User data stored in PostgreSQL database
- Password hashes stored using bcrypt with appropriate work factor
- Session tokens stored in Redis for fast access and automatic expiration
- Verification tokens stored in Redis with TTL

#### 2.5.3 Caching Approach
- Role and permission data cached in Redis
- Active sessions cached in Redis
- Cache invalidation on role/permission changes

## 3. Customer Service

### 3.1 Component Overview

#### 3.1.1 Purpose and Responsibilities
The Customer Service manages customer profiles and related information. It serves as the system of record for customer data and provides APIs for customer data management.

#### 3.1.2 Key Features
- Customer profile management
- Customer data validation
- Customer search and retrieval
- Customer data history tracking

#### 3.1.3 Dependencies
- Authentication Service: For user identity information
- Document Management Service: For accessing customer documents
- Notification Service: For customer communications

### 3.2 Domain Model

#### 3.2.1 Customer
```
Customer {
    id: UUID (PK)
    userId: UUID (FK to User)
    firstName: String
    middleName: String (optional)
    lastName: String
    dateOfBirth: Date
    gender: Enum [MALE, FEMALE, OTHER]
    nationality: String
    taxId: String (PAN number)
    aadhaarNumber: String (masked)
    createdAt: DateTime
    updatedAt: DateTime
    status: Enum [ACTIVE, INACTIVE, BLOCKED]
}
```

#### 3.2.2 ContactInformation
```
ContactInformation {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    email: String
    mobileNumber: String
    alternateMobileNumber: String (optional)
    addressLine1: String
    addressLine2: String (optional)
    city: String
    state: String
    postalCode: String
    country: String
    addressType: Enum [PERMANENT, CURRENT, OFFICE]
    isPrimaryAddress: Boolean
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 3.2.3 EmploymentDetails
```
EmploymentDetails {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    employmentType: Enum [SALARIED, SELF_EMPLOYED, BUSINESS, RETIRED, UNEMPLOYED]
    employerName: String
    designation: String
    employmentDuration: Integer (in months)
    monthlyIncome: Decimal
    officeAddressId: UUID (FK to ContactInformation)
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 3.2.4 CustomerNote
```
CustomerNote {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    createdBy: UUID
    note: String
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 3.2.5 CustomerDataChange
```
CustomerDataChange {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    fieldName: String
    oldValue: String
    newValue: String
    changedBy: UUID
    changeReason: String
    changedAt: DateTime
}
```

#### 3.2.6 Business Rules
- Customer email and mobile number must be verified before profile activation
- PAN and Aadhaar numbers must be validated through external verification services
- Customer must have at least one permanent address
- Customer data changes must be tracked for audit purposes
- Sensitive data like Aadhaar number must be stored in masked format

### 3.3 API Specification

#### 3.3.1 Customer Profile Management

##### Create Customer Profile
- **Endpoint**: `POST /api/customers`
- **Description**: Creates a new customer profile
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "userId": "uuid",
    "firstName": "string",
    "middleName": "string",
    "lastName": "string",
    "dateOfBirth": "date",
    "gender": "MALE|FEMALE|OTHER",
    "nationality": "string",
    "taxId": "string",
    "aadhaarNumber": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "userId": "uuid",
    "firstName": "string",
    "lastName": "string",
    "status": "ACTIVE",
    "createdAt": "datetime"
  }
  ```
- **Error Responses**:
  - 400 Bad Request: Invalid input data
  - 409 Conflict: Customer already exists for user

##### Get Customer Profile
- **Endpoint**: `GET /api/customers/{customerId}`
- **Description**: Retrieves a customer profile by ID
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "userId": "uuid",
    "firstName": "string",
    "middleName": "string",
    "lastName": "string",
    "dateOfBirth": "date",
    "gender": "string",
    "nationality": "string",
    "taxId": "string",
    "aadhaarNumber": "string (masked)",
    "status": "string",
    "createdAt": "datetime",
    "updatedAt": "datetime",
    "contactInformation": [
      {
        "id": "uuid",
        "email": "string",
        "mobileNumber": "string",
        "addressLine1": "string",
        "city": "string",
        "state": "string",
        "postalCode": "string",
        "addressType": "string",
        "isPrimaryAddress": true
      }
    ],
    "employmentDetails": {
      "id": "uuid",
      "employmentType": "string",
      "employerName": "string",
      "designation": "string",
      "monthlyIncome": "decimal"
    }
  }
  ```

##### Update Customer Profile
- **Endpoint**: `PUT /api/customers/{customerId}`
- **Description**: Updates a customer profile
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "firstName": "string",
    "middleName": "string",
    "lastName": "string",
    "dateOfBirth": "date",
    "gender": "string",
    "nationality": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "firstName": "string",
    "middleName": "string",
    "lastName": "string",
    "dateOfBirth": "date",
    "gender": "string",
    "nationality": "string",
    "updatedAt": "datetime"
  }
  ```

#### 3.3.2 Contact Information Management

##### Add Contact Information
- **Endpoint**: `POST /api/customers/{customerId}/contacts`
- **Description**: Adds contact information to a customer profile
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "email": "string",
    "mobileNumber": "string",
    "alternateMobileNumber": "string",
    "addressLine1": "string",
    "addressLine2": "string",
    "city": "string",
    "state": "string",
    "postalCode": "string",
    "country": "string",
    "addressType": "PERMANENT|CURRENT|OFFICE",
    "isPrimaryAddress": true
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "email": "string",
    "mobileNumber": "string",
    "addressLine1": "string",
    "city": "string",
    "state": "string",
    "postalCode": "string",
    "country": "string",
    "addressType": "string",
    "isPrimaryAddress": true,
    "createdAt": "datetime"
  }
  ```

##### Update Contact Information
- **Endpoint**: `PUT /api/customers/{customerId}/contacts/{contactId}`
- **Description**: Updates contact information
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "email": "string",
    "mobileNumber": "string",
    "alternateMobileNumber": "string",
    "addressLine1": "string",
    "addressLine2": "string",
    "city": "string",
    "state": "string",
    "postalCode": "string",
    "country": "string",
    "isPrimaryAddress": true
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "email": "string",
    "mobileNumber": "string",
    "addressLine1": "string",
    "city": "string",
    "state": "string",
    "postalCode": "string",
    "country": "string",
    "addressType": "string",
    "isPrimaryAddress": true,
    "updatedAt": "datetime"
  }
  ```

#### 3.3.3 Employment Details Management

##### Add Employment Details
- **Endpoint**: `POST /api/customers/{customerId}/employment`
- **Description**: Adds employment details to a customer profile
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "employmentType": "SALARIED|SELF_EMPLOYED|BUSINESS|RETIRED|UNEMPLOYED",
    "employerName": "string",
    "designation": "string",
    "employmentDuration": "integer",
    "monthlyIncome": "decimal",
    "officeAddressId": "uuid"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "employmentType": "string",
    "employerName": "string",
    "designation": "string",
    "employmentDuration": "integer",
    "monthlyIncome": "decimal",
    "officeAddressId": "uuid",
    "createdAt": "datetime"
  }
  ```

#### 3.3.4 Customer Search

##### Search Customers
- **Endpoint**: `GET /api/customers/search`
- **Description**: Searches for customers based on criteria
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - name: Customer name (partial match)
  - email: Customer email
  - mobileNumber: Customer mobile number
  - taxId: PAN number
  - aadhaarNumber: Aadhaar number (exact match)
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "customers": [
      {
        "id": "uuid",
        "firstName": "string",
        "lastName": "string",
        "email": "string",
        "mobileNumber": "string",
        "status": "string"
      }
    ]
  }
  ```

#### 3.3.5 Customer Notes

##### Add Customer Note
- **Endpoint**: `POST /api/customers/{customerId}/notes`
- **Description**: Adds a note to a customer profile
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "note": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "createdBy": "uuid",
    "note": "string",
    "createdAt": "datetime"
  }
  ```

##### Get Customer Notes
- **Endpoint**: `GET /api/customers/{customerId}/notes`
- **Description**: Retrieves notes for a customer
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "notes": [
      {
        "id": "uuid",
        "createdBy": "uuid",
        "createdByName": "string",
        "note": "string",
        "createdAt": "datetime"
      }
    ]
  }
  ```

### 3.4 Component Interactions

#### 3.4.1 Interaction with Authentication Service
- The Customer Service receives user identity information from the Authentication Service
- It listens for user registration and verification events

#### 3.4.2 Interaction with Document Management Service
- The Customer Service queries the Document Management Service for customer documents
- It receives document verification status updates

#### 3.4.3 Interaction with Verification Service
- The Customer Service requests verification of customer information (PAN, Aadhaar)
- It receives verification results and updates customer status accordingly

#### 3.4.4 Event Publishing
- Customer Created: Published when a new customer profile is created
- Customer Updated: Published when customer information is updated
- Customer Status Changed: Published when customer status changes

### 3.5 Data Management

#### 3.5.1 Data Ownership
The Customer Service owns:
- Customer profiles
- Contact information
- Employment details
- Customer notes
- Customer data change history

#### 3.5.2 Storage Strategy
- Customer data stored in PostgreSQL database
- Sensitive data (Aadhaar number) stored in encrypted format
- Customer search index maintained in Elasticsearch

#### 3.5.3 Caching Approach
- Frequently accessed customer profiles cached in Redis
- Cache invalidation on customer data updates
## 4. Document Management Service

### 4.1 Component Overview

#### 4.1.1 Purpose and Responsibilities
The Document Management Service handles all aspects of document upload, storage, retrieval, and lifecycle management. It ensures secure storage and efficient access to customer documents.

#### 4.1.2 Key Features
- Document upload and validation
- Document metadata management
- Document security scanning
- Document versioning
- Document lifecycle management
- Document expiration tracking

#### 4.1.3 Dependencies
- Customer Service: For customer information
- Verification Service: For document verification
- Notification Service: For document status notifications
- External security scanning service: For malware detection

### 4.2 Domain Model

#### 4.2.1 Document
```
Document {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    applicationId: UUID (optional, FK to Application)
    documentType: Enum [ID_PROOF, ADDRESS_PROOF, INCOME_PROOF, VEHICLE_INFO, OTHER]
    documentSubType: String (e.g., PAN_CARD, AADHAAR_CARD, PASSPORT)
    fileName: String
    fileSize: Long
    mimeType: String
    s3Key: String
    uploadedBy: UUID
    uploadedAt: DateTime
    expiryDate: Date (optional)
    status: Enum [UPLOADED, SCANNING, SCAN_FAILED, SCAN_PASSED, VERIFIED, REJECTED, EXPIRED]
    isLatestVersion: Boolean
    version: Integer
}
```

#### 4.2.2 DocumentMetadata
```
DocumentMetadata {
    id: UUID (PK)
    documentId: UUID (FK to Document)
    metadataKey: String
    metadataValue: String
    extractionConfidence: Float (optional)
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 4.2.3 DocumentRequest
```
DocumentRequest {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    applicationId: UUID (optional, FK to Application)
    documentType: Enum [ID_PROOF, ADDRESS_PROOF, INCOME_PROOF, VEHICLE_INFO, OTHER]
    documentSubType: String
    requestedBy: UUID
    requestedAt: DateTime
    reason: String
    status: Enum [PENDING, FULFILLED, EXPIRED]
    dueDate: DateTime
    fulfilledAt: DateTime (optional)
    fulfilledDocumentId: UUID (optional, FK to Document)
}
```

#### 4.2.4 DocumentScanResult
```
DocumentScanResult {
    id: UUID (PK)
    documentId: UUID (FK to Document)
    scanProvider: String
    scanTimestamp: DateTime
    scanStatus: Enum [CLEAN, INFECTED, ERROR]
    threatDetails: String (optional)
    scanDuration: Integer (in milliseconds)
}
```

#### 4.2.5 DocumentAuditLog
```
DocumentAuditLog {
    id: UUID (PK)
    documentId: UUID (FK to Document)
    action: Enum [UPLOAD, DOWNLOAD, VIEW, DELETE, STATUS_CHANGE]
    performedBy: UUID
    performedAt: DateTime
    details: String
    ipAddress: String
}
```

#### 4.2.6 Business Rules
- Documents must be scanned for malware before being made available
- Maximum file size is 10MB per document
- Supported file formats are PDF, JPEG, and PNG
- Document expiration must be tracked and notifications sent before expiry
- Document access must be logged for audit purposes
- Document versioning must maintain history of all document versions

### 4.3 API Specification

#### 4.3.1 Document Upload and Management

##### Upload Document
- **Endpoint**: `POST /api/documents`
- **Description**: Uploads a new document
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body** (multipart/form-data):
  - file: Document file
  - customerId: Customer ID
  - applicationId: Application ID (optional)
  - documentType: Document type
  - documentSubType: Document subtype
  - expiryDate: Document expiry date (optional)
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "documentType": "string",
    "documentSubType": "string",
    "fileName": "string",
    "fileSize": "long",
    "status": "UPLOADED",
    "uploadedAt": "datetime"
  }
  ```
- **Error Responses**:
  - 400 Bad Request: Invalid input or file too large
  - 415 Unsupported Media Type: Unsupported file format

##### Get Document
- **Endpoint**: `GET /api/documents/{documentId}`
- **Description**: Retrieves document metadata
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "documentType": "string",
    "documentSubType": "string",
    "fileName": "string",
    "fileSize": "long",
    "mimeType": "string",
    "uploadedBy": "uuid",
    "uploadedAt": "datetime",
    "expiryDate": "date",
    "status": "string",
    "version": "integer",
    "metadata": [
      {
        "metadataKey": "string",
        "metadataValue": "string",
        "extractionConfidence": "float"
      }
    ]
  }
  ```

##### Download Document
- **Endpoint**: `GET /api/documents/{documentId}/content`
- **Description**: Downloads the document content
- **Headers**:
  - Authorization: Bearer {token}
- **Response**: Document file with appropriate Content-Type header
- **Error Responses**:
  - 404 Not Found: Document not found
  - 403 Forbidden: Insufficient permissions

##### Update Document Status
- **Endpoint**: `PATCH /api/documents/{documentId}/status`
- **Description**: Updates the status of a document
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "status": "VERIFIED|REJECTED",
    "reason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "string",
    "updatedAt": "datetime"
  }
  ```

##### Replace Document
- **Endpoint**: `PUT /api/documents/{documentId}`
- **Description**: Replaces an existing document with a new version
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body** (multipart/form-data):
  - file: Document file
  - reason: Reason for replacement
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "documentType": "string",
    "documentSubType": "string",
    "fileName": "string",
    "status": "UPLOADED",
    "uploadedAt": "datetime",
    "version": "integer"
  }
  ```

#### 4.3.2 Document Search and Retrieval

##### Search Documents
- **Endpoint**: `GET /api/documents/search`
- **Description**: Searches for documents based on criteria
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - customerId: Customer ID
  - applicationId: Application ID
  - documentType: Document type
  - documentSubType: Document subtype
  - status: Document status
  - fromDate: Upload date range start
  - toDate: Upload date range end
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "documents": [
      {
        "id": "uuid",
        "customerId": "uuid",
        "documentType": "string",
        "documentSubType": "string",
        "fileName": "string",
        "status": "string",
        "uploadedAt": "datetime"
      }
    ]
  }
  ```

##### Get Document Versions
- **Endpoint**: `GET /api/documents/{documentId}/versions`
- **Description**: Retrieves all versions of a document
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "documentId": "uuid",
    "versions": [
      {
        "id": "uuid",
        "version": "integer",
        "fileName": "string",
        "uploadedBy": "uuid",
        "uploadedAt": "datetime",
        "status": "string",
        "isLatestVersion": true
      }
    ]
  }
  ```

#### 4.3.3 Document Request Management

##### Request Document
- **Endpoint**: `POST /api/document-requests`
- **Description**: Creates a request for a customer to upload a document
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "applicationId": "uuid",
    "documentType": "string",
    "documentSubType": "string",
    "reason": "string",
    "dueDate": "datetime"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "documentType": "string",
    "documentSubType": "string",
    "requestedBy": "uuid",
    "requestedAt": "datetime",
    "status": "PENDING",
    "dueDate": "datetime"
  }
  ```

##### Get Document Requests
- **Endpoint**: `GET /api/document-requests`
- **Description**: Retrieves document requests based on criteria
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - customerId: Customer ID
  - applicationId: Application ID
  - status: Request status
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "requests": [
      {
        "id": "uuid",
        "customerId": "uuid",
        "applicationId": "uuid",
        "documentType": "string",
        "documentSubType": "string",
        "requestedBy": "uuid",
        "requestedAt": "datetime",
        "status": "string",
        "dueDate": "datetime"
      }
    ]
  }
  ```

#### 4.3.4 Document Metadata Management

##### Add Document Metadata
- **Endpoint**: `POST /api/documents/{documentId}/metadata`
- **Description**: Adds metadata to a document
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "metadata": [
      {
        "metadataKey": "string",
        "metadataValue": "string",
        "extractionConfidence": "float"
      }
    ]
  }
  ```
- **Response**:
  ```json
  {
    "documentId": "uuid",
    "metadata": [
      {
        "metadataKey": "string",
        "metadataValue": "string",
        "extractionConfidence": "float",
        "createdAt": "datetime"
      }
    ]
  }
  ```

### 4.4 Component Interactions

#### 4.4.1 Interaction with Customer Service
- The Document Management Service validates customer IDs with the Customer Service
- It provides document information to the Customer Service when requested

#### 4.4.2 Interaction with Verification Service
- The Document Management Service sends documents to the Verification Service for data extraction and verification
- It receives verification results and updates document status

#### 4.4.3 Interaction with Notification Service
- The Document Management Service triggers notifications for:
  - Document status changes
  - Document expiration warnings
  - Document request notifications

#### 4.4.4 Integration with Security Scanning Service
- Documents are sent to an external or internal security scanning service
- Scan results determine if documents are safe to store and process

#### 4.4.5 Event Publishing
- Document Uploaded: Published when a new document is uploaded
- Document Status Changed: Published when document status changes
- Document Requested: Published when a document is requested
- Document Expiring Soon: Published when a document is approaching expiration

### 4.5 Data Management

#### 4.5.1 Data Ownership
The Document Management Service owns:
- Document metadata
- Document requests
- Document scan results
- Document audit logs

#### 4.5.2 Storage Strategy
- Document metadata stored in PostgreSQL database
- Document files stored in Amazon S3
- Document search index maintained in Elasticsearch
- Document audit logs stored in PostgreSQL with archiving to S3

#### 4.5.3 Caching Approach
- Document metadata cached in Redis
- S3 pre-signed URLs cached for frequent downloads
- Cache invalidation on document updates
## 5. Verification Service

### 5.1 Component Overview

#### 5.1.1 Purpose and Responsibilities
The Verification Service is responsible for verifying customer identity and documents. It integrates with external verification services, performs data extraction from documents, and manages the verification workflow.

#### 5.1.2 Key Features
- Document data extraction using OCR/AI
- Identity verification through external services
- PAN and Aadhaar verification
- Watchlist screening
- Biometric verification
- Verification result management

#### 5.1.3 Dependencies
- Document Management Service: For accessing documents
- Customer Service: For customer information
- External verification services:
  - PAN verification service
  - Aadhaar verification API
  - Watchlist screening services
  - Facial recognition service

### 5.2 Domain Model

#### 5.2.1 VerificationRequest
```
VerificationRequest {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    applicationType: String
    requestType: Enum [IDENTITY, ADDRESS, DOCUMENT, WATCHLIST, BIOMETRIC]
    status: Enum [PENDING, IN_PROGRESS, COMPLETED, FAILED]
    priority: Enum [LOW, MEDIUM, HIGH]
    createdAt: DateTime
    updatedAt: DateTime
    completedAt: DateTime (optional)
    requestedBy: UUID
}
```

#### 5.2.2 VerificationResult
```
VerificationResult {
    id: UUID (PK)
    verificationRequestId: UUID (FK to VerificationRequest)
    resultType: Enum [IDENTITY, ADDRESS, DOCUMENT, WATCHLIST, BIOMETRIC]
    status: Enum [VERIFIED, REJECTED, INCONCLUSIVE]
    confidence: Float
    details: JSON
    verifiedBy: String (system or user ID)
    verifiedAt: DateTime
}
```

#### 5.2.3 DocumentExtraction
```
DocumentExtraction {
    id: UUID (PK)
    documentId: UUID (FK to Document)
    extractionEngine: String
    extractedData: JSON
    confidenceScores: JSON
    extractedAt: DateTime
    processingTime: Integer (in milliseconds)
    status: Enum [SUCCESSFUL, PARTIAL, FAILED]
}
```

#### 5.2.4 IdentityVerification
```
IdentityVerification {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    verificationType: Enum [PAN, AADHAAR, PASSPORT, DRIVING_LICENSE]
    referenceNumber: String
    verificationService: String
    requestPayload: JSON
    responsePayload: JSON
    status: Enum [VERIFIED, REJECTED, ERROR]
    verifiedAt: DateTime
    externalReferenceId: String
}
```

#### 5.2.5 BiometricVerification
```
BiometricVerification {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    biometricType: Enum [FACIAL, FINGERPRINT, IRIS]
    referenceDocumentId: UUID (FK to Document)
    capturedBiometricId: UUID
    matchScore: Float
    livenessScore: Float (optional)
    status: Enum [MATCHED, NOT_MATCHED, ERROR]
    verifiedAt: DateTime
}
```

#### 5.2.6 WatchlistScreening
```
WatchlistScreening {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    watchlistType: Enum [PEP, SANCTIONS, ADVERSE_MEDIA]
    screeningService: String
    matchStatus: Enum [NO_MATCH, POTENTIAL_MATCH, CONFIRMED_MATCH]
    matchDetails: JSON
    screenedAt: DateTime
    reviewStatus: Enum [PENDING_REVIEW, REVIEWED, FALSE_POSITIVE]
    reviewedBy: UUID (optional)
    reviewedAt: DateTime (optional)
}
```

#### 5.2.7 Business Rules
- All verification requests must be traceable to the requesting user or system
- Verification results must include confidence scores where applicable
- Biometric verification must include liveness detection
- Watchlist screening results must be reviewed by staff for potential matches
- Verification history must be maintained for audit purposes
- External service integration must handle timeouts and failures gracefully

### 5.3 API Specification

#### 5.3.1 Verification Requests

##### Create Verification Request
- **Endpoint**: `POST /api/verifications`
- **Description**: Creates a new verification request
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "applicationType": "string",
    "requestType": "IDENTITY|ADDRESS|DOCUMENT|WATCHLIST|BIOMETRIC",
    "priority": "LOW|MEDIUM|HIGH",
    "metadata": {
      "documentId": "uuid",
      "verificationType": "string",
      "referenceNumber": "string"
    }
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "requestType": "string",
    "status": "PENDING",
    "createdAt": "datetime"
  }
  ```

##### Get Verification Request
- **Endpoint**: `GET /api/verifications/{requestId}`
- **Description**: Retrieves a verification request by ID
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationType": "string",
    "requestType": "string",
    "status": "string",
    "priority": "string",
    "createdAt": "datetime",
    "updatedAt": "datetime",
    "completedAt": "datetime",
    "requestedBy": "uuid",
    "results": [
      {
        "id": "uuid",
        "resultType": "string",
        "status": "string",
        "confidence": "float",
        "verifiedAt": "datetime"
      }
    ]
  }
  ```

##### Search Verification Requests
- **Endpoint**: `GET /api/verifications`
- **Description**: Searches for verification requests based on criteria
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - customerId: Customer ID
  - requestType: Request type
  - status: Request status
  - fromDate: Creation date range start
  - toDate: Creation date range end
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "requests": [
      {
        "id": "uuid",
        "customerId": "uuid",
        "requestType": "string",
        "status": "string",
        "createdAt": "datetime",
        "completedAt": "datetime"
      }
    ]
  }
  ```

#### 5.3.2 Document Extraction

##### Extract Document Data
- **Endpoint**: `POST /api/verifications/extract`
- **Description**: Extracts data from a document
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "documentId": "uuid",
    "extractionType": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "documentId": "uuid",
    "status": "string",
    "extractedData": {
      "field1": "value1",
      "field2": "value2"
    },
    "confidenceScores": {
      "field1": 0.95,
      "field2": 0.87
    },
    "extractedAt": "datetime"
  }
  ```

##### Get Document Extraction
- **Endpoint**: `GET /api/verifications/extractions/{extractionId}`
- **Description**: Retrieves document extraction results
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "documentId": "uuid",
    "extractionEngine": "string",
    "extractedData": {
      "field1": "value1",
      "field2": "value2"
    },
    "confidenceScores": {
      "field1": 0.95,
      "field2": 0.87
    },
    "extractedAt": "datetime",
    "processingTime": "integer",
    "status": "string"
  }
  ```

#### 5.3.3 Identity Verification

##### Verify PAN
- **Endpoint**: `POST /api/verifications/pan`
- **Description**: Verifies PAN card details with the Income Tax Department
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "panNumber": "string",
    "name": "string",
    "dateOfBirth": "date"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "verificationType": "PAN",
    "status": "string",
    "verifiedAt": "datetime",
    "details": {
      "nameMatch": true,
      "dobMatch": true
    }
  }
  ```

##### Verify Aadhaar
- **Endpoint**: `POST /api/verifications/aadhaar`
- **Description**: Verifies Aadhaar details with UIDAI
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "aadhaarNumber": "string",
    "name": "string",
    "verificationType": "DEMOGRAPHIC|OTP|BIOMETRIC"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "verificationType": "AADHAAR",
    "status": "string",
    "verifiedAt": "datetime",
    "externalReferenceId": "string"
  }
  ```

#### 5.3.4 Watchlist Screening

##### Screen Customer
- **Endpoint**: `POST /api/verifications/watchlist`
- **Description**: Screens customer against watchlists
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "watchlistTypes": ["PEP", "SANCTIONS", "ADVERSE_MEDIA"],
    "customerData": {
      "name": "string",
      "dateOfBirth": "date",
      "nationality": "string",
      "identifiers": [
        {
          "type": "string",
          "value": "string"
        }
      ]
    }
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "screeningResults": [
      {
        "watchlistType": "string",
        "matchStatus": "string",
        "matchDetails": [
          {
            "listName": "string",
            "matchScore": "float",
            "entityDetails": {
              "name": "string",
              "aliases": ["string"],
              "dateOfBirth": "date",
              "nationality": "string"
            }
          }
        ],
        "screenedAt": "datetime"
      }
    ]
  }
  ```

##### Review Watchlist Match
- **Endpoint**: `POST /api/verifications/watchlist/{screeningId}/review`
- **Description**: Reviews a watchlist screening match
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "reviewStatus": "REVIEWED|FALSE_POSITIVE",
    "comments": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "reviewStatus": "string",
    "reviewedBy": "uuid",
    "reviewedAt": "datetime"
  }
  ```

#### 5.3.5 Biometric Verification

##### Facial Recognition
- **Endpoint**: `POST /api/verifications/facial`
- **Description**: Performs facial recognition between selfie and ID document
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body** (multipart/form-data):
  - selfieImage: Image file
  - documentId: ID document containing face
  - customerId: Customer ID
  - livenessCheck: Boolean
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "biometricType": "FACIAL",
    "matchScore": "float",
    "livenessScore": "float",
    "status": "string",
    "verifiedAt": "datetime"
  }
  ```

##### Liveness Detection
- **Endpoint**: `POST /api/verifications/liveness`
- **Description**: Performs liveness detection on a video or series of images
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body** (multipart/form-data):
  - mediaFile: Video or image file
  - customerId: Customer ID
  - challengeType: Type of liveness challenge
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "livenessScore": "float",
    "status": "string",
    "details": {
      "challengeCompleted": true,
      "detectedSpoofingAttempts": false
    },
    "verifiedAt": "datetime"
  }
  ```

### 5.4 Component Interactions

#### 5.4.1 Interaction with Document Management Service
- The Verification Service retrieves documents from the Document Management Service
- It updates document verification status after processing

#### 5.4.2 Interaction with Customer Service
- The Verification Service retrieves customer information for verification
- It updates customer verification status based on verification results

#### 5.4.3 Integration with External Verification Services
- PAN Verification Service: For verifying PAN card details
- Aadhaar API: For verifying Aadhaar details
- Watchlist Services: For screening against sanctions lists and PEP databases
- OCR/AI Services: For document data extraction

#### 5.4.4 Event Publishing
- Verification Requested: Published when a verification request is created
- Verification Completed: Published when a verification is completed
- Verification Failed: Published when a verification fails
- Watchlist Match Found: Published when a potential watchlist match is found

### 5.5 Data Management

#### 5.5.1 Data Ownership
The Verification Service owns:
- Verification requests and results
- Document extraction data
- Identity verification records
- Biometric verification records
- Watchlist screening results

#### 5.5.2 Storage Strategy
- Verification data stored in PostgreSQL database
- Extracted document data stored in PostgreSQL with large fields in JSON format
- Biometric templates stored securely with encryption
- Verification history maintained with appropriate retention periods

#### 5.5.3 Caching Approach
- Frequently accessed verification results cached in Redis
- External service responses cached where appropriate
- Cache invalidation on verification updates
## 6. Risk Assessment Service

### 6.1 Component Overview

#### 6.1.1 Purpose and Responsibilities
The Risk Assessment Service evaluates customer risk levels based on various factors including verification results, customer profile, and transaction history. It identifies high-risk cases requiring enhanced due diligence and provides risk scoring for decision-making.

#### 6.1.2 Key Features
- Risk score calculation
- Risk categorization
- PEP identification
- Enhanced due diligence workflow
- Ongoing risk monitoring
- Risk assessment history

#### 6.1.3 Dependencies
- Customer Service: For customer information
- Verification Service: For verification results
- Document Management Service: For document information
- Workflow Service: For triggering enhanced due diligence workflows

### 6.2 Domain Model

#### 6.2.1 RiskAssessment
```
RiskAssessment {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    applicationId: UUID (optional, FK to Application)
    assessmentType: Enum [INITIAL, PERIODIC, TRIGGERED]
    overallRiskScore: Float
    riskCategory: Enum [LOW, MEDIUM, HIGH]
    assessedAt: DateTime
    assessedBy: String (system or user ID)
    previousRiskCategory: Enum [LOW, MEDIUM, HIGH] (optional)
    nextAssessmentDate: Date (optional)
    status: Enum [ACTIVE, SUPERSEDED]
}
```

#### 6.2.2 RiskFactor
```
RiskFactor {
    id: UUID (PK)
    riskAssessmentId: UUID (FK to RiskAssessment)
    factorType: String
    factorName: String
    weight: Float
    score: Float
    contributionToOverall: Float
    evidence: JSON
    createdAt: DateTime
}
```

#### 6.2.3 RiskRule
```
RiskRule {
    id: UUID (PK)
    name: String
    description: String
    factorType: String
    evaluationLogic: JSON
    weight: Float
    version: Integer
    status: Enum [ACTIVE, INACTIVE]
    createdAt: DateTime
    updatedAt: DateTime
    createdBy: UUID
}
```

#### 6.2.4 EnhancedDueDiligence
```
EnhancedDueDiligence {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    applicationId: UUID (optional, FK to Application)
    riskAssessmentId: UUID (FK to RiskAssessment)
    reason: String
    status: Enum [PENDING, IN_PROGRESS, COMPLETED, REJECTED]
    assignedTo: UUID (optional)
    startedAt: DateTime
    completedAt: DateTime (optional)
    decision: Enum [APPROVED, REJECTED, ESCALATED] (optional)
    decisionBy: UUID (optional)
    decisionReason: String (optional)
}
```

#### 6.2.5 EnhancedDueDiligenceTask
```
EnhancedDueDiligenceTask {
    id: UUID (PK)
    eddId: UUID (FK to EnhancedDueDiligence)
    taskType: String
    description: String
    status: Enum [PENDING, COMPLETED, SKIPPED]
    assignedTo: UUID (optional)
    completedAt: DateTime (optional)
    completedBy: UUID (optional)
    notes: String (optional)
}
```

#### 6.2.6 RiskChangeEvent
```
RiskChangeEvent {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    eventType: String
    previousRiskCategory: Enum [LOW, MEDIUM, HIGH]
    newRiskCategory: Enum [LOW, MEDIUM, HIGH]
    changeReason: String
    triggeredBy: String
    eventTime: DateTime
}
```

#### 6.2.7 Business Rules
- Risk scores must be between 0 and 100
- Risk categories are defined as: Low (0-30), Medium (31-70), High (71-100)
- PEP status automatically triggers high-risk categorization
- Risk assessments must be performed at customer onboarding
- Periodic risk reassessments must be scheduled based on risk category
- Changes in risk category must be documented with reasons
- Enhanced due diligence is required for all high-risk customers

### 6.3 API Specification

#### 6.3.1 Risk Assessment

##### Calculate Risk Score
- **Endpoint**: `POST /api/risk/assessments`
- **Description**: Calculates risk score for a customer
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "applicationId": "uuid",
    "assessmentType": "INITIAL|PERIODIC|TRIGGERED",
    "triggerReason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "overallRiskScore": "float",
    "riskCategory": "string",
    "assessedAt": "datetime",
    "factors": [
      {
        "factorType": "string",
        "factorName": "string",
        "score": "float",
        "contributionToOverall": "float"
      }
    ]
  }
  ```

##### Get Risk Assessment
- **Endpoint**: `GET /api/risk/assessments/{assessmentId}`
- **Description**: Retrieves a risk assessment by ID
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "assessmentType": "string",
    "overallRiskScore": "float",
    "riskCategory": "string",
    "assessedAt": "datetime",
    "assessedBy": "string",
    "previousRiskCategory": "string",
    "nextAssessmentDate": "date",
    "status": "string",
    "factors": [
      {
        "factorType": "string",
        "factorName": "string",
        "weight": "float",
        "score": "float",
        "contributionToOverall": "float",
        "evidence": {
          "key1": "value1",
          "key2": "value2"
        }
      }
    ]
  }
  ```

##### Get Customer Risk History
- **Endpoint**: `GET /api/risk/customers/{customerId}/history`
- **Description**: Retrieves risk assessment history for a customer
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - fromDate: Start date
  - toDate: End date
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "customerId": "uuid",
    "currentRiskCategory": "string",
    "assessments": [
      {
        "id": "uuid",
        "assessmentType": "string",
        "overallRiskScore": "float",
        "riskCategory": "string",
        "assessedAt": "datetime",
        "status": "string"
      }
    ],
    "riskChanges": [
      {
        "eventType": "string",
        "previousRiskCategory": "string",
        "newRiskCategory": "string",
        "changeReason": "string",
        "eventTime": "datetime"
      }
    ]
  }
  ```

#### 6.3.2 Risk Rules Management

##### Get Risk Rules
- **Endpoint**: `GET /api/risk/rules`
- **Description**: Retrieves all active risk rules
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - factorType: Filter by factor type
  - status: Filter by status
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "rules": [
      {
        "id": "uuid",
        "name": "string",
        "description": "string",
        "factorType": "string",
        "weight": "float",
        "version": "integer",
        "status": "string"
      }
    ]
  }
  ```

##### Create Risk Rule
- **Endpoint**: `POST /api/risk/rules`
- **Description**: Creates a new risk rule
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "name": "string",
    "description": "string",
    "factorType": "string",
    "evaluationLogic": {
      "conditions": [
        {
          "field": "string",
          "operator": "string",
          "value": "any"
        }
      ],
      "combinator": "AND|OR"
    },
    "weight": "float"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "factorType": "string",
    "weight": "float",
    "version": "integer",
    "status": "ACTIVE",
    "createdAt": "datetime",
    "createdBy": "uuid"
  }
  ```

##### Update Risk Rule
- **Endpoint**: `PUT /api/risk/rules/{ruleId}`
- **Description**: Updates an existing risk rule
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "description": "string",
    "evaluationLogic": {
      "conditions": [
        {
          "field": "string",
          "operator": "string",
          "value": "any"
        }
      ],
      "combinator": "AND|OR"
    },
    "weight": "float",
    "status": "ACTIVE|INACTIVE"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "factorType": "string",
    "weight": "float",
    "version": "integer",
    "status": "string",
    "updatedAt": "datetime"
  }
  ```

#### 6.3.3 Enhanced Due Diligence

##### Initiate Enhanced Due Diligence
- **Endpoint**: `POST /api/risk/edd`
- **Description**: Initiates enhanced due diligence for a customer
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "applicationId": "uuid",
    "riskAssessmentId": "uuid",
    "reason": "string",
    "assignedTo": "uuid"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "reason": "string",
    "status": "PENDING",
    "assignedTo": "uuid",
    "startedAt": "datetime",
    "tasks": [
      {
        "id": "uuid",
        "taskType": "string",
        "description": "string",
        "status": "PENDING"
      }
    ]
  }
  ```

##### Get Enhanced Due Diligence
- **Endpoint**: `GET /api/risk/edd/{eddId}`
- **Description**: Retrieves enhanced due diligence details
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationId": "uuid",
    "riskAssessmentId": "uuid",
    "reason": "string",
    "status": "string",
    "assignedTo": "uuid",
    "startedAt": "datetime",
    "completedAt": "datetime",
    "decision": "string",
    "decisionBy": "uuid",
    "decisionReason": "string",
    "tasks": [
      {
        "id": "uuid",
        "taskType": "string",
        "description": "string",
        "status": "string",
        "assignedTo": "uuid",
        "completedAt": "datetime",
        "completedBy": "uuid",
        "notes": "string"
      }
    ]
  }
  ```

##### Complete EDD Task
- **Endpoint**: `PUT /api/risk/edd/{eddId}/tasks/{taskId}`
- **Description**: Completes an enhanced due diligence task
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "status": "COMPLETED|SKIPPED",
    "notes": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "string",
    "completedAt": "datetime",
    "completedBy": "uuid",
    "notes": "string"
  }
  ```

##### Complete Enhanced Due Diligence
- **Endpoint**: `PUT /api/risk/edd/{eddId}/complete`
- **Description**: Completes enhanced due diligence with decision
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "decision": "APPROVED|REJECTED|ESCALATED",
    "decisionReason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "COMPLETED",
    "completedAt": "datetime",
    "decision": "string",
    "decisionBy": "uuid",
    "decisionReason": "string"
  }
  ```

#### 6.3.4 Risk Monitoring

##### Schedule Risk Reassessment
- **Endpoint**: `POST /api/risk/customers/{customerId}/schedule-reassessment`
- **Description**: Schedules a risk reassessment for a customer
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "scheduledDate": "date",
    "reason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "customerId": "uuid",
    "currentRiskCategory": "string",
    "nextAssessmentDate": "date",
    "reason": "string"
  }
  ```

##### Trigger Risk Reassessment
- **Endpoint**: `POST /api/risk/customers/{customerId}/trigger-reassessment`
- **Description**: Triggers an immediate risk reassessment
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "triggerReason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "assessmentId": "uuid",
    "customerId": "uuid",
    "triggerReason": "string",
    "status": "PENDING"
  }
  ```

### 6.4 Component Interactions

#### 6.4.1 Interaction with Customer Service
- The Risk Assessment Service retrieves customer profile information
- It updates customer risk status based on assessment results

#### 6.4.2 Interaction with Verification Service
- The Risk Assessment Service retrieves verification results for risk evaluation
- It considers verification outcomes in risk scoring

#### 6.4.3 Interaction with Workflow Service
- The Risk Assessment Service triggers enhanced due diligence workflows
- It receives workflow status updates

#### 6.4.4 Event Publishing
- Risk Assessment Completed: Published when a risk assessment is completed
- Risk Category Changed: Published when a customer's risk category changes
- Enhanced Due Diligence Required: Published when EDD is required
- Enhanced Due Diligence Completed: Published when EDD is completed

### 6.5 Data Management

#### 6.5.1 Data Ownership
The Risk Assessment Service owns:
- Risk assessments and factors
- Risk rules and configurations
- Enhanced due diligence records
- Risk change events

#### 6.5.2 Storage Strategy
- Risk data stored in PostgreSQL database
- Risk rules and evaluation logic stored in JSON format
- Risk assessment history maintained with appropriate retention periods

#### 6.5.3 Caching Approach
- Active risk rules cached in Redis
- Recent risk assessments cached for quick access
- Cache invalidation on rule updates
## 7. Workflow Service

### 7.1 Component Overview

#### 7.1.1 Purpose and Responsibilities
The Workflow Service manages the application processing workflow, including task assignment, status tracking, and decision management. It orchestrates the flow of applications through the KYC process and ensures proper routing based on risk levels.

#### 7.1.2 Key Features
- Application status management
- Task assignment and routing
- Escalation management
- Decision recording
- Workflow configuration
- SLA monitoring

#### 7.1.3 Dependencies
- Customer Service: For customer information
- Document Management Service: For document status
- Verification Service: For verification status
- Risk Assessment Service: For risk assessment results
- Notification Service: For workflow notifications

### 7.2 Domain Model

#### 7.2.1 Application
```
Application {
    id: UUID (PK)
    customerId: UUID (FK to Customer)
    applicationType: String
    referenceNumber: String
    status: Enum [DRAFT, SUBMITTED, IN_REVIEW, PENDING_DOCUMENTS, PENDING_VERIFICATION, PENDING_APPROVAL, APPROVED, REJECTED, CANCELLED]
    currentStage: String
    submittedAt: DateTime
    lastUpdatedAt: DateTime
    completedAt: DateTime (optional)
    assignedTo: UUID (optional)
    priority: Enum [LOW, MEDIUM, HIGH]
    slaDeadline: DateTime (optional)
    riskCategory: Enum [LOW, MEDIUM, HIGH]
}
```

#### 7.2.2 WorkflowStage
```
WorkflowStage {
    id: UUID (PK)
    workflowId: UUID (FK to Workflow)
    name: String
    description: String
    displayOrder: Integer
    expectedDuration: Integer (in minutes)
    actions: JSON
    requiredRoles: Array<String>
    nextStages: JSON
    createdAt: DateTime
    updatedAt: DateTime
}
```

#### 7.2.3 Workflow
```
Workflow {
    id: UUID (PK)
    name: String
    description: String
    applicationType: String
    version: Integer
    status: Enum [DRAFT, ACTIVE, DEPRECATED]
    createdAt: DateTime
    updatedAt: DateTime
    createdBy: UUID
}
```

#### 7.2.4 Task
```
Task {
    id: UUID (PK)
    applicationId: UUID (FK to Application)
    workflowStageId: UUID (FK to WorkflowStage)
    taskType: String
    name: String
    description: String
    status: Enum [PENDING, IN_PROGRESS, COMPLETED, CANCELLED, BLOCKED]
    assignedTo: UUID (optional)
    assignedAt: DateTime (optional)
    dueDate: DateTime
    priority: Enum [LOW, MEDIUM, HIGH]
    completedAt: DateTime (optional)
    completedBy: UUID (optional)
    outcome: String (optional)
    blockedReason: String (optional)
}
```

#### 7.2.5 Decision
```
Decision {
    id: UUID (PK)
    applicationId: UUID (FK to Application)
    decisionType: String
    decision: Enum [APPROVED, REJECTED, REFERRED]
    decisionBy: UUID
    decisionAt: DateTime
    reason: String
    comments: String (optional)
    overrideReason: String (optional)
    systemRecommendation: Enum [APPROVED, REJECTED, NONE]
}
```

#### 7.2.6 Escalation
```
Escalation {
    id: UUID (PK)
    applicationId: UUID (FK to Application)
    taskId: UUID (FK to Task) (optional)
    escalationType: Enum [SLA_BREACH, RISK_LEVEL, MANUAL]
    reason: String
    status: Enum [ACTIVE, RESOLVED, CANCELLED]
    escalatedTo: UUID
    escalatedAt: DateTime
    resolvedAt: DateTime (optional)
    resolvedBy: UUID (optional)
    resolutionNotes: String (optional)
}
```

#### 7.2.7 ApplicationHistory
```
ApplicationHistory {
    id: UUID (PK)
    applicationId: UUID (FK to Application)
    eventType: String
    previousStatus: String (optional)
    newStatus: String (optional)
    previousStage: String (optional)
    newStage: String (optional)
    previousAssignee: UUID (optional)
    newAssignee: UUID (optional)
    performedBy: UUID
    performedAt: DateTime
    comments: String (optional)
}
```

#### 7.2.8 Business Rules
- Applications must follow the defined workflow stages
- Task assignments must consider staff roles and workload
- SLA breaches must trigger escalations
- High-risk applications must be assigned to senior staff
- All application status changes must be recorded in history
- Decisions must include justification, especially for rejections or overrides
- Escalations must be resolved before application completion

### 7.3 API Specification

#### 7.3.1 Application Management

##### Create Application
- **Endpoint**: `POST /api/workflows/applications`
- **Description**: Creates a new application
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "customerId": "uuid",
    "applicationType": "string",
    "referenceNumber": "string",
    "priority": "LOW|MEDIUM|HIGH"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationType": "string",
    "referenceNumber": "string",
    "status": "DRAFT",
    "submittedAt": "datetime",
    "priority": "string"
  }
  ```

##### Submit Application
- **Endpoint**: `POST /api/workflows/applications/{applicationId}/submit`
- **Description**: Submits an application for processing
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "SUBMITTED",
    "currentStage": "string",
    "submittedAt": "datetime"
  }
  ```

##### Get Application
- **Endpoint**: `GET /api/workflows/applications/{applicationId}`
- **Description**: Retrieves application details
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "applicationType": "string",
    "referenceNumber": "string",
    "status": "string",
    "currentStage": "string",
    "submittedAt": "datetime",
    "lastUpdatedAt": "datetime",
    "completedAt": "datetime",
    "assignedTo": "uuid",
    "assigneeName": "string",
    "priority": "string",
    "slaDeadline": "datetime",
    "riskCategory": "string",
    "tasks": [
      {
        "id": "uuid",
        "taskType": "string",
        "name": "string",
        "status": "string",
        "dueDate": "datetime"
      }
    ],
    "pendingDocuments": [
      {
        "documentType": "string",
        "documentSubType": "string",
        "status": "string"
      }
    ],
    "decisions": [
      {
        "decisionType": "string",
        "decision": "string",
        "decisionBy": "string",
        "decisionAt": "datetime"
      }
    ]
  }
  ```

##### Search Applications
- **Endpoint**: `GET /api/workflows/applications`
- **Description**: Searches for applications based on criteria
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - customerId: Customer ID
  - status: Application status
  - stage: Current workflow stage
  - assignedTo: Assigned user ID
  - priority: Application priority
  - riskCategory: Risk category
  - fromDate: Submission date range start
  - toDate: Submission date range end
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "applications": [
      {
        "id": "uuid",
        "customerId": "uuid",
        "referenceNumber": "string",
        "status": "string",
        "currentStage": "string",
        "submittedAt": "datetime",
        "assignedTo": "string",
        "priority": "string",
        "riskCategory": "string"
      }
    ]
  }
  ```

##### Update Application Status
- **Endpoint**: `PATCH /api/workflows/applications/{applicationId}/status`
- **Description**: Updates the status of an application
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "status": "string",
    "comments": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "string",
    "lastUpdatedAt": "datetime"
  }
  ```

#### 7.3.2 Task Management

##### Create Task
- **Endpoint**: `POST /api/workflows/applications/{applicationId}/tasks`
- **Description**: Creates a new task for an application
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "workflowStageId": "uuid",
    "taskType": "string",
    "name": "string",
    "description": "string",
    "assignedTo": "uuid",
    "dueDate": "datetime",
    "priority": "LOW|MEDIUM|HIGH"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "applicationId": "uuid",
    "taskType": "string",
    "name": "string",
    "status": "PENDING",
    "assignedTo": "uuid",
    "dueDate": "datetime",
    "priority": "string"
  }
  ```

##### Get Task
- **Endpoint**: `GET /api/workflows/tasks/{taskId}`
- **Description**: Retrieves task details
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "applicationId": "uuid",
    "workflowStageId": "uuid",
    "taskType": "string",
    "name": "string",
    "description": "string",
    "status": "string",
    "assignedTo": "uuid",
    "assigneeName": "string",
    "assignedAt": "datetime",
    "dueDate": "datetime",
    "priority": "string",
    "completedAt": "datetime",
    "completedBy": "uuid",
    "outcome": "string",
    "blockedReason": "string"
  }
  ```

##### Update Task Status
- **Endpoint**: `PATCH /api/workflows/tasks/{taskId}/status`
- **Description**: Updates the status of a task
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "status": "IN_PROGRESS|COMPLETED|CANCELLED|BLOCKED",
    "outcome": "string",
    "blockedReason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "string",
    "completedAt": "datetime",
    "completedBy": "uuid"
  }
  ```

##### Reassign Task
- **Endpoint**: `PATCH /api/workflows/tasks/{taskId}/assign`
- **Description**: Reassigns a task to another user
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "assignedTo": "uuid",
    "reason": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "assignedTo": "uuid",
    "assignedAt": "datetime"
  }
  ```

#### 7.3.3 Decision Management

##### Record Decision
- **Endpoint**: `POST /api/workflows/applications/{applicationId}/decisions`
- **Description**: Records a decision for an application
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "decisionType": "string",
    "decision": "APPROVED|REJECTED|REFERRED",
    "reason": "string",
    "comments": "string",
    "overrideReason": "string",
    "systemRecommendation": "APPROVED|REJECTED|NONE"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "applicationId": "uuid",
    "decisionType": "string",
    "decision": "string",
    "decisionBy": "uuid",
    "decisionAt": "datetime"
  }
  ```

##### Get Decision History
- **Endpoint**: `GET /api/workflows/applications/{applicationId}/decisions`
- **Description**: Retrieves decision history for an application
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "applicationId": "uuid",
    "decisions": [
      {
        "id": "uuid",
        "decisionType": "string",
        "decision": "string",
        "decisionBy": "uuid",
        "decisionByName": "string",
        "decisionAt": "datetime",
        "reason": "string",
        "comments": "string",
        "overrideReason": "string",
        "systemRecommendation": "string"
      }
    ]
  }
  ```

#### 7.3.4 Escalation Management

##### Create Escalation
- **Endpoint**: `POST /api/workflows/applications/{applicationId}/escalations`
- **Description**: Creates an escalation for an application
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "taskId": "uuid",
    "escalationType": "SLA_BREACH|RISK_LEVEL|MANUAL",
    "reason": "string",
    "escalatedTo": "uuid"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "applicationId": "uuid",
    "taskId": "uuid",
    "escalationType": "string",
    "reason": "string",
    "status": "ACTIVE",
    "escalatedTo": "uuid",
    "escalatedAt": "datetime"
  }
  ```

##### Resolve Escalation
- **Endpoint**: `PATCH /api/workflows/escalations/{escalationId}/resolve`
- **Description**: Resolves an escalation
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "resolutionNotes": "string"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "RESOLVED",
    "resolvedAt": "datetime",
    "resolvedBy": "uuid",
    "resolutionNotes": "string"
  }
  ```

#### 7.3.5 Workflow Configuration

##### Get Workflows
- **Endpoint**: `GET /api/workflows/definitions`
- **Description**: Retrieves workflow definitions
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - applicationType: Application type
  - status: Workflow status
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "workflows": [
      {
        "id": "uuid",
        "name": "string",
        "description": "string",
        "applicationType": "string",
        "version": "integer",
        "status": "string",
        "createdAt": "datetime"
      }
    ]
  }
  ```

##### Create Workflow
- **Endpoint**: `POST /api/workflows/definitions`
- **Description**: Creates a new workflow definition
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "name": "string",
    "description": "string",
    "applicationType": "string",
    "stages": [
      {
        "name": "string",
        "description": "string",
        "displayOrder": "integer",
        "expectedDuration": "integer",
        "actions": [
          {
            "name": "string",
            "type": "string",
            "parameters": {}
          }
        ],
        "requiredRoles": ["string"],
        "nextStages": [
          {
            "stageId": "string",
            "conditions": {}
          }
        ]
      }
    ]
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "applicationType": "string",
    "version": "integer",
    "status": "DRAFT",
    "createdAt": "datetime",
    "createdBy": "uuid"
  }
  ```

##### Activate Workflow
- **Endpoint**: `PATCH /api/workflows/definitions/{workflowId}/activate`
- **Description**: Activates a workflow definition
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "ACTIVE",
    "updatedAt": "datetime"
  }
  ```

### 7.4 Component Interactions

#### 7.4.1 Interaction with Customer Service
- The Workflow Service retrieves customer information for application processing
- It updates customer status based on application outcomes

#### 7.4.2 Interaction with Document Management Service
- The Workflow Service checks document status for workflow progression
- It triggers document requests when needed

#### 7.4.3 Interaction with Verification Service
- The Workflow Service initiates verification requests
- It receives verification results for decision-making

#### 7.4.4 Interaction with Risk Assessment Service
- The Workflow Service receives risk assessment results
- It routes applications based on risk categories
- It triggers enhanced due diligence workflows for high-risk cases

#### 7.4.5 Interaction with Notification Service
- The Workflow Service triggers notifications for:
  - Application status changes
  - Task assignments
  - Escalations
  - Decisions

#### 7.4.6 Event Publishing
- Application Created: Published when a new application is created
- Application Status Changed: Published when application status changes
- Task Created: Published when a new task is created
- Task Completed: Published when a task is completed
- Decision Recorded: Published when a decision is recorded
- Escalation Created: Published when an escalation is created

### 7.5 Data Management

#### 7.5.1 Data Ownership
The Workflow Service owns:
- Applications and their status
- Tasks and assignments
- Workflow definitions and stages
- Decisions
- Escalations
- Application history

#### 7.5.2 Storage Strategy
- Workflow data stored in PostgreSQL database
- Workflow definitions stored in JSON format
- Application history maintained with appropriate retention periods

#### 7.5.3 Caching Approach
- Active workflow definitions cached in Redis
- Current application status cached for quick access
- Task assignments cached for performance
- Cache invalidation on workflow updates
## 8. Notification Service

### 8.1 Component Overview

#### 8.1.1 Purpose and Responsibilities
The Notification Service manages all communications with customers and staff. It handles email and SMS notifications, notification preferences, templates, and delivery tracking.

#### 8.1.2 Key Features
- Email and SMS notifications
- Notification templates
- Notification preferences
- Notification history
- Delivery tracking
- Scheduled notifications

#### 8.1.3 Dependencies
- Customer Service: For customer contact information
- Workflow Service: For application status events
- Document Management Service: For document status events
- External email service provider
- External SMS service provider

### 8.2 Domain Model

#### 8.2.1 Notification
```
Notification {
    id: UUID (PK)
    recipientId: UUID
    recipientType: Enum [CUSTOMER, STAFF]
    notificationType: String
    channel: Enum [EMAIL, SMS, PUSH, IN_APP]
    subject: String
    content: String
    templateId: UUID (FK to NotificationTemplate)
    status: Enum [PENDING, SENT, DELIVERED, FAILED]
    priority: Enum [LOW, MEDIUM, HIGH]
    createdAt: DateTime
    scheduledAt: DateTime (optional)
    sentAt: DateTime (optional)
    deliveredAt: DateTime (optional)
    failureReason: String (optional)
    retryCount: Integer
    metadata: JSON
}
```

#### 8.2.2 NotificationTemplate
```
NotificationTemplate {
    id: UUID (PK)
    name: String
    description: String
    notificationType: String
    channel: Enum [EMAIL, SMS, PUSH, IN_APP]
    subject: String
    contentTemplate: String
    variables: Array<String>
    status: Enum [DRAFT, ACTIVE, DEPRECATED]
    createdAt: DateTime
    updatedAt: DateTime
    createdBy: UUID
}
```

#### 8.2.3 NotificationPreference
```
NotificationPreference {
    id: UUID (PK)
    userId: UUID
    notificationType: String
    channel: Enum [EMAIL, SMS, PUSH, IN_APP]
    enabled: Boolean
    updatedAt: DateTime
}
```

#### 8.2.4 NotificationDelivery
```
NotificationDelivery {
    id: UUID (PK)
    notificationId: UUID (FK to Notification)
    channel: Enum [EMAIL, SMS, PUSH, IN_APP]
    recipient: String
    providerName: String
    providerMessageId: String
    status: Enum [SENT, DELIVERED, FAILED, BOUNCED, OPENED, CLICKED]
    statusDetails: String
    deliveryAttemptedAt: DateTime
    statusUpdatedAt: DateTime
    metadata: JSON
}
```

#### 8.2.5 ScheduledNotification
```
ScheduledNotification {
    id: UUID (PK)
    notificationType: String
    recipientId: UUID
    recipientType: Enum [CUSTOMER, STAFF]
    scheduledAt: DateTime
    recurrence: String (optional, cron expression)
    templateId: UUID (FK to NotificationTemplate)
    templateData: JSON
    status: Enum [SCHEDULED, PROCESSED, CANCELLED]
    createdAt: DateTime
    createdBy: UUID
    lastProcessedAt: DateTime (optional)
    nextScheduledAt: DateTime (optional)
}
```

#### 8.2.6 Business Rules
- Notifications must respect user preferences
- Failed notifications must be retried with exponential backoff
- Notification delivery status must be tracked
- Notification templates must be versioned
- Sensitive information must not be included in notifications
- Notifications must be sent according to priority
- Scheduled notifications must be processed on time

### 8.3 API Specification

#### 8.3.1 Notification Management

##### Send Notification
- **Endpoint**: `POST /api/notifications`
- **Description**: Sends a notification to a recipient
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "recipientId": "uuid",
    "recipientType": "CUSTOMER|STAFF",
    "notificationType": "string",
    "channel": "EMAIL|SMS|PUSH|IN_APP",
    "templateId": "uuid",
    "templateData": {
      "key1": "value1",
      "key2": "value2"
    },
    "priority": "LOW|MEDIUM|HIGH",
    "scheduledAt": "datetime"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "recipientId": "uuid",
    "notificationType": "string",
    "channel": "string",
    "status": "PENDING",
    "createdAt": "datetime",
    "scheduledAt": "datetime"
  }
  ```

##### Get Notification
- **Endpoint**: `GET /api/notifications/{notificationId}`
- **Description**: Retrieves notification details
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "recipientId": "uuid",
    "recipientType": "string",
    "notificationType": "string",
    "channel": "string",
    "subject": "string",
    "content": "string",
    "templateId": "uuid",
    "status": "string",
    "priority": "string",
    "createdAt": "datetime",
    "scheduledAt": "datetime",
    "sentAt": "datetime",
    "deliveredAt": "datetime",
    "failureReason": "string",
    "retryCount": "integer",
    "delivery": {
      "recipient": "string",
      "providerName": "string",
      "providerMessageId": "string",
      "status": "string",
      "statusUpdatedAt": "datetime"
    }
  }
  ```

##### Search Notifications
- **Endpoint**: `GET /api/notifications`
- **Description**: Searches for notifications based on criteria
- **Headers**:
  - Authorization: Bearer {token}
- **Query Parameters**:
  - recipientId: Recipient ID
  - notificationType: Notification type
  - channel: Notification channel
  - status: Notification status
  - fromDate: Creation date range start
  - toDate: Creation date range end
  - page: Page number
  - size: Page size
- **Response**:
  ```json
  {
    "totalItems": "integer",
    "totalPages": "integer",
    "currentPage": "integer",
    "notifications": [
      {
        "id": "uuid",
        "recipientId": "uuid",
        "notificationType": "string",
        "channel": "string",
        "subject": "string",
        "status": "string",
        "createdAt": "datetime",
        "sentAt": "datetime"
      }
    ]
  }
  ```

##### Cancel Notification
- **Endpoint**: `DELETE /api/notifications/{notificationId}`
- **Description**: Cancels a pending notification
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "CANCELLED",
    "updatedAt": "datetime"
  }
  ```

#### 8.3.2 Template Management

##### Create Template
- **Endpoint**: `POST /api/notifications/templates`
- **Description**: Creates a new notification template
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "name": "string",
    "description": "string",
    "notificationType": "string",
    "channel": "EMAIL|SMS|PUSH|IN_APP",
    "subject": "string",
    "contentTemplate": "string",
    "variables": ["string"]
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "notificationType": "string",
    "channel": "string",
    "subject": "string",
    "contentTemplate": "string",
    "variables": ["string"],
    "status": "DRAFT",
    "createdAt": "datetime",
    "createdBy": "uuid"
  }
  ```

##### Get Template
- **Endpoint**: `GET /api/notifications/templates/{templateId}`
- **Description**: Retrieves template details
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "notificationType": "string",
    "channel": "string",
    "subject": "string",
    "contentTemplate": "string",
    "variables": ["string"],
    "status": "string",
    "createdAt": "datetime",
    "updatedAt": "datetime",
    "createdBy": "uuid"
  }
  ```

##### Update Template
- **Endpoint**: `PUT /api/notifications/templates/{templateId}`
- **Description**: Updates a notification template
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "description": "string",
    "subject": "string",
    "contentTemplate": "string",
    "variables": ["string"],
    "status": "DRAFT|ACTIVE|DEPRECATED"
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "description": "string",
    "subject": "string",
    "contentTemplate": "string",
    "variables": ["string"],
    "status": "string",
    "updatedAt": "datetime"
  }
  ```

##### Preview Template
- **Endpoint**: `POST /api/notifications/templates/{templateId}/preview`
- **Description**: Previews a template with sample data
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "templateData": {
      "key1": "value1",
      "key2": "value2"
    }
  }
  ```
- **Response**:
  ```json
  {
    "subject": "string",
    "content": "string"
  }
  ```

#### 8.3.3 Notification Preferences

##### Get User Preferences
- **Endpoint**: `GET /api/notifications/preferences/{userId}`
- **Description**: Retrieves notification preferences for a user
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "userId": "uuid",
    "preferences": [
      {
        "notificationType": "string",
        "channel": "string",
        "enabled": true,
        "updatedAt": "datetime"
      }
    ]
  }
  ```

##### Update User Preference
- **Endpoint**: `PUT /api/notifications/preferences/{userId}`
- **Description**: Updates notification preferences for a user
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "preferences": [
      {
        "notificationType": "string",
        "channel": "string",
        "enabled": true
      }
    ]
  }
  ```
- **Response**:
  ```json
  {
    "userId": "uuid",
    "updatedPreferences": [
      {
        "notificationType": "string",
        "channel": "string",
        "enabled": true,
        "updatedAt": "datetime"
      }
    ]
  }
  ```

#### 8.3.4 Scheduled Notifications

##### Schedule Notification
- **Endpoint**: `POST /api/notifications/schedule`
- **Description**: Schedules a notification for future delivery
- **Headers**:
  - Authorization: Bearer {token}
- **Request Body**:
  ```json
  {
    "notificationType": "string",
    "recipientId": "uuid",
    "recipientType": "CUSTOMER|STAFF",
    "scheduledAt": "datetime",
    "recurrence": "string",
    "templateId": "uuid",
    "templateData": {
      "key1": "value1",
      "key2": "value2"
    }
  }
  ```
- **Response**:
  ```json
  {
    "id": "uuid",
    "notificationType": "string",
    "recipientId": "uuid",
    "scheduledAt": "datetime",
    "recurrence": "string",
    "status": "SCHEDULED",
    "nextScheduledAt": "datetime"
  }
  ```

##### Cancel Scheduled Notification
- **Endpoint**: `DELETE /api/notifications/schedule/{scheduleId}`
- **Description**: Cancels a scheduled notification
- **Headers**:
  - Authorization: Bearer {token}
- **Response**:
  ```json
  {
    "id": "uuid",
    "status": "CANCELLED",
    "updatedAt": "datetime"
  }
  ```

### 8.4 Component Interactions

#### 8.4.1 Interaction with Customer Service
- The Notification Service retrieves customer contact information
- It respects customer notification preferences

#### 8.4.2 Interaction with Workflow Service
- The Notification Service receives events from the Workflow Service
- It sends notifications based on application status changes

#### 8.4.3 Interaction with Document Management Service
- The Notification Service receives events from the Document Management Service
- It sends notifications based on document status changes

#### 8.4.4 Integration with External Services
- Email Service Provider: For sending email notifications
- SMS Service Provider: For sending SMS notifications
- Push Notification Service: For sending push notifications (if applicable)

#### 8.4.5 Event Consumption
- The Notification Service consumes events from various services:
  - Customer events (registration, profile updates)
  - Document events (upload, verification, expiry)
  - Application events (status changes, decisions)
  - Verification events (completion, failures)

### 8.5 Data Management

#### 8.5.1 Data Ownership
The Notification Service owns:
- Notifications and delivery status
- Notification templates
- Notification preferences
- Scheduled notifications

#### 8.5.2 Storage Strategy
- Notification data stored in PostgreSQL database
- Templates stored in PostgreSQL with versioning
- Notification content stored with appropriate retention periods

#### 8.5.3 Caching Approach
- Active templates cached in Redis
- User preferences cached for quick access
- Cache invalidation on template or preference updates
