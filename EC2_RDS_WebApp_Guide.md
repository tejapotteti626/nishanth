
# AWS Project: EC2 Web App Connected to RDS MySQL in Private Subnet

## Overview

This project sets up a simple web application hosted on an EC2 instance in a **public subnet** that connects to a **MySQL RDS instance** located in a **private subnet**. The goal is to submit employee data through a PHP form and store it in the database. The data is also displayed in a table on the same page.

We will follow a step-by-step approach, from setting up the infrastructure to deploying and testing the PHP web application.

---

## 1. Architecture and Setup

### 1.1 Network Architecture

- **VPC**: Custom VPC with public and private subnets.
- **Public Subnet**: Contains the EC2 instance that hosts the web application.
- **Private Subnet**: Contains the MySQL RDS instance, inaccessible directly from the internet.
- **Security Groups**: Used to control traffic between EC2 and RDS.

### 1.2 Security Group Rules

- **EC2 SG**: Allows inbound HTTP (port 80) and SSH (port 22).
- **RDS SG**: Allows inbound MySQL (port 3306) **only from EC2's security group**.

---

## 2. Step-by-Step Implementation

### 2.1 RDS Setup

1. Go to **RDS → Create Database**.
2. Choose **MySQL** engine.
3. Set instance name, username, and password.
4. **Disable public access** (RDS should be private).
5. Place it inside the **private subnet group**.
6. Assign **RDS security group** to allow port 3306 from EC2 SG.

### 2.2 EC2 Setup

1. Go to **EC2 → Launch Instance**.
2. Choose **Amazon Linux 2**.
3. Place it inside the **public subnet**.
4. Assign **EC2 security group** with port 22 (SSH) and 80 (HTTP).
5. Connect using SSH and install Apache + PHP:

```bash
sudo yum update -y
sudo yum install -y httpd php php-mysqli
sudo systemctl start httpd
sudo systemctl enable httpd
```

---

## 3. Deploying the PHP Web App

### 3.1 `dbinfo.inc` File

```php
<?php
// dbinfo.inc
define('DB_SERVER', 'your-rds-endpoint.amazonaws.com');
define('DB_USERNAME', 'admin');
define('DB_PASSWORD', 'yourpassword');
define('DB_DATABASE', 'employeedb');
?>
```

### 3.2 `SamplePage.php` File

```php
<?php
include("dbinfo.inc");
$conn = new mysqli(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_DATABASE);
if ($conn->connect_error) { die("Connection failed: " . $conn->connect_error); }

// Create table if not exists
$conn->query("CREATE TABLE IF NOT EXISTS Employee_Info (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address TEXT NOT NULL
)");

if ($_SERVER["REQUEST_METHOD"] == "POST") {
    $name = $_POST["name"];
    $address = $_POST["address"];
    $stmt = $conn->prepare("INSERT INTO Employee_Info (name, address) VALUES (?, ?)");
    $stmt->bind_param("ss", $name, $address);
    $stmt->execute();
    $stmt->close();
}
?>

<html>
<body>
<h2>Employee Form</h2>
<form method="post">
    Name: <input type="text" name="name"><br>
    Address: <input type="text" name="address"><br>
    <input type="submit">
</form>

<h2>Employee Records</h2>
<table border="1">
<tr><th>ID</th><th>Name</th><th>Address</th></tr>
<?php
$result = $conn->query("SELECT * FROM Employee_Info");
while ($row = $result->fetch_assoc()) {
    echo "<tr><td>{$row['id']}</td><td>{$row['name']}</td><td>{$row['address']}</td></tr>";
}
$conn->close();
?>
</table>
</body>
</html>
```

### 3.3 Upload Files to EC2

```bash
scp -i key.pem SamplePage.php ec2-user@<EC2-IP>:/var/www/html/
scp -i key.pem dbinfo.inc ec2-user@<EC2-IP>:/var/www/html/
```

Ensure Apache has permission:

```bash
sudo chown apache:apache /var/www/html/*
```

---

## 4. Testing and Verification

- Open your browser and visit: `http://<EC2-Public-IP>/SamplePage.php`
- Enter name and address → Click Submit.
- Verify data appears in the table below.
- Check MySQL RDS instance using a DB client (optional) to confirm data insertion.

---

## 5. Security and Best Practices

- Do **not** hardcode passwords in source code for production — use environment variables or AWS Secrets Manager.
- Use **HTTPS** with SSL for secure data transfer.
- Regularly update packages on EC2.
- RDS should have backups, encryption, and automatic failover enabled if needed.

---

## Conclusion

You have successfully built a secure AWS-based architecture using EC2, RDS, PHP, and MySQL. This setup demonstrates VPC networking, private-public subnet communication, secure data input and retrieval using a basic web application.
