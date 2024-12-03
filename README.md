## Phase 7: Advanced Database Programming and Auditing
### Problem Statement:
With the increasing demand for sustainable urban farming, managing multiple farm locations and associated activities has become challenging. Advanced database programming is needed to enhance data integrity, enforce business rules, streamline workflows, and optimize resource use. This database will efficiently handle crop cycles, staff and volunteer assignments, inventory tracking, and client interactions.

### the use of triggers, cursors, functions, packages, and auditing techniques to solve specific challenges or improve system functionality

`Triggers`: it will  Automate workflows and enforce business rules, e.g., updating inventory levels automatically after a harvest entry or ensuring that orders don’t exceed available inventory.

`Cursors`: it will Handle row-by-row processing for scenarios like calculating sustainability metrics for each farm or crop individually.

`Functions`: it will Encapsulate reusable calculations, e.g., total yield per farm, average sustainability score, or total volunteer hours.

`Packages`:it will  Group related database objects like functions and procedures to improve modularity, security, and organization.

`Auditing Techniques`: it will Monitor critical activities such as staff assignments or order modifications for transparency and accountability.

### Trigger Implementation 
#### BEFORE INSERT Trigger for Orders
this will Ensure that the quantity ordered does not exceed the available inventory before an order is placed.
```sql
CREATE OR REPLACE TRIGGER BeforeInsertOnOrders
BEFORE INSERT ON Orders
FOR EACH ROW
DECLARE
    v_AvailableQuantity NUMBER;
BEGIN
    -- Check inventory levels for the specified InventoryID
    SELECT Quantity
    INTO v_AvailableQuantity
    FROM Inventory
    WHERE InventoryID = :NEW.InventoryID;

    -- Raise an error if the quantity ordered exceeds available inventory
    IF :NEW.QuantityOrdered > v_AvailableQuantity THEN
        RAISE_APPLICATION_ERROR(-20001, 'Quantity ordered exceeds available inventory!');
    END IF;
END;
/

```
### Compound Trigger AFTER INSERT OR UPDATE Trigger for Harvests
it will Automatically update the inventory table after a harvest is recorded or modified.
```sql
CREATE OR REPLACE TRIGGER UpdateInventoryAfterHarvest
FOR INSERT OR UPDATE ON Harvests
COMPOUND TRIGGER

    -- Local variable to track changes
    TYPE HarvestChange IS RECORD (
        CropID INT,
        AdditionalYield NUMBER
    );
    TYPE HarvestChangeList IS TABLE OF HarvestChange;
    HarvestChangesList HarvestChangeList := HarvestChangeList();

    BEFORE EACH ROW IS
    BEGIN
        -- Track the additional yield
        HarvestChangesList.EXTEND;
        HarvestChangesList(HarvestChangesList.COUNT) := HarvestChange(:NEW.CropID, :NEW.Yield);
    END BEFORE EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        -- Update inventory for each change
        FOR i IN HarvestChangesList.FIRST..HarvestChangesList.LAST LOOP
            UPDATE Inventory
            SET Quantity = Quantity + HarvestChangesList(i).AdditionalYield
            WHERE CropID = HarvestChangesList(i).CropID;
        END LOOP;

        -- Clear the list
        HarvestChangesList.DELETE;
    END AFTER STATEMENT;

END UpdateInventoryAfterHarvest;
/

```
### Inventory Table (Before Trigger Execution)
![image](https://github.com/user-attachments/assets/2a511e8d-7c7b-4536-9e34-61104e8d824d)

### Insert Data into Harvests
```sql
INSERT INTO Harvests (HarvestID, CropID, FarmID, Date_, Yield, QualityRating)
VALUES (6, 2, 1, SYSDATE, 20, 4.9);
```
###  the Inventory table  Quantity  is updated from 780 to 800 for CropId 2
```sql
SELECT * FROM Inventory;
```
![image](https://github.com/user-attachments/assets/bb32939a-f8a3-4896-8133-12502c51c48f)

### Update the Yield to 25
```sql
UPDATE Harvests
SET Yield = 25
WHERE HarvestID = 2;
```
### The trigger will  update the Inventory Quantity from 800 to 850 for CropID 2

![image](https://github.com/user-attachments/assets/9d0b3025-4733-435f-94f6-dd566cfaef6d)

###Ensure that when an entry is added to the Inventory table, the corresponding CropID exists in the Crops table
```sql
CREATE OR REPLACE TRIGGER EnsureCropExists
BEFORE INSERT OR UPDATE ON Harvests
FOR EACH ROW
BEGIN
    -- Check if the CropID exists in the Crops table
    DECLARE
        v_CropExists NUMBER;
    BEGIN
        SELECT COUNT(*)
        INTO v_CropExists
        FROM Crops
        WHERE CropID = :NEW.CropID;

        IF v_CropExists = 0 THEN
            RAISE_APPLICATION_ERROR(-20001, 'CropID does not exist in the Crops table.');
        END IF;
    END;
END EnsureCropExists;
/

```
![image](https://github.com/user-attachments/assets/a0cae65c-6f23-4bce-b39b-431e99540e30)

### When an Order is placed, reduce the corresponding Inventory quantity for the selected InventoryID. If the quantity becomes less than a threshold, log a message to indicate a low stock.

```sql
CREATE OR REPLACE TRIGGER ManageInventoryOnOrder
FOR INSERT OR UPDATE ON Orders
COMPOUND TRIGGER
    -- Variables for statement-level operations
    TYPE OrderChange IS RECORD (
        InventoryID INT,
        QuantityOrdered NUMBER
    );
    OrderChangesList SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST();

    -- Row-level AFTER EACH ROW logic
    AFTER EACH ROW IS
    BEGIN
        -- Record the changes
        OrderChangesList.EXTEND;
        OrderChangesList(OrderChangesList.LAST) := TO_CHAR(:NEW.InventoryID) || ',' || TO_CHAR(:NEW.QuantityOrdered);
    END AFTER EACH ROW;

    -- Statement-level AFTER STATEMENT logic
    AFTER STATEMENT IS
        v_InventoryID INT;
        v_QuantityOrdered NUMBER;
        v_CurrentQuantity NUMBER; -- Variable to hold current inventory quantity
    BEGIN
        -- Process the changes
        FOR i IN 1..OrderChangesList.COUNT LOOP
            -- Parse InventoryID and QuantityOrdered
            v_InventoryID := TO_NUMBER(SUBSTR(OrderChangesList(i), 1, INSTR(OrderChangesList(i), ',') - 1));
            v_QuantityOrdered := TO_NUMBER(SUBSTR(OrderChangesList(i), INSTR(OrderChangesList(i), ',') + 1));

            -- Fetch current inventory quantity
            SELECT Quantity
            INTO v_CurrentQuantity
            FROM Inventory
            WHERE InventoryID = v_InventoryID;

            -- Check if the inventory will go negative
            IF v_CurrentQuantity - v_QuantityOrdered < 0 THEN
                RAISE_APPLICATION_ERROR(-20002, 'Insufficient inventory for InventoryID: ' || v_InventoryID);
            END IF;

            -- Update inventory
            UPDATE Inventory
            SET Quantity = Quantity - v_QuantityOrdered
            WHERE InventoryID = v_InventoryID;
        END LOOP;

        -- Clear the list
        OrderChangesList.DELETE;
    END AFTER STATEMENT;
END ManageInventoryOnOrder;
/

```
### Setup
```sql
INSERT INTO Inventory (InventoryID, CropID, Quantity, FreshnessStatus, StorageLocation)
VALUES (11, 2, 50, 'Fresh', 'Storage A');
```
### Place Order
```sql
INSERT INTO Orders (OrderID, ClientID, InventoryID, OrderDate, QuantityOrdered, DeliveryStatus)
VALUES (4, 1, 11, SYSDATE, 45, 'Pending');
```
### Quantity will be 50-45 = 5 the remaing quantity is 5
![image](https://github.com/user-attachments/assets/a23bc752-ea20-4e75-b66f-28be81d1552c)

### Place Order 
```sql
INSERT INTO Orders (OrderID, ClientID, InventoryID, OrderDate, QuantityOrdered, DeliveryStatus)
VALUES (5, 1, 11, SYSDATE, 6, 'Pending');
```
### Insufficient inventory for InventoryID
![image](https://github.com/user-attachments/assets/4bbf8bc8-2fa9-4b01-a22b-4553ee15a9b0)

### Row-by-Row Processing for Low Inventory
Generate a report of all Inventory items with a Quantity below a specified threshold, showing the InventoryID, CropID, and Quantity
```sql
CREATE OR REPLACE PROCEDURE GenerateLowInventoryReport(
    p_Threshold NUMBER
) IS
    CURSOR LowInventoryCursor IS
        SELECT InventoryID, CropID, Quantity
        FROM Inventory
        WHERE Quantity < p_Threshold;

    v_InventoryID Inventory.InventoryID%TYPE;
    v_CropID Inventory.CropID%TYPE;
    v_Quantity Inventory.Quantity%TYPE;
BEGIN
    DBMS_OUTPUT.PUT_LINE('Low Inventory Report:');
    DBMS_OUTPUT.PUT_LINE('--------------------------------');

    OPEN LowInventoryCursor;
    LOOP
        FETCH LowInventoryCursor INTO v_InventoryID, v_CropID, v_Quantity;
        EXIT WHEN LowInventoryCursor%NOTFOUND;

        DBMS_OUTPUT.PUT_LINE('InventoryID: ' || v_InventoryID || ', CropID: ' || v_CropID || ', Quantity: ' || v_Quantity);
    END LOOP;

    CLOSE LowInventoryCursor;
END;
/

```
### check Quantity under 100
```sql
BEGIN
    GenerateLowInventoryReport(100);
END;

```
![image](https://github.com/user-attachments/assets/f86c688d-509a-43b5-b959-f3d023939dd8)

###  Attributes and Functions
#### Calculate Total Orders for a Client
```sql
CREATE OR REPLACE FUNCTION GetTotalOrdersByClient(
    p_ClientID IN Clients.ClientID%TYPE
) RETURN NUMBER IS
    v_TotalOrders NUMBER;
BEGIN
    SELECT COUNT(*)
    INTO v_TotalOrders
    FROM Orders
    WHERE ClientID = p_ClientID;

    RETURN v_TotalOrders;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 0; -- If no orders are found, return 0
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        RETURN NULL;
END;
/

```
Using %TYPE ensures that the function is flexible and can handle future schema changes (e.g., changing data types in the database).

### geting total order for clientid 1
```sql
DECLARE
    v_TotalOrders NUMBER;
BEGIN
    v_TotalOrders := GetTotalOrdersByClient(1);
    DBMS_OUTPUT.PUT_LINE('Total Orders for Client 1: ' || v_TotalOrders);
END;
```
![image](https://github.com/user-attachments/assets/e34f1338-7ceb-4b09-bcde-c8c5cd851f96)

### Package Development
FarmManagement a package to handle operations related to farm management, such as adding farms, assigning staff, and updating inventory. The package will contain both procedures and functions that are related to managing farm activities.
```sql
CREATE OR REPLACE PACKAGE FarmManagement AS
    -- Procedure to add a new farm
    PROCEDURE AddFarm(
        p_FarmID INT, 
        p_Name VARCHAR2, 
        p_Address VARCHAR2, 
        p_TotalPlantingArea NUMBER, 
        p_AssignedStaff VARCHAR2
    );

    -- Procedure to assign staff to a farm
    PROCEDURE AssignStaffToFarm(
        p_PersonID INT, 
        p_AssignedFarm INT
    );

    -- Function to calculate the total crop yield for a given farm
    FUNCTION GetTotalCropYield(p_FarmID INT) RETURN NUMBER;

    -- Procedure to update inventory for a specific crop
    PROCEDURE UpdateInventory(
        p_CropID INT, 
        p_Quantity NUMBER, 
        p_FreshnessStatus VARCHAR2, 
        p_StorageLocation VARCHAR2
    );
END FarmManagement;
---------------------------------------------------------------------------------
---body of the package

CREATE OR REPLACE PACKAGE BODY FarmManagement AS
    -- Procedure to add a new farm
    PROCEDURE AddFarm(
        p_FarmID INT, 
        p_Name VARCHAR2, 
        p_Address VARCHAR2, 
        p_TotalPlantingArea NUMBER, 
        p_AssignedStaff VARCHAR2
    ) IS
    BEGIN
        INSERT INTO Farms (FarmID, Name, Address, TotalPlantingArea, AssignedStaff)
        VALUES (p_FarmID, p_Name, p_Address, p_TotalPlantingArea, p_AssignedStaff);
        COMMIT;
    END AddFarm;

    -- Procedure to assign staff to a farm
    PROCEDURE AssignStaffToFarm(
        p_PersonID INT, 
        p_AssignedFarm INT
    ) IS
    BEGIN
        UPDATE StaffAndVolunteers
        SET AssignedFarm = p_AssignedFarm
        WHERE PersonID = p_PersonID;
        COMMIT;
    END AssignStaffToFarm;

    -- Function to calculate the total crop yield for a given farm
    FUNCTION GetTotalCropYield(p_FarmID INT) RETURN NUMBER IS
        v_TotalYield NUMBER := 0;
    BEGIN
        SELECT NVL(SUM(AverageYield), 0)
        INTO v_TotalYield
        FROM Crops
        WHERE FarmID = p_FarmID;

        RETURN v_TotalYield;
    END GetTotalCropYield;

    -- Procedure to update inventory for a specific crop
    PROCEDURE UpdateInventory(
        p_CropID INT, 
        p_Quantity NUMBER, 
        p_FreshnessStatus VARCHAR2, 
        p_StorageLocation VARCHAR2
    ) IS
    BEGIN
        INSERT INTO Inventory (InventoryID, CropID, Quantity, FreshnessStatus, StorageLocation)
        VALUES (Inventory_SEQ.NEXTVAL, p_CropID, p_Quantity, p_FreshnessStatus, p_StorageLocation);
        COMMIT;
    END UpdateInventory;
END FarmManagement;
/


```
### out put
```sql
BEGIN
    FarmManagement.AddFarm(
        p_FarmID => 8, 
        p_Name => 'Green Valley Rwanda', 
        p_Address => '123 Farm kigali', 
        p_TotalPlantingArea => 1000, 
        p_AssignedStaff => 'John Gasore'
    );
END;

```
![image](https://github.com/user-attachments/assets/32a7cc25-58dd-4a1c-9aee-c9b3cc578dbb)

### GetTotalCropYield 
```sql
DECLARE
    v_Yield NUMBER;
BEGIN
    v_Yield := FarmManagement.GetTotalCropYield(1);
    DBMS_OUTPUT.PUT_LINE('Total Crop Yield for Farm 1: ' || v_Yield);
END;

```
![image](https://github.com/user-attachments/assets/40c160c1-bb70-4c91-8300-502c3afd35cc)

### triggers
Create the AuditLog Table
```sql
CREATE TABLE AuditLog (
    AuditID INT PRIMARY KEY,
    TableName VARCHAR2(50),
    Operation VARCHAR2(20),
    UserID VARCHAR2(50),
    Timestamp DATE DEFAULT SYSDATE,
    OldData VARCHAR2(4000),
    NewData VARCHAR2(4000)
);

```
### TRIGGER LogOrderChanges

```sql
CREATE SEQUENCE AuditLog_SEQ START WITH 1 INCREMENT BY 1;

CREATE OR REPLACE TRIGGER LogOrderChanges
AFTER INSERT OR UPDATE OR DELETE ON Orders
FOR EACH ROW
BEGIN
    -- Handle INSERT operations
    IF INSERTING THEN
        INSERT INTO AuditLog (AuditID, TableName, Operation, UserID, NewData)
        VALUES (
            AuditLog_SEQ.NEXTVAL, 
            'Orders', 
            'INSERT', 
            USER, 
            :NEW.OrderID || ' - ' || :NEW.QuantityOrdered
        );
    -- Handle UPDATE operations
    ELSIF UPDATING THEN
        INSERT INTO AuditLog (AuditID, TableName, Operation, UserID, OldData, NewData)
        VALUES (
            AuditLog_SEQ.NEXTVAL, 
            'Orders', 
            'UPDATE', 
            USER, 
            :OLD.OrderID || ' - ' || :OLD.QuantityOrdered, 
            :NEW.OrderID || ' - ' || :NEW.QuantityOrdered
        );
    -- Handle DELETE operations
    ELSIF DELETING THEN
        INSERT INTO AuditLog (AuditID, TableName, Operation, UserID, OldData)
        VALUES (
            AuditLog_SEQ.NEXTVAL, 
            'Orders', 
            'DELETE', 
            USER, 
            :OLD.OrderID || ' - ' || :OLD.QuantityOrdered
        );
    END IF;
END;
/

```
### Testing
```sql
DELETE FROM Orders WHERE OrderID = 1;
```
### Verify Logs
```sql
SELECT * FROM AuditLog;
```
![image](https://github.com/user-attachments/assets/c0de7e10-f4bf-409b-9af1-ced4f064a97b)

### Explanation
Insert:
NewData logs the OrderID and QuantityOrdered of the newly inserted row.
Update:
OldData captures the OrderID and QuantityOrdered before the update.
NewData captures the updated values.
Delete:
OldData captures the row data before deletion

### Tracking User Actions
#### LogUserActions

```sql
CREATE SEQUENCE UserActions_SEQ START WITH 1 INCREMENT BY 1;

CREATE OR REPLACE TRIGGER LogUserActions
AFTER LOGON ON DATABASE
DECLARE
    v_UserID VARCHAR2(50);
BEGIN
    -- Log user action for specific users (optional condition)
    v_UserID := USER;
    IF v_UserID NOT IN ('SYS', 'SYSTEM') THEN
        INSERT INTO UserActions (ActionID, UserID, Action)
        VALUES (UserActions_SEQ.NEXTVAL, v_UserID, 'Logged In');
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        -- Prevent errors from affecting the logon process
        NULL;
END;
/

```
### Grant Necessary Privileges: If the trigger is created by a privileged user, ensure that the UserActions table and UserActions_SEQ sequence are accessible to that user
```sql
GRANT INSERT, SELECT ON UserActions TO PUBLIC;
GRANT SELECT ON UserActions_SEQ TO PUBLIC;
```
### Restrictions
#### Access Control by User Roles
Use GRANT statements and custom roles to restrict access to specific tables or operations.
Create a view for non-privileged users to limit their access to sensitive data.
```sql
CREATE ROLE VolunteerRole;

GRANT SELECT ON Inventory TO VolunteerRole;
GRANT SELECT ON Crops TO VolunteerRole;

-- Create a view for volunteers to access only necessary information
CREATE OR REPLACE VIEW VolunteerInventoryView AS
SELECT CropID, Quantity, FreshnessStatus
FROM Inventory;

GRANT SELECT ON VolunteerInventoryView TO VolunteerRole;

```
### output 
![image](https://github.com/user-attachments/assets/802e74b2-0a37-47d8-8339-411f08fe79a8)

### Create a function to check user privileges before allowing operations.
```sql
CREATE OR REPLACE FUNCTION IsAuthorized(p_Role VARCHAR2) RETURN BOOLEAN AS
    v_Count NUMBER;
BEGIN
    SELECT COUNT(*)
    INTO v_Count
    FROM USER_ROLE_PRIVS
    WHERE GRANTED_ROLE = p_Role;

    RETURN v_Count > 0; -- Returns TRUE if the user has the specified role
END;
/
```
### test for DBA and thu_peacock_ user
```sql
BEGIN
    IF IsAuthorized('DBA') THEN
        DBMS_OUTPUT.PUT_LINE('User is authorized.');
    ELSE
        DBMS_OUTPUT.PUT_LINE('User is not authorized.');
    END IF;
END;
/
---
BEGIN
    IF IsAuthorized('thu_peacock_') THEN
        DBMS_OUTPUT.PUT_LINE('User is authorized.');
    ELSE
        DBMS_OUTPUT.PUT_LINE('User is not authorized.');
    END IF;
END;
/

```
![image](https://github.com/user-attachments/assets/9a13a30f-8d9d-4c91-bfaf-c5045ca4fac8)

###  Documentation: How Auditing Improves Security
### g) Scope and Limitations
#### h) Documentation and Demonstration

## structure for ou ppt
1. Report Outline
1.1 Title Page
Title: Database Programming for Urban Farming Management
Prepared by: [Your Name]
Date: [Submission Date]
1.2 Table of Contents
Problem Statement
Objectives and Scope
Database Design
Advanced Programming Features
Triggers
Cursors
Functions
Packages
Auditing
Testing and Validation
Scope and Limitations
Conclusion
