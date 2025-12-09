# ABL Business Entity Architecture Pattern

## Overview

The Business Entity pattern provides a standardized, maintainable approach to data access in OpenEdge ABL applications. It separates UI logic from database operations through a layered architecture that promotes reusability, testability, and consistency across the application.

## Architecture Layers

### 1. UI Layer (Windows/Forms)
- **Responsibility**: User interaction and presentation
- **Access**: Never directly accesses database tables
- **Communication**: Calls Business Entity methods with datasets

### 2. Business Entity Layer
- **Responsibility**: Data access, business rules, validation
- **Inheritance**: Extends `OpenEdge.BusinessLogic.BusinessEntity`
- **Management**: Instantiated through EntityFactory (singleton pattern)

### 3. Database Layer
- **Responsibility**: Persistent storage
- **Access**: Only through data-sources attached to business entities

## Key Components

### EntityFactory (Singleton Pattern)

**Purpose**: Centralized management of business entity lifecycle

**Pattern**:
```abl
CLASS business.EntityFactory:
    /* Singleton instance */
    VAR PRIVATE STATIC EntityFactory objInstance.
    
    /* Entity instances */
    VAR PRIVATE CustomerEntity objCustomerEntityInstance.
    
    /* Private constructor prevents direct instantiation */
    CONSTRUCTOR PRIVATE EntityFactory():
    END CONSTRUCTOR.
    
    /* Public factory method */
    METHOD PUBLIC STATIC EntityFactory GetInstance():
        IF objInstance = ? THEN
            objInstance = NEW EntityFactory().
        RETURN objInstance.
    END METHOD.
    
    /* Entity getters with lazy initialization */
    METHOD PUBLIC CustomerEntity GetCustomerEntity():
        IF objCustomerEntityInstance = ? THEN
            objCustomerEntityInstance = NEW CustomerEntity().
        RETURN objCustomerEntityInstance.
    END METHOD.
END CLASS.
```

**Benefits**:
- Single source of truth for entity instances
- Prevents memory waste from duplicate objects
- Enables centralized cleanup and testing

### Dataset Definition (.i Include Files)

**Purpose**: Defines temp-tables and datasets for data transfer

**Pattern**:
```abl
/* Define temp-table with BEFORE-TABLE for change tracking */
DEFINE TEMP-TABLE ttCustomer BEFORE-TABLE bttCustomer
    FIELD CustNum AS INTEGER INITIAL "0" LABEL "Cust Num"
    FIELD Name AS CHARACTER LABEL "Name"
    FIELD Address AS CHARACTER LABEL "Address"
    FIELD City AS CHARACTER LABEL "City"
    /* ... additional fields ... */
    INDEX CustNum IS PRIMARY UNIQUE CustNum ASCENDING.

/* Define dataset containing the temp-table */
DEFINE DATASET dsCustomer FOR ttCustomer.
```

**Key Points**:
- `BEFORE-TABLE` enables change tracking for updates
- Temp-table fields match database table structure
- Primary index mirrors database primary key
- Shared via include file for consistency

### Business Entity Class

**Purpose**: Encapsulates all data operations for a specific entity

**Pattern**:
```abl
CLASS business.CustomerEntity INHERITS BusinessEntity USE-WIDGET-POOL:
    
    /* Include dataset definition */
    {business/CustomerDataset.i}
    
    /* Define data sources - one per database table */
    DEFINE DATA-SOURCE srcCustomer FOR Customer.
    
    CONSTRUCTOR PUBLIC CustomerEntity():
        /* Pass dataset handle to parent class */
        SUPER(DATASET dsCustomer:HANDLE).

        /* Create array of data source handles */
        VAR HANDLE[1] hDataSourceArray = DATA-SOURCE srcCustomer:HANDLE.
        VAR CHARACTER[1] cSkipListArray = [""].
        
        /* Set parent class properties */
        THIS-OBJECT:ProDataSource = hDataSourceArray.
        THIS-OBJECT:SkipList = cSkipListArray.
    END CONSTRUCTOR.
    
    /* Business methods follow... */
    
END CLASS.
```

