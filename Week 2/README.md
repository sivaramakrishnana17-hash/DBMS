# E-Commerce Order Management Database System

## Project Overview
This repository contains the architecture, structural design, and relational database layout for an enterprise-level **E-Commerce Order Management Database System**. The database is designed to handle high-volume transactions securely, ensure strict data integrity, and support analytical operations across customer profiles, multi-destination ordering, global cataloging, automated payments, and logistics tracking.

The current implementation achieves **Third Normal Form (3NF)** compliance, eliminating functional dependency anomalies and separating concerns to maximize performance under high concurrent loads.

---

## Repository Structure
```text
├── README.md                           # Main project documentation and setup guide
└── Week2/
    ├── Entity_Relationship_Analysis.pdf # Deep-dive structural analysis map & cascade laws
    └── Entity_Relationship_Analysis.html# HTML source template for the ER analysis map
```

---

## Architectural Highlights & Relational Schema
As mapped in the `Entity_Relationship_Analysis.pdf`, the system relies on structured relationship layers to eliminate typical data anomalies:

*   **Many-to-Many (M:N) Resolution**: Resolved via the `OrderItems` associative entity, capturing point-in-time metrics (`UnitPrice`, `Discount`) to insulate historical orders from subsequent catalog updates.
*   **Double-Vector Routing**: Handles decoupled geo-tracking by resolving two separate foreign keys from `Orders` (`ShippingAddressID` and `BillingAddressID`) back to a unified `Addresses` repository.
*   **One-to-One (1:1) Transact Isolation**: Enforces a strict singular invoice-to-payment lifecycle using unique indexes over `OrderID` in both the `Payments` and `ShippingDetails` domains.

### Logical Path Vectors
```text
[CUSTOMERS] (1) ────────────────────────────────────────────── (N) [ADDRESSES]
     │ (1)
     │
    (N) [ORDERS] (1) ───────────────────────────────────────── (1) [PAYMENTS] (Unique)
         │ (1)
         ├──────────────────────────────────────────────────── (1) [SHIPPING_DETAILS] (Unique)
         │ (1)
         └─── (N) [ORDER_ITEMS] (N) ────────────────────────── (1) [PRODUCTS]
                                                                     │
                                                                     │ (N)
                                                                     │ (1)
                                                               <strong>[CATEGORIES]</strong>
```

---

## Data Model & Foreign Key Constraints

The foundational entities are bound together by strict referential integrity properties:

| Parent Entity | Relation | Child Entity | Cardinality | ON DELETE | ON UPDATE |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Customers` | Places | `Orders` | 1 : N | `RESTRICT` | `CASCADE` |
| `Customers` | Registers | `Addresses` | 1 : N | `CASCADE` | `CASCADE` |
| `Categories` | Houses | `Products` | 1 : N | `RESTRICT` | `CASCADE` |
| `Orders` | Contains | `OrderItems` | 1 : N | `CASCADE` | `CASCADE` |
| `Products` | Appears In | `OrderItems` | 1 : N | `RESTRICT` | `CASCADE` |
| `Orders` | Settles Via | `Payments` | 1 : 1 | `RESTRICT` | `CASCADE` |

---

## Getting Started: SQL DDL Implementation

To initialize the database layout with all cascading properties matching the structural analysis map, execute the script below in your relational database engine (PostgreSQL / MySQL compatible):

```sql
-- 1. Create Categories Table
CREATE TABLE Categories (
    CategoryID INT AUTO_INCREMENT PRIMARY KEY,
    CategoryName VARCHAR(100) NOT NULL,
    Description TEXT
);

