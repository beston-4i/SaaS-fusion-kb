# Purchasing Master Instructions

**Module:** Purchasing (PO)
**Tag:** `#SCM #PO #Purchasing`
**Status:** Active

---

## 1. 🚨 Critical PO Constraints

1.  **Open PO Status:**
    *   **Rule:** `AND CLOSED_CODE NOT IN ('CLOSED', 'FINALLY CLOSED')`
    *   **Why:** For "Open PO" reports, you must exclude closed documents.

2.  **Line Locations (Shipments):**
    *   **Rule:** Always join `PO_LINE_LOCATIONS_ALL` for quantity and receiving details.
    *   **Why:** `PO_LINES_ALL` only holds the item/price. The "Balance" is on the shipment.

3.  **Distributions (Accounts):**
    *   **Rule:** Join `PO_DISTRIBUTIONS_ALL` for Cost Centers and Project info.

---

## 2. 🗺️ Schema Map

### Core PO Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PHA** | `PO_HEADERS_ALL` | PO Header (Supplier, Number, Currency, Rates) |
| **PLA** | `PO_LINES_ALL` | PO Line (Item, Price, Quantity, Description) |
| **PLLA** | `PO_LINE_LOCATIONS_ALL`| Shipment/Schedule (Qty Ordered, Qty Received, Need By Date) |
| **PDA** | `PO_DISTRIBUTIONS_ALL` | GL Account, Project coding, Charge Accounts, Tax |
| **PLTB** | `PO_LINE_TYPES_V` | Line Type (Item, Amount based) |
| **PDSH** | `PO_DOC_STYLE_HEADERS` | Document style |
| **PDSLB** | `PO_DOC_STYLE_LINES_B` | Document style lines base |
| **PDSLT** | `PO_DOC_STYLE_LINES_TL` | Document style lines translated |

### Approval & Action History
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PAH** | `PO_ACTION_HISTORY` | PO approval workflow and action history |

### Supplier Tables  
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PSV** | `POZ_SUPPLIERS_V` | Supplier master (Vendor Name, Site, Party) |
| **PSSV** | `POZ_SUPPLIER_SITES_V` | Supplier site details |
| **PSSAM** | `POZ_SUPPLIER_SITES_ALL_M` | Supplier site additional details |
| **PSAV** | `POZ_SUPPLIER_ADDRESS_V` | Supplier address information |
| **PASCV** | `POZ_ALL_SUPPLIER_CONTACTS_V` | Supplier contact information |
| **PBCV** | `POZ_BUSINESS_CLASSIFICATIONS_V` | Business classifications |
| **ZPTP** | `ZX_PARTY_TAX_PROFILE` | Party tax profile |
| **ZR** | `ZX_REGISTRATIONS` | Tax registrations |

### Requisition Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PRHA** | `POR_REQUISITION_HEADERS_ALL` | Requisition Header |
| **PRLA** | `POR_REQUISITION_LINES_ALL` | Requisition Lines |
| **PRDA** | `POR_REQ_DISTRIBUTIONS_ALL` | Requisition Distributions |

### Receiving Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **RSH** | `RCV_SHIPMENT_HEADERS` | Receipt header |
| **RSL** | `RCV_SHIPMENT_LINES` | Receipt lines |
| **RT** | `RCV_TRANSACTIONS` | Receipt transactions |

### Negotiation Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PAHA** | `PON_AUCTION_HEADERS_ALL` | Negotiation/Tender header |
| **PBH** | `PON_BID_HEADERS` | Bid headers from suppliers |
| **PBR** | `PON_BACKING_REQUISITIONS` | Requisitions backing negotiations |
| **PAIPA** | `PON_AUCTION_ITEM_PRICES_ALL` | Auction item pricing |
| **PBP** | `PON_BIDDING_PARTIES` | Suppliers invited to bid |
| **PNST** | `PON_NEGOTIATION_STYLES_TL` | Negotiation style (Open, Limited tender) |
| **PBR** | `PON_BID_REQUIREMENTS` | Bid technical/commercial requirements |
| **PRS** | `PON_REQUIREMENT_SECTIONS` | Requirement sections (Technical, Commercial) |
| **PNAA** | `PON_NEG_AUDIT_ACTIVITIES` | Negotiation audit trail |
| **PBPN** | `PON_BID_PO_NUMBERS` | Link between bid and PO |

