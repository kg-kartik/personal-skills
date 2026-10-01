---
name: monthly-expense-analyzer
description: Fetch bank statements from Gmail, parse and categorize transactions, and generate structured hierarchical tracking tables with opening balance, total credits, total debits, closing balance, grand totals, and category rollups. Handles sheet permissions by creating a new sheet if access fails.
---
# Monthly Expense Analyzer

Fetch monthly account statements from Gmail, analyze and categorize cash inflows and outflows, and log them into Google Sheets or generate structured tables.

## Statement Timing & Gmail Search Logic
1. **Next-Month Statement Delivery Rule**:
   - Monthly bank statements are generated and delivered in the **following month** (e.g., the statement covering **June** arrives in early **July**, the statement covering **July** arrives in early **August**, etc.).
   - When the user asks for a particular month's statement (Month $M$), search Gmail for statements sent in Month $M+1$ (e.g., for June 2026, search for statement emails received in July 2026).
2. **Missing PDF Fallback & Exact Search Filter Block**:
   - Never provide raw URL search links, as browser redirect mechanisms frequently fail or route to default pages.
   - If the statement PDF attachment cannot be accessed or loaded directly in the environment, ALWAYS provide the exact, copy-pasteable Gmail search filter query in a standalone code block so the user can easily paste it into the Gmail search bar.
   - Standard format:
     ```text
     from:<bank_name> filename:pdf after:YYYY/MM/DD before:YYYY/MM/DD
     ```
   - Dynamically substitute `<bank_name>` with the bank identifier/name specified by the user (or search based on the specific institution requested).

## Core Rules & Formatting Requirements

### 1. Sheet Ingestion & Fallback
- Always attempt to locate and write to the user's primary tracking sheet (e.g. `Monthly Expense Tracker`).
- **Permission / Access Fallback**: If an existing spreadsheet is inaccessible or returns `PERMISSION_DENIED`, do not fail or stall. Immediately create a brand new Google Spreadsheet in Google Drive titled `[Month Year] Expense Tracker`, format it properly, and present the file chip/link.

### 2. Mandatory 4 Balance & Flow Overview Metrics
At the very top of the table/sheet (or rows 1-4), always display these 4 numbers clearly:
- **Opening Balance**
- **Total Credits (Inflow)**
- **Total Debits (Outflow)**
- **Closing Balance**

### 3. Hierarchical Category Structure
- **Main Category Headers**: Clean bold header row with no numbers (e.g. `Shopping & Lifestyle`, `Personal Transfers`, `Stocks & Broking (Groww/TMPVL)`, `Mutual Funds & SIP`, `Groceries`, `Dining & Food Delivery`, `Utilities & Bills`, `Subscriptions`, `Refunds & Reversals`, `Others / Local Spends`, `Income / Salary`).
- **Sub-category Entities**: Listed underneath each category with:
  - Entity Name
  - Debit (₹)
  - Credit (₹)
  - Net (₹)
- **Category Total Row**: Bold summation row at the bottom of each section (e.g. `Total Shopping & Lifestyle`).

### 4. Grand Totals
- Conclude the main table with a bold **GRAND TOTAL** row containing the overall sum of debits, overall sum of credits, and net variance.

### 5. In-Chat Presentation
- Whenever providing data directly in the chat, always provide a ready-to-copy tabular code block formatted with tabs so it pastes directly into spreadsheet cells with proper row/column alignment.