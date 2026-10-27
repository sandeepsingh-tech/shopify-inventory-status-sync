# Automatically Sync Shopify Product Status Based on Inventory Levels

A complete, beginner-friendly guide to automatically moving out-of-stock products to **Draft** and reactivating restocked products to **Active** using Python and the Shopify GraphQL API.

---

## 1. Why Automate Product Status by Inventory?

Managing catalog visibility manually across hundreds or thousands of SKUs is time-consuming and error-prone. Automating this sync provides immediate business benefits:

* **Protects Conversion Rates & Customer Trust:** Shoppers landing on unavailable items often bounce; keeping out-of-stock items off your active storefront prevents dead clicks and friction.
* **Eliminates Manual Catalog Maintenance:** Your team no longer needs to review spreadsheets or bulk-edit products whenever stock levels drop to zero.
* **Instant Restock Visibility:** As soon as an item is replenished in your inventory, it is automatically returned to **Active** status without staff intervention.
* **Preserves SEO & Product Data:** Moving items to **Draft** hides them safely from storefront collections while keeping tags, images, variants, and historical data intact.

---

## 2. Setting Up Python on Your Computer

You only need to install Python once. Follow the instructions for your operating system:

### Windows

1. Visit [python.org/downloads](https://www.python.org/downloads/) and click **Download Python**.
2. Run the downloaded installer.
3. **Crucial:** On the very first screen of the installer, check the box that says **"Add python.exe to PATH"** before clicking **Install Now**.
4. Open **Command Prompt** (search for `cmd` in the Start menu) and test the installation:
```bash
python --version

```


*(It should display `Python 3.x.x`)*

### Mac (macOS)

1. Visit [python.org/downloads](https://www.python.org/downloads/) and download the macOS installer.
2. Run the package installer and follow the on-screen prompts.
3. Open **Terminal** (press `Cmd + Space`, type `Terminal`, and press Enter) and check:
```bash
python3 --version

```



### Install the Required `requests` Library

In your Command Prompt or Terminal, install the network library required to talk to Shopify:

* **Windows:**
```bash
pip install requests

```


* **Mac:**
```bash
pip3 install requests

```



---

## 3. Gathering Your Shopify API Credentials

To let the script communicate with your store, create a dedicated Custom App in Shopify:

1. **Log in to Shopify Admin** as the store owner or an administrator.
2. Go to **Settings** (gear icon, bottom-left corner) → **Apps and sales channels**.
3. Click **Develop apps** (top-right). If prompted to allow custom app development, click **Allow custom app development**.
4. Click **Create an app**:
* App name: `Inventory Status Automation`
* App developer: Select your email account.
* Click **Create app**.


5. Click **Configure Admin API scopes**:
* In the search box, find and check the following permissions:
* `read_products` and `write_products`
* `read_inventory`




6. Click **Save** in the top-right corner.
7. Click the **API credentials** tab, then click **Install app** and confirm.
8. Under **App credentials / Client credentials**, note down:
* **Client ID** (a 32-character string)
* **Client Secret** (click the reveal icon or copy button; treat this like a password)


9. Find your **Shop Name**: Look at your browser URL bar. The store name is the subdomain before `.myshopify.com` (for example, if your admin URL is `[https://admin.shopify.com/store/cool-apparel-co](https://admin.shopify.com/store/cool-apparel-co)` or `cool-apparel-co.myshopify.com`, your shop name is `cool-apparel-co`).

---

## 4. The Python Code

Create a new folder on your computer (e.g., `shopify-inventory-sync`). Inside that folder, create a file named `sync_inventory.py` using any text editor (Notepad, VS Code, or TextEdit).

Paste the following script:

```python
import sys
import time
import requests

# ==============================================================================
# 1. CONFIGURATION: UPDATE THE 4 LINES BELOW WITH YOUR STORE DETAILS
# ==============================================================================

# ---> [REPLACE THIS]: Enter your Shopify subdomain (without '.myshopify.com')
SHOP_NAME = "YOUR_SHOP_NAME_HERE"

# ---> [REPLACE THIS]: Enter your Shopify App Client ID
CLIENT_ID = "YOUR_CLIENT_ID_HERE"

# ---> [REPLACE THIS]: Enter your Shopify App Client Secret
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"

# ---> [LEAVE AS IS OR CHANGE]: Shopify API Version
API_VERSION = "2024-07"

# ---> [SAFETY SWITCH]:
# Set DRY_RUN = True  to safely test and see what will change WITHOUT altering store data
# Set DRY_RUN = False to actually apply changes and update products in Shopify
DRY_RUN = True

# ==============================================================================
# DO NOT MODIFY CODE BELOW THIS LINE UNLESS YOU ARE CUSTOMIZING FUNCTIONALITY
# ==============================================================================

GRAPHQL_URL = f"https://{SHOP_NAME}.myshopify.com/admin/api/{API_VERSION}/graphql.json"

# --- 2. AUTHENTICATION (OAuth Client Credentials) ---
print("[*] Authenticating with Shopify API...")
auth_url = f"https://{SHOP_NAME}.myshopify.com/admin/oauth/access_token"
auth_payload = {
    "client_id": CLIENT_ID,
    "client_secret": CLIENT_SECRET,
    "grant_type": "client_credentials"
}

resp = requests.post(auth_url, json=auth_payload, headers={"Content-Type": "application/json"})
data = resp.json()

if "access_token" not in data:
    sys.exit(f"[!] Authentication failed: {data}")

ACCESS_TOKEN = data["access_token"]
print("[+] Authenticated successfully.")

HEADERS = {
    "Content-Type": "application/json",
    "X-Shopify-Access-Token": ACCESS_TOKEN
}

# --- 3. GRAPHQL QUERIES & MUTATIONS ---
ACTIVE_ZERO_STOCK_QUERY = """
query getZeroStockActiveProducts($cursor: String) {
  products(first: 250, query: "status:active AND inventory_total:<=0", after: $cursor) {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      node {
        id
        title
        handle
        totalInventory
      }
    }
  }
}
"""

DRAFT_IN_STOCK_QUERY = """
query getInStockDraftProducts($cursor: String) {
  products(first: 250, query: "status:draft AND inventory_total:>0", after: $cursor) {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      node {
        id
        title
        handle
        totalInventory
      }
    }
  }
}
"""

UPDATE_PRODUCT_STATUS_MUTATION = """
mutation updateProductStatus($product: ProductUpdateInput!) {
  productUpdate(product: $product) {
    product {
      id
      title
      status
    }
    userErrors {
      field
      message
    }
  }
}
"""

# --- 4. FETCH PRODUCTS TO SYNC ---
# Phase 4A: Active products with zero/negative stock -> targets for DRAFT
print("\n[*] Scanning active products with 0 stock (targets for DRAFT)...")
to_draft = []
cursor = None
has_next = True

while has_next:
    variables = {"cursor": cursor}
    res = requests.post(
        GRAPHQL_URL,
        json={"query": ACTIVE_ZERO_STOCK_QUERY, "variables": variables},
        headers=HEADERS
    ).json()

    prod_data = res.get("data", {}).get("products", {})
    edges = prod_data.get("edges", [])

    for edge in edges:
        node = edge["node"]
        if node.get("totalInventory", 0) <= 0:
            to_draft.append(node)

    page_info = prod_data.get("pageInfo", {})
    has_next = page_info.get("hasNextPage", False)
    cursor = page_info.get("endCursor")

    print(f"    -> Identified {len(to_draft)} active products with 0 stock...")
    time.sleep(0.05)

print(f"[+] Total active products to move to DRAFT: {len(to_draft)}")

# Phase 4B: Draft products with stock replenished -> targets for ACTIVE
print("\n[*] Scanning draft products with stock > 0 (targets for ACTIVE)...")
to_activate = []
cursor = None
has_next = True

while has_next:
    variables = {"cursor": cursor}
    res = requests.post(
        GRAPHQL_URL,
        json={"query": DRAFT_IN_STOCK_QUERY, "variables": variables},
        headers=HEADERS
    ).json()

    prod_data = res.get("data", {}).get("products", {})
    edges = prod_data.get("edges", [])

    for edge in edges:
        node = edge["node"]
        if node.get("totalInventory", 0) > 0:
            to_activate.append(node)

    page_info = prod_data.get("pageInfo", {})
    has_next = page_info.get("hasNextPage", False)
    cursor = page_info.get("endCursor")

    print(f"    -> Identified {len(to_activate)} restocked draft products...")
    time.sleep(0.05)

print(f"[+] Total draft products to restore to ACTIVE: {len(to_activate)}")

if not to_draft and not to_activate:
    print("\n[+] Catalog is completely in sync! No status changes required.")
    sys.exit(0)

# --- 5. EXECUTE STATUS UPDATES ---
if DRY_RUN:
    print("\n" + "=" * 50)
    print("[i] SAFETY PREVIEW (DRY_RUN is set to True)")
    print("No changes were made to your Shopify store.")
    if to_draft:
        print(f"    -> Sample item that would move to DRAFT:  '{to_draft[0]['title']}' (Stock: {to_draft[0]['totalInventory']})")
    if to_activate:
        print(f"    -> Sample item that would move to ACTIVE: '{to_activate[0]['title']}' (Stock: {to_activate[0]['totalInventory']})")
    print("\nTo apply these updates for real, change DRY_RUN = False in the script and re-run.")
    print("=" * 50)
    sys.exit(0)

def update_product_statuses(products, target_status):
    success_count = 0
    fail_count = 0
    total = len(products)

    for i, prod in enumerate(products, 1):
        payload = {
            "product": {
                "id": prod["id"],
                "status": target_status
            }
        }

        m_res = requests.post(
            GRAPHQL_URL,
            json={"query": UPDATE_PRODUCT_STATUS_MUTATION, "variables": payload},
            headers=HEADERS
        ).json()

        errors = m_res.get("data", {}).get("productUpdate", {}).get("userErrors", [])

        if errors:
            print(f"[!] Error updating '{prod['title']}': {errors[0]['message']}")
            fail_count += 1
        else:
            success_count += 1
            if success_count % 50 == 0 or i == total:
                print(f"[+] Updated {success_count} / {total} products to {target_status}...")

        # Small pause to stay well within Shopify API rate limits
        time.sleep(0.06)

    return success_count, fail_count

# Execute moving zero-stock items to DRAFT
drafted_success, drafted_fails = 0, 0
if to_draft:
    print(f"\n[*] Moving {len(to_draft)} zero-stock products to DRAFT...")
    drafted_success, drafted_fails = update_product_statuses(to_draft, "DRAFT")

# Execute moving restocked items to ACTIVE
activated_success, activated_fails = 0, 0
if to_activate:
    print(f"\n[*] Restoring {len(to_activate)} restocked products to ACTIVE...")
    activated_success, activated_fails = update_product_statuses(to_activate, "ACTIVE")

# Final summary output
print("\n" + "=" * 50)
print("INVENTORY STATUS SYNC COMPLETE")
print(f"Moved to DRAFT  (Out of Stock) : {drafted_success} (Failed: {drafted_fails})")
print(f"Restored ACTIVE (Restocked)    : {activated_success} (Failed: {activated_fails})")
print("=" * 50)

```

---

## 5. How to Run the Script

1. **Open Command Prompt or Terminal:**
Open your command line interface and navigate to the directory where you saved `sync_inventory.py`.

```bash
cd path/to/shopify-inventory-sync

```


2. **Execute the Safe Preview Run:** DRY_RUN mode active.
Run the script with the default dry-run flag:

* **Windows:** `python sync_inventory.py`
* **Mac:** `python3 sync_inventory.py`

Inspect the terminal output. It will authenticate with Shopify, count how many products match the criteria, display a sample title, and exit cleanly without modifying your live store.


3. **Switch Off Safety Mode:** Enable live modifications.
Open `sync_inventory.py` in your text editor. Find line 24:

```python
DRY_RUN = True

```

Change it to:

```python
DRY_RUN = False

```

Save the file.


4. **Execute Live Synchronization:**
Re-run the command in your terminal:

* **Windows:** `python sync_inventory.py`
* **Mac:** `python3 sync_inventory.py`

Watch the live progress counter as items transition between **Draft** and **Active**.


---

## 6. Pre-Flight Checklist & Things to Check

Before scheduling or regularly running this workflow, confirm these details:

| Checkpoint | What to Look For | Why It Matters |
| --- | --- | --- |
| **Pre-orders & Backorders** | Check if any product has **"Continue selling when out of stock"** checked. | If a product allows overselling/backorders but has `inventory_total = 0`, this script will draft it. Exclude pre-order tags or adjust the GraphQL query if you sell backorders. |
| **Multi-Location Inventory** | Verify your locations report to `inventory_total`. | `inventory_total` calculates aggregate stock across all fulfillment locations combined. |
| **Digital / Untracked Products** | Ensure digital products or gift cards have inventory tracking configured appropriately. | Untracked products without quantities could register as 0 inventory and move to draft. |
| **API Rate Limits** | Keep the `time.sleep(0.06)` pause intact. | Shopify GraphQL mutations use a leaky-bucket cost algorithm. The built-in delay ensures smooth updates without hitting rate caps. |