### Contract Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **OKHAB** | `OKC_K_HEADERS_ALL_B` | Contract header |

### Work Confirmation Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PWH** | `PO_WC_HEADERS` | Work confirmation header |
| **PWL** | `PO_WC_LINES` | Work confirmation lines |
| **APTC** | `POR_APPROVAL_TASK_COMMENTS` | Approval task comments |

### Supplier Qualification Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PI** | `POQ_INITIATIVES` | Supplier qualification initiatives |
| **PQ** | `POQ_QUALIFICATIONS` | Supplier qualifications |
| **PIS** | `POQ_INITIATIVE_SUPPLIERS` | Suppliers in qualification initiative |

### Item & UOM Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **ESIV** | `EGP_SYSTEM_ITEMS_VL` | Item master with description |
| **ESIB** | `EGP_SYSTEM_ITEMS_B` | Item master base |
| **ECV** | `EGP_CATEGORIES_VL` | Item categories |
| **IUOMV** | `INV_UNITS_OF_MEASURE_VL` | Unit of measure |

### Financial & GL Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **GCC** | `GL_CODE_COMBINATIONS` | Chart of accounts combination |
| **APT** | `AP_TERMS_TL` | Payment terms |
| **FABUV** | `FUN_ALL_BUSINESS_UNITS_V` | Business unit details |

### HCM Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PPNF** | `PER_PERSON_NAMES_F` | Person names (Buyer, Requester, Approver) |
| **PU** | `PER_USERS` | User details |
| **PAAF** | `PER_ALL_ASSIGNMENTS_F` | Employee assignments |
| **PD** | `PER_DEPARTMENTS` | Departments |
| **PAPF** | `PER_ALL_PEOPLE_F` | Person details |
| **HAPF** | `HR_ALL_POSITIONS_F_VL` | Position details |

### Lookup Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **FLVV** | `FND_LOOKUP_VALUES_VL` | Lookup values (Status, Type) |
| **FLVT** | `FND_LOOKUP_VALUES_TL` | Lookup values translated |

### Bank Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **IEBA** | `IBY_EXT_BANK_ACCOUNTS` | External bank accounts |
| **IAO** | `IBY_ACCOUNT_OWNERS` | Account owners |
| **IEPA** | `IBY_EXTERNAL_PAYEES_ALL` | External payees |
| **IPIUA** | `IBY_PMT_INSTR_USES_ALL` | Payment instrument uses |
| **CE** | `CE_BANKS_V` | Banks |
| **CBB** | `CE_BANK_BRANCHES_V` | Bank branches |

### Party Tables
| Alias | Table Name | Purpose |
|-------|------------|---------|
| **HP** | `HZ_PARTIES` | Trading partner parties |
| **HZ** | `HZ_LOCATIONS` | Location information |
| **PSAC** | `POZ_SUPPLIER_CONTACTS` | Supplier contacts |

---

## 3. 🔑 Primary Key Fields

- **PO_HEADERS_ALL:** `PO_HEADER_ID`
- **PO_LINES_ALL:** `PO_LINE_ID`
- **PO_LINE_LOCATIONS_ALL:** `LINE_LOCATION_ID`
- **PO_DISTRIBUTIONS_ALL:** `PO_DISTRIBUTION_ID`
- **POR_REQUISITION_HEADERS_ALL:** `REQUISITION_HEADER_ID`
- **PON_AUCTION_HEADERS_ALL:** `AUCTION_HEADER_ID`
- **POZ_SUPPLIERS_V:** `VENDOR_ID`
- **PO_WC_HEADERS:** `WORK_CONFIRMATION_ID`

