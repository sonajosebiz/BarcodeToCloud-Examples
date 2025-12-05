# Receive goods in a warehouse (Advanced)

The "Transaction Object" model (often called a **Ledger** model) is the industry standard for robust inventory systems. Instead of the mobile app directly overwriting the "Quantity on Hand" field, the app simply **creates a new record** for every scan.

Think of it like a bank statement: You don't just erase your balance and write a new number; you create a "Deposit" or "Withdrawal" transaction, and the bank calculates the balance from those.

### Why use this model?
1.  **Concurrency:** Multiple people can scan items for the same Purchase Order simultaneously without "row lock" errors.
2.  **Audit Trail:** You know exactly *who* scanned *what* and *when*. If a count is wrong, you can see the specific scan that caused it.
3.  **Offline Capability:** The app can queue up any number of scans while offline and push them all at once when connectivity returns.
4.  **Flexibility:** You can create a "correction" transaction with negative qty.

---

### The Data Model

Here we build on top of the previous example [Receive-goods-in-warehouse](../Receive-goods-in-warehouse/README.md), we introduce a new Custom Object that sits between your App and your Master Data.



#### 1. New Custom Object: Inventory Transaction (`Inventory_Transaction__c`)
*The "Log" of every action. The mobile app **only creates** records here.*

| Field Label | API Name | Data Type | Description/Usage |
| :--- | :--- | :--- | :--- |
| **Transaction ID** | `Name` | Auto Number | E.g., TRX-000542 |
| **Type** | `Type__c` | Picklist | Values: *Receipt, Pick, Adjustment, Transfer, Return*. |
| **Product** | `Product__c` | Lookup (`Product2`) | The item scanned. |
| **Quantity** | `Quantity__c` | Number (18,0) | Positive for Receipts, Negative for Picks. |
| **Related PO** | `Purchase_Order__c` | Lookup (`Purchase_Order__c`) | (Optional) Links scan to the inbound PO. |
| **Related Order** | `Order__c` | Lookup (`Order`) | (Optional) Links scan to the outbound Customer Order. |
| **Location** | `Location__c` | Text / Lookup | Where the scan happened (e.g., "Receiving Dock"). |
| **Status** | `Status__c` | Picklist | Values: *Pending, Processed, Error*. Default is "Pending". |
| **Error Log** | `Error_Message__c` | Text | Stores validation errors (e.g., "PO Closed"). |

---

### The Workflow: "Fire and Forget"

#### Step A: The Mobile App Action
When the user scans an item in the **Goods Receipt** workflow:
1.  **App Logic:** The app doesn't calculate the new total. It just captures the raw data.
2.  **App Action:** It executes a `create()` operation on `Inventory_Transaction__c`.
    * `Type__c`: "Receipt"
    * `Product__c`: [Scanned ID]
    * `Purchase_Order__c`: [Current PO ID]
    * `Quantity__c`: 1 (or user input)
    * `Status__c`: "Pending"

#### Step B: Salesforce Automation (The "Processor")
You build a **Record-Triggered Flow** (or Apex Trigger) that listens for new `Inventory_Transaction__c` records.

**Trigger:** After Record is Created (Status = Pending)

**Logic Flow:**
1.  **Lock & Load:** The Flow finds the related `PO_Line_Item__c` and `Inventory__c` master records.
2.  **Validation:**
    * *Check:* Is the PO Open?
    * *Check:* Does `Qty_Received` + `Transaction.Quantity` exceed `Qty_Ordered`? (Optional)
3.  **Update Master Records:**
    * **PO Line Item:** Add `Transaction.Quantity` to `Qty_Received__c`.
    * **Inventory:** Add `Transaction.Quantity` to `Qty_Hand__c`.
4.  **Close Transaction:**
    * If successful, update `Inventory_Transaction__c.Status__c` to **Processed**.
    * If failed, update `Status__c` to **Error** and write the reason to `Error_Message__c`.

---

### Comparison: Direct Update vs. Transaction Model

| Feature | Direct Update Model (Simple) | Transaction Model (Robust) |
| :--- | :--- | :--- |
| **App Complexity** | Higher (App must read current qty, do math, write back). | **Lower** (App just dumps data). |
| **Conflict Risk** | High (Last save wins if two users scan same item). | **Zero** (Each scan is a separate row). |
| **Mistake Correction** | Hard (Must manually calculate how to fix it). | **Easy** (Create a "correction" transaction with negative qty). |
| **Storage Usage** | Low (Only master records). | **High** (One record per scan; requires data archiving strategy). |

### When to use which?
* **Use Direct Update** if you are a small shop with 1-2 warehouse users and low volume.
* **Use Transaction Model** if you have a team of people, high volume, or need strict audit compliance (e.g., medical devices, high-value electronics).