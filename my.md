# AWS Project: EC2 Web App Connected to RDS MySQL in Private Subnet

## Overview

This project creates a basic web application using an EC2 instance placed in a public subnet. This EC2 instance connects to a MySQL RDS database that is placed in a private subnet. The main purpose of this setup is to allow users to submit employee details using a PHP form. These details are then saved in the database and shown on the same web page in a table. The whole process includes setting up the infrastructure, deploying the PHP web app, and testing it step by step.

---

## 1. Architecture and Setup (Simplified & Explained)

### 1.1 Network Architecture (How things are organized)

In this project, we divide our cloud network into two parts to improve both security and functionality:

**Public Subnet**: This is the part of the network that is open to the internet. We place our EC2 instance (a virtual server) here. Since it’s in the public subnet, it can be accessed from anywhere using the internet. This is useful because we can connect to it through SSH (a secure way to log in remotely), and we can also use it to host our PHP web application that users can access from their browsers.

**Private Subnet**: This is the part of the network that is not connected directly to the internet. We place our RDS (Relational Database Service) MySQL database here. Since it’s in a private subnet, it cannot be accessed from outside the network, which helps keep it safe from hackers or unwanted access. Only the EC2 instance in the public subnet is allowed to connect to it and send or retrieve data.

To organize and manage both of these subnets, we use something called a VPC (Virtual Private Cloud). A VPC is like your own private data center inside AWS. It allows you to fully control your network, including how your servers and databases communicate with each other.

So, inside the VPC, we create:

A **public subnet**, where we run the EC2 instance to host our website.

A **private subnet**, where we place the RDS database to store user data securely.

This setup is designed to work like a real-world system: the users only interact with the web interface (EC2), and the database (RDS) stays hidden in the background, protected from the internet, and only reachable by the EC2 instance.

### 1.2 Security Group Rules (Allowing traffic safely)

**Security Groups** are like firewalls. They control who can access our EC2 and RDS. Here's how we configure them:

- **EC2 Security Group**:  
  - Allows HTTP traffic on **port 80**, so users can access the website.
  - Allows SSH on **port 22**, so we can log into the EC2 instance.

- **RDS Security Group**:  
  - Allows **only MySQL traffic on port 3306**, and only **from the EC2's security group**. This means **only the EC2 instance can talk to the RDS**, and no one else.

This ensures that our website is accessible, but the database stays secure and private.

---

## 2. Step-by-Step Implementation (Simple walkthrough)

### 2.1 RDS Setup (Creating a secure MySQL database)

1. Go to the **RDS section** in AWS and click **Create Database**.
2. Select **MySQL** as the database engine.
3. Set up:
   - A database name
   - A username (e.g., `admin`)
   - A strong password
4. Under "Connectivity":
   - Choose **"Do not make it publicly accessible"** – this ensures the database stays inside the private subnet.
   - Attach the **private subnet group**, so RDS launches in the private zone.
5. Choose the **RDS security group** that **only allows access from EC2**.

👉 This ensures the database is secure and only connected to the web server.

### 2.2 EC2 Setup (Creating the web server)

1. Go to the **EC2 section** in AWS and click **Launch Instance**.
2. Choose the **Amazon Linux 2** AMI (free tier eligible).
3. Place the instance in the **public subnet** so we can access it from the internet.
4. Attach the **EC2 security group**, which:
   - Allows **SSH (port 22)** so you can log in remotely
   - Allows **HTTP (port 80)** so your web application can be accessed
5. After launching, connect to the instance using SSH and install the web server:

```bash
sudo yum update -y
sudo yum install -y httpd php php-mysqli
sudo systemctl start httpd
sudo systemctl enable httpd
```

👉 This sets up Apache and PHP so that our web server is ready to serve a PHP website.

---

## 3. Deploying the PHP Web App (Simplified)

### 3.1 Create Configuration File

Create a file named `dbinfo.inc` to store your database credentials securely (for demo only):

```php
<?php
// dbinfo.inc
define('DB_SERVER', 'your-rds-endpoint.amazonaws.com');
define('DB_USERNAME', 'admin');
define('DB_PASSWORD', 'yourpassword');
define('DB_DATABASE', 'employeedb');
?>
```

### 3.2 Create Web Page

Create a file named `SamplePage.php` with a simple form to add employee data and display it:

```php
<?php
include("dbinfo.inc");
$conn = new mysqli(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_DATABASE);
if ($conn->connect_error) { die("Connection failed: " . $conn->connect_error); }

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

Set proper permissions:

```bash
sudo chown apache:apache /var/www/html/*
```

---

## 4. Testing and Verification (Simple Guide)

1. Open your browser and go to: `http://<EC2-Public-IP>/SamplePage.php`
2. Fill out the form with name and address → Click Submit.
3. You should see the submitted data appear in the table.
4. You can also verify entries inside your RDS MySQL instance using a MySQL client.

✅ This confirms your EC2 is able to communicate with the RDS and everything works end to end.

---

## 5. Security and Best Practices (Final Touches)

--Never hardcode passwords in code: In production environments, you should never write your database username or password directly in your code. Instead, use AWS Secrets Manager or environment variables to store and access sensitive information in a secure way.

--Enable HTTPS: Always use HTTPS to make sure the data that travels between the user and your web application is encrypted and secure. You can do this by setting up an SSL certificate.

--Keep your software updated: Regularly install updates and security patches for your EC2 instance and your PHP application to protect against known bugs and vulnerabilities.

--Turn on automated backups for your RDS: This helps you avoid data loss in case something goes wrong. Backups allow you to restore your database to a previous state if needed.

--Use IAM roles and least privilege access: If your EC2 instance needs to connect to other AWS services (like S3 or Secrets Manager), give it an IAM role with only the minimum permissions it needs. This keeps your setup more secure.

---

## Conclusion

You have successfully built a secure AWS-based architecture using EC2, RDS, PHP, and MySQL. This setup demonstrates VPC networking, private-public subnet communication, secure data input and retrieval using a basic web application.