---

## 4. 🚨 Critical Constraints & Business Rules

### PO Status Filtering
**Rule:** Filter by document status to get specific states
```sql
AND PHA.DOCUMENT_STATUS IN ('APPROVED', 'OPEN')
-- Common values: 'INCOMPLETE', 'APPROVED', 'OPEN', 'CLOSED', 'FINALLY CLOSED'
```

### Line Status Filtering
**Rule:** Exclude cancelled lines
```sql
AND PLA.LINE_STATUS <> 'CANCELED'
```

### Cancelled Invoices
**Rule:** Exclude cancelled dates where applicable
```sql
AND PHA.CANCEL_FLAG = 'N'
AND NVL(PLLA.CANCEL_FLAG, 'N') = 'N'
```

### Multi-Tenant Context
**Rule:** Always include BU or ORG context
```sql
AND PHA.PRC_BU_ID = :P_BU_ID
-- OR
AND (FABUV.BU_NAME IN (:P_BU_NAME) OR 'ALL' IN (:P_BU_NAME || 'ALL'))
```

### Approval History - Latest Approver
**Critical Pattern:** Get most recent approver using sequence number
```sql
AND PAH.SEQUENCE_NUM = (
    SELECT MAX(PAH1.SEQUENCE_NUM) 
    FROM PO_ACTION_HISTORY PAH1 
    WHERE PAH1.OBJECT_ID = PAH.OBJECT_ID 
    AND PAH1.ACTION_CODE = 'APPROVE'
)
```

### Exchange Rate Handling
**Rule:** Always default exchange rate to 1 if NULL
```sql
NVL(PHA.RATE, 1) AS EXCHANGE_RATE
```

### PO Amount Calculation
**Rule:** Calculate PO amount from lines
```sql
CASE 
    WHEN PLA.UNIT_PRICE IS NOT NULL AND PLA.QUANTITY IS NOT NULL 
    THEN (PLA.UNIT_PRICE * PLA.QUANTITY)
    ELSE PLA.LIST_PRICE 
END AS LINE_AMOUNT
```

### UOM Display Logic
**Critical Pattern:** Convert UOM codes to display values
```sql
CASE 
    WHEN PLLA.UOM_CODE = 'Ea' THEN 'Each'
    WHEN PLLA.UOM_CODE = 'MON' THEN 'Months'
    WHEN PLLA.UOM_CODE = 'YRS' THEN 'Years'
    ELSE PLLA.UOM_CODE
END AS UOM
```

### Prepayment & Retainage Handling
**Rule:** Track retainage on contracts
```sql
NVL(PLLA.RETAINAGE_RATE, 0) AS RETAINAGE_RATE
NVL(PLLA.RETAINAGE_WITHHELD_AMOUNT, 0) AS RETAINAGE_WITHHELD
NVL(PLLA.RETAINAGE_RELEASED_AMOUNT, 0) AS RETAINAGE_RELEASED
```

### Person Name Date-Effective Filtering
**Critical:** Always filter by effective dates for person names
```sql
AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) 
                       AND TRUNC(PPNF.EFFECTIVE_END_DATE)
AND PPNF.NAME_TYPE = 'GLOBAL'
```

### Tax Calculation
**Rule:** Calculate total with tax
```sql
NVL(PDA.NONRECOVERABLE_TAX, 0) + NVL(PDA.TAX_EXCLUSIVE_AMOUNT, 0) AS TOTAL_WITH_TAX
```

### Negotiation Status
**Rule:** Get negotiation status using package function
```sql
PON_AUCTION_PKG.GET_AUCTION_STATUS_DISPLAY(PAHA.AUCTION_HEADER_ID, 'Y') AS TENDER_STATUS
```

### Work Confirmation Status
**Rule:** Filter work confirmations by status
```sql
AND PWH.STATUS IN ('PENDING', 'APPROVED')
```

---
