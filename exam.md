Examen SPABD


CREATE TABLE Customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(50),
    customer_email VARCHAR(100),
    customer_address VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_registered_date DATE
);


CREATE TABLE Products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(50),
    product_price DECIMAL(10, 2),
    product_description VARCHAR(200),
    product_category VARCHAR(50),
    product_stock INT
);


DROP TABLE IF EXISTS Transactions;

-- Create Transactions table with transaction_id as auto-increment primary key
CREATE TABLE Transactions (
    transaction_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    transaction_date DATE,
    quantity INT,
    total_amount DECIMAL(10, 2),
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);


INSERT INTO Customers (customer_id, customer_name, customer_email, customer_address, customer_phone, customer_registered_date)
VALUES
    (1, 'Radu M', 'radu@example.com', '123 Dristor', '0700000000', '2023-01-01'),
    (2, 'Alex A', 'alex1@example.com', '456 Victoriei', '0710000000', '2023-02-02'),
    (3, 'David D', 'david@example.com', '789 Vacaresti', '0720000000', '2023-03-03'),
    (4, 'Mihai G', 'mihai@example.com', '321 Unirii', '0730000000', '2023-04-04'),
    (5, 'Alex B', 'ale2@example.com', '654 Romana', '0400000000', '2023-05-05');


INSERT INTO Products (product_id, product_name, product_price, product_description, product_category, product_stock)
VALUES
    (1, 'Product A', 9.99, 'This is product A', 'Category 1', 50),
    (2, 'Product B', 19.99, 'This is product B', 'Category 2', 100),
    (3, 'Product C', 14.99, 'This is product C', 'Category 1', 25),
    (4, 'Product D', 24.99, 'This is product D', 'Category 3', 75),
    (5, 'Product E', 29.99, 'This is product E', 'Category 2', 80),
    (6, 'Product F', 11.99, 'This is product F', 'Category 1', 60),
    (7, 'Product G', 17.99, 'This is product G', 'Category 3', 90),
    (8, 'Product H', 39.99, 'This is product H', 'Category 2', 70),
    (9, 'Product I', 8.99, 'This is product I', 'Category 1', 40),
    (10, 'Product J', 22.99, 'This is product J', 'Category 3', 120);



INSERT INTO Transactions (transaction_id, customer_id, product_id, transaction_date, quantity, total_amount)
VALUES
    (2, 1, 3, '2023-06-02', 1, 14.99),
    (3, 2, 2, '2023-06-03', 3, 59.97),
    (4, 3, 5, '2023-06-04', 2, 59.98),
    (5, 4, 7, '2023-06-05', 1, 17.99),
    (6, 5, 4, '2023-06-06', 4, 99.96),
    (7, 1, 6, '2023-06-07', 2, 23.98),
    (8, 3, 1, '2023-06-08', 3, 29.97),
    (9, 4, 8, '2023-06-09', 1, 39.99),
    (10, 5, 10, '2023-06-10', 2, 45.98);


Triggers&Procedures

DELIMITER //

CREATE TRIGGER update_product_stock
AFTER INSERT ON Transactions
FOR EACH ROW
BEGIN
    DECLARE quantity_purchased INT;
    DECLARE product_id INT;

    
    SET quantity_purchased = NEW.quantity;
    SET product_id = NEW.product_id;

   
    UPDATE Products
    SET product_stock = product_stock - quantity_purchased
    WHERE product_id = product_id;
END//

DELIMITER ;



DELIMITER //

CREATE TRIGGER calculate_total_amount
BEFORE INSERT ON Transactions
FOR EACH ROW
BEGIN
    DECLARE quantity_purchased INT;
    DECLARE product_price DECIMAL(10, 2);

   
    SET quantity_purchased = NEW.quantity;
    SET product_price = (SELECT product_price FROM Products WHERE product_id = NEW.product_id);

    SET NEW.total_amount = quantity_purchased * product_price;
END//

DELIMITER ;


######################

DELIMITER//

CREATE PROCEDURE InsertCustomer(
    IN customerID INT,
    IN customerName VARCHAR(50),
    IN customerEmail VARCHAR(100),
    IN customerAddress VARCHAR(100),
    IN customerPhone VARCHAR(20),
    IN customerRegisteredDate DATE
)
BEGIN
    INSERT INTO Customers (customer_id, customer_name, customer_email, customer_address, customer_phone, customer_registered_date)
    VALUES (customerID, customerName, customerEmail, customerAddress, customerPhone, customerRegisteredDate);
END //

DELIMITER ;


CALL InsertCustomer(6, 'Robert J', 'robertj@example.com', '987 Unirii, ‘0770000000, '2023-06-08');


#######################################


DELIMITER //

-- Create stored procedure to buy a certain amount of a product
CREATE PROCEDURE BuyProduct(
    IN customerID INT,
    IN productID INT,
    IN quantity INT
)
BEGIN
    DECLARE totalAmount DECIMAL(10, 2);
    
    
    SELECT product_price INTO totalAmount
    FROM Products
    WHERE product_id = productID;
    
    
    SET totalAmount = totalAmount * quantity;
    
    
    INSERT INTO Transactions (customer_id, product_id, transaction_date, quantity, total_amount)
    VALUES (customerID, productID, CURDATE(), quantity, totalAmount);
    
    
    UPDATE Products
    SET product_stock = product_stock - quantity
    WHERE product_id = productID;
    
    SELECT 'Trnzactie Reusita!.’ AS Message;
END //

DELIMITER ;

CALL BuyProduct(1, 2, 3); 





DELIMITER //

CREATE PROCEDURE GetCustomerTransactions(IN customerID INT)
BEGIN
    SELECT Transactions.transaction_id, Customers.customer_name, Products.product_name,
           Transactions.transaction_date, Transactions.quantity, Transactions.total_amount
    FROM Transactions
    INNER JOIN Customers ON Transactions.customer_id = Customers.customer_id
    INNER JOIN Products ON Transactions.product_id = Products.product_id
    WHERE Customers.customer_id = customerID;
END//

DELIMITER ;

CALL GetCustomerTransactions(1);



