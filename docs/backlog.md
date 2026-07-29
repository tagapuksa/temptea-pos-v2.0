# Deliverable 1 – Individual Contribution

## TeamPTea POS

**Student:** Benjie Pahamutag
**Role:** Team Leader / Repo Lead

---

# 1. CRUD User Stories

## Record Type: Products

### Create Product

**User Story**
As a staff member, I want to add a new product so that customers can purchase newly available milk tea products.

**Acceptance Criteria**

* Product name is required.
* Price must be greater than zero.
* The new product appears in the product list after saving.

---

### View Product List

**User Story**
As a staff member, I want to view all available products so that I can manage the menu.

**Acceptance Criteria**

* All saved products are displayed.
* Products show their name, category, price, and stock.
* A message is displayed if there are no products.

---

### View Product Details

**User Story**
As a staff member, I want to view product details so that I can check complete product information.

**Acceptance Criteria**

* Displays complete product information.
* Shows current stock quantity.
* Displays the product status.

---

### Update Product

**User Story**
As a staff member, I want to edit product information so that product details remain accurate.

**Acceptance Criteria**

* Existing product information loads correctly.
* Invalid prices are rejected.
* Updated information appears immediately after saving.

---

### Delete Product

**User Story**
As a staff member, I want to delete discontinued products so that the product list remains organized.

**Acceptance Criteria**

* Confirmation dialog appears before deletion.
* Deleted products are removed from the list.
* Canceling the action keeps the product unchanged.

---

# 2. Application Screens

### Product List Screen

Displays all milk tea products with Add, Edit, Delete, and Search buttons.

### Product Detail Screen

Displays complete information about the selected product.

### Create Product Screen

Allows users to enter a new product.

### Edit Product Screen

Allows users to modify an existing product.

### Delete Confirmation Dialog

Prompts users to confirm before deleting a product.

### Empty State

Displays **"No products available."**

### Error State

Displays an error message if saving or loading fails.

---

# 3. Low-Fidelity Wireframes

## Product List

```
-----------------------------------------------------
 TeamPTea POS
-----------------------------------------------------

 Search: ______________________ [Search]

 + Add Product

-----------------------------------------------------
| Product | Category | Price | Stock | Actions |
-----------------------------------------------------
| Wintermelon | Classic | ₱39 | 25 | Edit Delete |
| Taro | Premium | ₱49 | 15 | Edit Delete |
-----------------------------------------------------

```

---

## Product Details

```
-----------------------------
Product Details
-----------------------------

Product Name:
Category:
Price:
Stock:
Description:

[ Edit ]
[ Back ]

```

---

## Add Product

```
----------------------------
Add Product
----------------------------

Product Name

Category

Price

Stock

Description

[ Save ]
[ Cancel ]

```

---

## Edit Product

```
----------------------------
Edit Product
----------------------------

Product Name

Category

Price

Stock

Description

[ Update ]
[ Cancel ]

```

---

## Delete Confirmation

```
Delete Product?

Are you sure you want to delete this product?

[ Yes ]
[ No ]

```

---

## Empty State

```
----------------------------

No products available.

[ Add Product ]

```

---

## Error State

```
----------------------------

Unable to load products.

Please try again.

[ Retry ]

```

---

# 4. AI Scope Review

### Prompt Used

> Review the CRUD features for a small Milk Tea POS System developed by five BSIT students within ten weeks. Identify missing CRUD operations, missing UI states, and suggest features that should be removed if the project becomes too large.

### AI Feedback Summary

The AI confirmed that the CRUD operations are complete for the Product module. It also recommended including empty states, error messages, and delete confirmation dialogs. Advanced features such as online ordering, loyalty rewards, analytics dashboards, and delivery tracking were identified as outside the intended project scope and should be excluded.

### Final Decision

The project scope will remain focused on essential POS functionality, including Product Management, Order Processing, Customer Management, and Sales Records to ensure successful completion within the semester.

---

**Prepared by:**

**Benjie Pahamutag**
Team Leader / Repo Lead
TeamPTea POS