-- 2. Create Products Table
CREATE TABLE Products (
    ProductID INT AUTO_INCREMENT PRIMARY KEY,
    CategoryID INT NOT NULL,
    ProductName VARCHAR(150) NOT NULL,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    BasePrice DECIMAL(10, 2) NOT NULL,
    StockQuantity INT DEFAULT 0,
    CONSTRAINT FK_Products_Categories FOREIGN KEY (CategoryID)
        REFERENCES Categories(CategoryID) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 3. Create Customers Table
CREATE TABLE Customers (
    CustomerID INT AUTO_INCREMENT PRIMARY KEY,
    FirstName VARCHAR(50) NOT NULL,
    LastName VARCHAR(50) NOT NULL,
    Email VARCHAR(100) UNIQUE NOT NULL,
    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. Create Addresses Table
CREATE TABLE Addresses (
    AddressID INT AUTO_INCREMENT PRIMARY KEY,
    CustomerID INT NOT NULL,
    StreetAddress VARCHAR(255) NOT NULL,
    City VARCHAR(100) NOT NULL,
    State VARCHAR(100) NOT NULL,
    PostalCode VARCHAR(20) NOT NULL,
    Country VARCHAR(100) NOT NULL,
    CONSTRAINT FK_Addresses_Customers FOREIGN KEY (CustomerID)
        REFERENCES Customers(CustomerID) ON DELETE CASCADE ON UPDATE CASCADE
);

-- 5. Create Orders Table
CREATE TABLE Orders (
    OrderID INT AUTO_INCREMENT PRIMARY KEY,
    CustomerID INT NOT NULL,
    ShippingAddressID INT NOT NULL,
    BillingAddressID INT NOT NULL,
    OrderDate TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    TotalAmount DECIMAL(10, 2) NOT NULL,
    OrderStatus VARCHAR(30) DEFAULT 'Pending',
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerID)
        REFERENCES Customers(CustomerID) ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT FK_Orders_ShippingAddress FOREIGN KEY (ShippingAddressID)
        REFERENCES Addresses(AddressID) ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT FK_Orders_BillingAddress FOREIGN KEY (BillingAddressID)
        REFERENCES Addresses(AddressID) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 6. Create OrderItems Table (Associative Entity)
CREATE TABLE OrderItems (
    OrderItemID INT AUTO_INCREMENT PRIMARY KEY,
    OrderID INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity INT NOT NULL CHECK (Quantity > 0),
    UnitPrice DECIMAL(10, 2) NOT NULL,
    Discount DECIMAL(5, 2) DEFAULT 0.00,
    CONSTRAINT FK_OrderItems_Orders FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID) ON DELETE CASCADE ON UPDATE CASCADE,
    CONSTRAINT FK_OrderItems_Products FOREIGN KEY (ProductID)
        REFERENCES Products(ProductID) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 7. Create Payments Table (1:1 Relation with Orders)
CREATE TABLE Payments (
    PaymentID INT AUTO_INCREMENT PRIMARY KEY,
    OrderID INT UNIQUE NOT NULL,
    PaymentDate TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    Amount DECIMAL(10, 2) NOT NULL,
    PaymentMethod VARCHAR(50) NOT NULL,
    PaymentStatus VARCHAR(30) NOT NULL,
    CONSTRAINT FK_Payments_Orders FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 8. Create ShippingDetails Table (1:1 Relation with Orders)
CREATE TABLE ShippingDetails (
    ShippingID INT AUTO_INCREMENT PRIMARY KEY,
    OrderID INT UNIQUE NOT NULL,
    TrackingNumber VARCHAR(100) UNIQUE,
    Carrier VARCHAR(50) NOT NULL,
    EstimatedDelivery DATE,
    ActualDelivery DATE,
    ShippingStatus VARCHAR(50) NOT NULL,
    CONSTRAINT FK_ShippingDetails_Orders FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID) ON DELETE RESTRICT ON UPDATE CASCADE
);
```

---

## Production Verification Checklist
Prior to moving this database design into staging, verify that the application layout adheres to the baseline rules:
1. **Index Coverage**: Confirm all foreign key vector components (`CustomerID`, `OrderID`, `ProductID`, `CategoryID`) are indexed explicitly to reduce sorting bottlenecks during complex execution joins.
2. **Point-In-Time Capture**: Ensure that your backend application logic populates `OrderItems.UnitPrice` directly from the active `Products.BasePrice` dynamically during checkout, rather than calculating it on-the-fly via views later on.
3. **Transaction Controls**: Enforce explicit transaction blocks (`BEGIN TRANSACTION` ... `COMMIT`) when recording entries across `Orders`, `OrderItems`, and `Payments` to keep records cleanly balanced across all layers.

---

## Review and Approvals
* **Prepared By**: Relational Database Architect (Data Infrastructure Division)
* **Reviewed By**: Engineering Review Committee (E-Commerce Architecture Team)
