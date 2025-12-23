# Environment Metadata Configuration

**Purpose:** Defines customer-specific mappings for Flexfields and COA.
**Usage:** The AI Agent reads this file to understand *your* specific data model.

---

## 1. 📊 Chart of Accounts (COA) Map
*Map `GL_CODE_COMBINATIONS` segments to Business Names.*

| Segment Column | Business Name | Description | Value Set Name |
|----------------|---------------|-------------|----------------|
| `SEGMENT1` | **Company** | Legal Entity / Balancing Segment | `XX_COMPANY_VS` |
| `SEGMENT2` | **Cost Center** | Department / Cost Owner | `XX_COST_CENTER_VS` |
| `SEGMENT3` | **Account** | Natural Account (Assets, Exp) | `XX_ACCOUNT_VS` |
| `SEGMENT4` | **Intercompany** | Trading Partner | `XX_INTERCO_VS` |
| `SEGMENT5` | **Product** | Product Line or LOB | `XX_PRODUCT_VS` |
| `SEGMENT6` | **Future1** | Reserved for future usage | `XX_FUTURE1_VS` |

---

## 2. 🔑 Key Flexfields (KFF)
*Map other critical KFF structures.*

### Assets (Fixed Assets)
*   **Table:** `FA_CATEGORIES_B`
*   **Segment1:** Major Category (e.g., Hardware)
*   **Segment2:** Minor Category (e.g., Laptop)

### Items (Inventory)
*   **Table:** `MTL_SYSTEM_ITEMS_B`
*   **Segment1:** Item Family
*   **Segment2:** SKU Number

---

## 3. 📝 Descriptive Flexfields (DFF)
*Map `ATTRIBUTE` columns to Business Purposes.*

### Accounts Payable
| Table Name | Column | Business Name | Usage Notes |
|------------|--------|---------------|-------------|
| `AP_INVOICES_ALL` | `ATTRIBUTE1` | **Legacy Invoices ID** | Migration reference |
| `AP_INVOICES_ALL` | `ATTRIBUTE2` | **Approval Route** | 'Fast' or 'Standard' |
| `AP_INVOICE_LINES_ALL` | `ATTRIBUTE1`| **Tax Authority** | Geo Code |

### Projects
| Table Name | Column | Business Name | Usage Notes |
|------------|--------|---------------|-------------|
| `PJF_PROJECTS_ALL_B` | `ATTRIBUTE1` | **Capital Request ID** | Capex Ref |
| `PJF_PROJECTS_ALL_B` | `ATTRIBUTE10`| **Region** | Sales Region |

---

## 4. 🏢 Ledger & Org IDs
*Hardcoded IDs for specific environments (Dev/Test/Prod).*

| Environment | Ledger ID | Org ID (US) | Org ID (UK) |
|-------------|-----------|-------------|-------------|
| **DEV1** | `300000001` | `204` | `205` |
| **TEST** | `300000002` | `204` | `205` |
| **PROD** | `300000003` | `204` | `205` |
