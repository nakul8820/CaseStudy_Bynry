# CaseStudy_Bynry
casestudy submission for Bynry


# StockFlow — Inventory Management System
### Take-Home Case Study Submission

---

## Part 1: Code Review & Debugging

### Issues Identified

1. The double commit. ( Atomicity problem )
2. No data validation at the start of the API ( for existing SKU, warehouse_id )
3. API success confusion , ( returns message instead of 201 code for CREATED)

### Impact

**1. The double commit issue**

This particular API snippet commits twice in the database once for new item creation and another one to update inventory for that particular item in warehouse inventory . If there is an error in the second commit or if the database has a unique constraint and the user has entered the wrong product id , the database might create different inventory records for different products altogether. If the internet crashes right after new product creation , inventory data will be lost and never updated .

**2. Data validation**

Data validation at start will make this API more robust. Which will help us in validating data , and identify which fields are compulsory and optional. In validation logic we need to check if that particular item exists or not ( SKU check ) and the same validation must be added for warehouse_id . warehouse must exist before commiting data to database. and price needs to validation.

**3. API status code**

API returns a message that product is created with product_id, which returns default 200 OK. instead it must return 201 Created .

### Code Fix

```python
@app.route('/api/products', methods=[POST])
def create_product():
    data = request.json

    # Data validation
    required = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']

    if not all(k in data for k in required):
        return jsonify({"error" : "Missing require fields"}), 400

    if Product.query.filter_by(sku=data['sku']).first():
        return jsonify({"error" : "SKU already exists" }), 409

    if not isinstance(data['price'], (int,float)) or data['price'] <= 0:
        return jsonify({"error" : "Price must be non-negative"}), 400

    if not Warehouse.query.get(data['warehouse_id']):
        return jsonify({"error": "Invalid warehouse ID"}), 400

    try:
        # Create a new product
        product = Product(
            name=data['name'],
            sku=data['sku'],
            warehouse_id=data['warehouse_id']
        )
        db.session.add(product)

        db.session.flush()
        # Not yet finalized , intermediate state in db and we use auto generated product id from the product table

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)

        db.session.commit() # single commit

        return jsonify({"message": "Product created ", "product_id": product.id}), 201

    except Exception as e:
        db.session.rollback()
        return jsonify({"error":"Internal server error"}), 500
```

---

## Part 2: Database Design

### Schema

```sql
CREATE TABLE companies(
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE warehouses(
    id SERIAL PRIMARY KEY,
    company_id INT NOT NULL REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    location TEXT,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE products(
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(12, 2) NOT NULL CHECK(price >=0),
    is_bundle BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE warehouse_inventory(
    id SERIAL PRIMARY KEY,
    warehouse_id INT NOT NULL REFERENCES warehouses(id),
    product_id INT NOT NULL REFERENCES products(id),
    quantity INT DEFAULT 0,
    reorder_point INT DEFAULT 15, -- minimum stock before alert
    UNIQUE (warehouse_id, product_id)
);

CREATE TABLE inventory_logs(
    id SERIAL PRIMARY KEY,
    inventory_id INT REFERENCES warehouse_inventory(id),
    change_amount INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_bundles(
    parent_id INT NOT NULL REFERENCES products(id),
    child_id INT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL DEFAULT 1,
    PRIMARY KEY (parent_id, child_id)
);

CREATE TABLE suppliers(
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE supplier_products(
    supplier_id INT NOT NULL REFERENCES suppliers(id),
    product_id INT NOT NULL REFERENCES products(id),
    supplier_price DECIMAL(12, 2) CHECK (supplier_price >= 0),
    PRIMARY KEY (supplier_id, product_id)
);
```

### Questions for the Product Team

1. Can two companies have different prices for same products ?
2. Regarding inventory change ( how client want to see change in database )
3. Do we need other specification of products in main product table

---

## Part 3: API Implementation

### Assumptions

Reorder point, linear sales., 1 sale is good enough for restock 

### Implementation

```python
@app.route('/api/companies/{company_id}/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    # Set our recent window 15
    time_window = datetime.now() - timedelta(days=15)

    # low stock items
    low_stock_items = db.session.query(warehouse_inventory, products, warehouses,
    supplier).join(product).join(warehouse).join(supplier).filter(
    warehouse.company_id == company_id,
    Inventory.quantity <= Inventory.reorder_point).all()
    # i joined warehouse_inventory, products, warehouses and suppliers to extract needed information for desired response

    alerts = []

    # now extract data for our time window
    for inventory, product, warehouse, supplier in low_stock_items:
        recent_sales = inventorylog.query.filter(
            inventorylog.inventory_id == inv.id,
            inventorylog.reason == 'Sale',
            inventorylog.created_at >= time_window).all()

        if len(recent_sales) > 0:
            total_units_sold = sum(abs(log.change_amount) for log in recent_sales)
            sales_per_day = total_units_sold / sales
            # assumed sales are linear
            if sales_per_day > 0:
                days_left = int(inventory.quantity / sales_per_day)

                alerts.append({
                    "product_id": product.id,
                    "product_name": product.name,
                    "sku": product.sku,
                    "warehouse_id": warehouse.id,
                    "warehouse_name": warehouse.name,
                    "current_stock": inventory.quantity,
                    "threshold": inventory.reorder_point,
                    "days_until_stockout": days_left,
                    "supplier": {
                        "id": supplier.id,
                        "name": supplier.name,
                        "contact_email": supplier.contact_email
                    }
                })

    return jsonify({
        "alerts": alerts,
        "total_alerts": len(alerts)
    }), 200
```
