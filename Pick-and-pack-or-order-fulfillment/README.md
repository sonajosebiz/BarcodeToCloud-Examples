## Pick and Pack or Order Fulfillment

This model uses the **Transaction (Ledger)** approach. This is critical for "Pick and Pack" because if a packer makes a mistake (e.g., scans the wrong item or too many items), you need a record of that specific scan to reverse it without messing up the entire order history.

### The Object Model

We will use standard Salesforce **Order** objects extended with custom fields to track packing progress, plus the **Inventory Transaction** object to handle the stock deduction.



#### 1. Standard Object: Order (The Header)
*Represents the customer's request and the "box" being packed.*

| Field Label | API Name | Data Type | Description/Usage |
| :--- | :--- | :--- | :--- |
| **Order Number** | `OrderNumber` | Auto Number | Standard ID. |
| **Status** | `Status` | Picklist | **Key Values:** *Draft, Activated, Packing in Progress, Ready to Ship, Shipped.* |
| **Account** | `AccountId` | Lookup | The Customer. |
| **Packing Progress** | `Packing_Status__c`| Formula (Text)| Visual indicator: `IF(Items_Packed__c >= Total_Items__c, "Complete", "In Progress")`. |

#### 2. Standard Object: Order Product (`OrderItem`)
*The checklist. The app compares scans against this list.*

| Field Label | API Name | Data Type | Description/Usage |
| :--- | :--- | :--- | :--- |
| **Product** | `Product2Id` | Lookup | The item to be packed. |
| **Quantity** | `Quantity` | Number | Total amount ordered by customer. |
| **Qty Packed** | `Qty_Packed__c` | Number (18,0) | **Target Field:** Starts at 0. Increments with every valid scan. |
| **Qty Remaining**| `Qty_Remaining__c` | Formula | `Quantity - Qty_Packed__c`. |
| **Line Status** | `Line_Status__c` | Picklist | Values: *Pending, Partial, Packed*. Updated by automation. |

#### 3. Custom Object: Inventory Transaction (`Inventory_Transaction__c`)
*The specific action of taking an item off the shelf and putting it in the box.*

| Field Label | API Name | Data Type | Description/Usage |
| :--- | :--- | :--- | :--- |
| **Type** | `Type__c` | Picklist | Value for this scenario: **"Pick"** (Outbound). |
| **Related Order** | `Order__c` | Lookup (`Order`) | Links the scan to the specific customer order. |
| **Related Line** | `Order_Line_Item__c`| Lookup (`OrderItem`)| Links to the specific line (Optional but recommended for speed). |
| **Product** | `Product__c` | Lookup (`Product2`) | The item scanned. |
| **Quantity** | `Quantity__c` | Number | **Negative Value:** e.g., `-1`. Indicates removal from stock. |
| **Status** | `Status__c` | Picklist | *Pending, Processed, Error*. |

---

### The Workflow Steps

Here is the step-by-step interaction between the Mobile App, the User, and Salesforce.

#### Phase 1: Setup & Validation
**1. User Action:** User opens the "Pick & Pack" module in the app and scans the **Order Number** (printed on a pick ticket).
**2. App Logic:**
   * Query `Order` where `OrderNumber` matches scan.
   * Query related `OrderItem` records where `Qty_Remaining__c > 0`.
**3. App Display:** Shows the Order Header and a list of items remaining to be packed (e.g., "3x Red Widgets, 1x Blue Widget").

#### Phase 2: The Packing Loop
**4. User Action:** User picks an item from the shelf, puts it in the box, and **scans the item barcode**.
**5. App Logic (Local Validation):**
   * Does the scanned barcode match a Product in the "Remaining" list?
   * **If NO:** Play "Error Beep" and display "Item not on this order!"
   * **If YES:** Proceed to Phase 3.

#### Phase 3: The Transaction (Fire & Forget)
**6. App Action:** The App creates a new record in `Inventory_Transaction__c`.
   * `Type`: "Pick"
   * `Order__c`: [Current Order ID]
   * `Product__c`: [Scanned Product ID]
   * `Quantity`: -1 (Negative because it's leaving inventory)
   * `Status`: "Pending"

#### Phase 4: Salesforce Automation (The Backend)
*A "After Create" Flow triggers on the `Inventory_Transaction__c` record.*

**7. Update Inventory (Stock Level):**
   * Find the master `Inventory__c` record for this Product.
   * Subtract 1 from `Qty_Hand__c`.

**8. Update Order Progress (The Checklist):**
   * Find the related `OrderItem` record.
   * Increment `Qty_Packed__c` by 1.
   * *Logic:* If `Qty_Packed__c` now equals `Quantity`, update `Line_Status__c` to **Packed**.

**9. Update Order Header:**
   * Check if all child `OrderItem` records are status "Packed".
   * If yes, update `Order.Status` to **Ready to Ship**.

#### Phase 5: Feedback
**10. App Display:** The app receives the update (via push or polling) and checks off the item on the screen. The progress bar moves to 100%.

---

### Why this specific model works for Picking

1.  **Partial Picking:** If an order requires 10 items but you only have 5 on the shelf, this model allows you to scan the 5 you have. The `OrderItem` will show `Qty_Packed__c = 5` and `Qty_Remaining__c = 5`. The Order Status remains "Packing in Progress," accurately reflecting reality.
2.  **Over-Pick Prevention:** The App Logic (Step 5) looks at `Qty_Remaining`. If the user tries to scan an 11th item for a 10-item order, the app creates a "Stop" block before even sending data to Salesforce.
3.  **Traceability:** If a customer complains they received a "Blue Widget" instead of a "Red Widget," you can query `Inventory_Transaction__c` filtered by that `Order__c`. You will see exactly what was scanned and at what time.