**Critical Requirements**:
- Must inherit from `OpenEdge.BusinessLogic.BusinessEntity`
- Dataset handle passed to SUPER() constructor
- Data-source handles assigned to ProDataSource property
- SkipList array matches data-source array length

## Standard CRUD Operations

### Read (Query) Operations

**Pattern**:
```abl
METHOD PUBLIC LOGICAL GetCustomerByNumber(INPUT ipiCustNum AS INTEGER,
                                          OUTPUT DATASET dsCustomer):
    VAR CHARACTER cFilter.
    VAR LOGICAL lFound = FALSE.
    
    /* Build WHERE clause for database query */
    cFilter = "WHERE Customer.CustNum = " + STRING(ipiCustNum).
    
    /* Parent class ReadData() executes query and fills temp-table */
    THIS-OBJECT:ReadData(cFilter).

    /* Check if data was found */
    lFound = CAN-FIND(FIRST ttCustomer).
    
    RETURN lFound.
END METHOD.
```

**Key Points**:
- Use `OUTPUT DATASET` parameter (NOT `BY-REFERENCE`)
- Build filter as string with WHERE clause
- Call parent's `ReadData()` method
- Verify results using temp-table
- Return success/failure indicator

### Create Operations

**Pattern**:
```abl
METHOD PUBLIC VOID CreateCustomer(INPUT-OUTPUT DATASET dsCustomer):
    /* Caller populates temp-table with new record */
    /* Parent class handles database insert */
    THIS-OBJECT:CreateData(DATASET dsCustomer BY-REFERENCE).
END METHOD.
```

### Update Operations

**Pattern**:
```abl
METHOD PUBLIC VOID UpdateCustomer(INPUT-OUTPUT DATASET dsCustomer):
    /* Parent class detects changes and updates database */
    THIS-OBJECT:UpdateData(DATASET dsCustomer BY-REFERENCE).
END METHOD.
```

### Delete Operations

**Pattern**:
```abl
METHOD PUBLIC VOID DeleteCustomer(INPUT-OUTPUT DATASET dsCustomer):
    /* Parent class processes deletion */
    THIS-OBJECT:DeleteData(DATASET dsCustomer BY-REFERENCE).
END METHOD.
```

## Validation Pattern

**Pattern**:
```abl
METHOD PUBLIC LOGICAL ValidateCustomer(INPUT-OUTPUT DATASET dsCustomer,
                                     OUTPUT errorMessage AS CHARACTER):
    VAR LOGICAL isValid = TRUE.
    
    /* Find the record to validate */
    FIND FIRST ttCustomer NO-ERROR.
    IF AVAILABLE ttCustomer THEN DO:
        /* Check business rules */
        IF ttCustomer.Name = "" THEN DO:
            isValid = FALSE.
            errorMessage = "Customer name cannot be empty".
        END.
    END.
    
    RETURN isValid.
END METHOD.
```

## UI Integration Pattern

```abl
/* Include business entity classes */
USING business.CustomerEntity FROM PROPATH.
USING business.EntityFactory FROM PROPATH.

/* Include dataset definition */
{business/CustomerDataset.i}

/* Button event handler */
ON CHOOSE OF GetCustomer IN FRAME DEFAULT-FRAME:
    VAR INTEGER iCustomerNumber = INTEGER(CustomerNumber:screen-value).
    VAR EntityFactory objFactory = EntityFactory:GetInstance().
    VAR CustomerEntity objCustomerEntity = objFactory:GetCustomerEntity().
    VAR LOGICAL lCustomerFound.
    
    /* Call entity to fetch data - use OUTPUT DATASET */
    lCustomerFound = objCustomerEntity:GetCustomerByNumber(
        iCustomerNumber, 
        OUTPUT DATASET dsCustomer
    ).
    
    /* Update UI based on results */
    IF lCustomerFound THEN DO:
        FIND FIRST ttCustomer.
        IF AVAILABLE ttCustomer THEN DO:
            CustomerName = ttCustomer.Name.
            DISPLAY CustomerName WITH FRAME {&frame-name}.
        END.
    END.
    ELSE 
        MESSAGE "Customer not found" VIEW-AS ALERT-BOX.
END.
```