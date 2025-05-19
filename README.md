# Lift-and-Shift a LAMP Web App to EC2 and RDS

1. [Building and Testing the LAMP App Locally](#building-and-testing-the-lamp-app-locally)
2. [Configuring VPC and Subnets](#configuring-vpc-and-subnets)
3. [Configuring Security Groups and IAM](#configuring-security-groups-and-iam)
4. [Provisioning EC2 Instance and Deploying Web App](#provisioning-ec2-instance-and-deploying-web-app)
5. [Provisioning RDS MySQL](#provisioning-rds-mysql)
6. [Scripting Backup to S3](#scripting-backup-to-s3)
7. [Monitoring, Scaling, and Load Balancing](#monitoring-scaling-and-load-balancing)


## Building and Testing the LAMP App Locally
- In the Lubuntu VM, install Apache, MySQL and PHP
  ```
  sudo apt update
  sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql -y
  ```
  ![image](https://github.com/user-attachments/assets/754241b8-146e-44a3-9b21-f762664eab1d) <br />

- Secure MySQL
  ```
  sudo mysql_secure_installation
  ```
  ![image](https://github.com/user-attachments/assets/eec875b7-975b-4454-b1f1-4815464ba454) <br />
  ![image](https://github.com/user-attachments/assets/9279b46a-fc4d-4dc8-8a82-c1966062c093) <br />

- Create a sample LAMP app
  ```
  cd /var/www/html/
  sudo touch index.php
  ```
  ![image](https://github.com/user-attachments/assets/d97cea61-146c-48ac-b864-8e50d6a17b21) <br />

  Use the following PHP code below
  ```
  <?php
  $servername = "localhost";
  $username = "root";
  $password = "your_mysql_password";
  $dbname = "testdb";
  
  // Create connection
  $conn = new mysqli($servername, $username, $password, $dbname);
  
  // Check connection
  if ($conn->connect_error) {
      die("Connection failed: " . $conn->connect_error);
  }
  
  // Handle form submission
  if ($_SERVER["REQUEST_METHOD"] == "POST") {
      $name = $conn->real_escape_string($_POST["name"]);
      $sql = "INSERT INTO users (name) VALUES ('$name')";
      if ($conn->query($sql) === TRUE) {
          echo "New record created successfully.<br>";
      } else {
          echo "Error: " . $sql . "<br>" . $conn->error;
      }
  }
  
  // Display form
  ?>
  <!DOCTYPE html>
  <html>
  <head>
      <title>Simple LAMP Web App</title>
  </head>
  <body>
      <h1>Submit Your Name</h1>
      <form method="post">
          <input type="text" name="name" placeholder="Enter your name" required>
          <input type="submit" value="Submit">
      </form>
      <h2>Stored Users</h2>
      <ul>
          <?php
          $result = $conn->query("SELECT name, created_at FROM users ORDER BY id DESC");
          while($row = $result->fetch_assoc()) {
              echo "<li>" . htmlspecialchars($row["name"]) . " - " . $row["created_at"] . "</li>";
          }
          ?>
      </ul>
  </body>
  </html>
  <?php $conn->close(); ?>
  ```
  ![image](https://github.com/user-attachments/assets/00226836-7a4c-4cff-bc0d-2f589e072416) <br />

- Create MySQL DB. The password to be entered is literally `your_mysql_password`
  ```
  sudo mysql -u root -p
  CREATE DATABASE testdb;
  ```
  ![image](https://github.com/user-attachments/assets/a1fe690e-0f6b-462f-828c-ecefc952a6f7) <br />

  Then create a table in testdb
  ```
  USE testdb;
  CREATE TABLE users (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(100) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  ```
  To view tables in MySQL, use
  ```
  SHOW TABLES;
  ```
  And to describe the table's structure, use
  ```
  DESCRIBE users;
  ```
  Use the following command to query the table to see the data
  ```
  SELECT * FROM users;
  ```
  ![image](https://github.com/user-attachments/assets/fc67671e-9732-4270-8e05-05b8517965f2) <br />

- Check if both Apache and MySQL services are running
  ```
  sudo systemctl status apache2
  sudo systemctl status mysql
  ```
  ![image](https://github.com/user-attachments/assets/6b8d3d8d-1d6d-4350-8051-5f2954393994) <br />

- Test the web app. Open a web browser and go to `http://<your-lubuntu-ip>/index.php`. The MySQL connection message should be displayed.
- Surprisingly, nothing is displayed in the web browser despite correct code and URL address <br />
  ![image](https://github.com/user-attachments/assets/82dc9911-8973-419c-b4af-27d0d2b8fbd2) <br />

- Reconfirm that the PHP file is serving from the `/var/www/html` directory. Use
  ```
  ls -l /var/www/html/
  ```
  It will also display the Linux file permissions <br />
  ![image](https://github.com/user-attachments/assets/138f1939-7e54-4fc1-94f9-f1bc72b507e8) <br />

- Check if Apache is actually listening on port 80. Run
  ```
  sudo netstat -tulnp | grep apache
  ```
  or
  ```
  sudo ss -tuln | grep :80
  ```
  ![image](https://github.com/user-attachments/assets/2880d265-1cb3-4de4-a68f-a935fa3595d3) <br />

- Since the `/var/www/html` directory has an `index.html` file, it is opened in the web browser to confirm if it can be accessed <br />
  ![image](https://github.com/user-attachments/assets/6f2657a9-bb47-48e7-918f-1cb410dd2d39) <br />

- Check if PHP is really installed. If yes, restart Apache service and check its status
  ```
  php -v
  sudo systemctl restart apache2
  sudo systemctl status apache2
  ```
  ![image](https://github.com/user-attachments/assets/a1c41716-e5b1-4ab0-a527-07df8a8e3cda) <br />

- To confirm if Apache is parsing PHP, create a test file
  ```
  echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
  ```
  Then open it in the web browser <br />
  ![image](https://github.com/user-attachments/assets/722c385e-1ef9-4a1c-9f68-0ecf5324fe7d) <br />
  ![image](https://github.com/user-attachments/assets/1d82204f-e297-4aa8-bfe3-907d85188bb8) <br />

- Since PHP is working, another troubleshooting step is to prioritiese `index.php` before `index.html`
  ```
  sudo nano /etc/apache2/mods-enabled/dir.conf
  ```
  ![image](https://github.com/user-attachments/assets/4bf28df6-683d-419a-b9e3-5402eae6881f) <br />
  Then make sure it looks like the following
  ```
  DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
  ```
  ![image](https://github.com/user-attachments/assets/e17b8e3d-81e8-44c8-918b-554be8a115aa) <br />
  Once this is done, reload Apache and reopen `index.php` in the web browser <br />
  ![image](https://github.com/user-attachments/assets/5d572917-2c3d-499a-82c4-2d44772140a9) <br />
  Unfortunately a blank page is still being displayed

- Since PHP disables error reporting by default as a security measure, add the following code into `index.php`
  ```
  <?php
  error_reporting(E_ALL);
  ini_set('display_errors', 1);
  ?>
  ```
  ![image](https://github.com/user-attachments/assets/795892ed-1253-4cd3-8b3b-5c2de62b9170) <br />

- Reloading the web browser now gives us this information <br />
  ![image](https://github.com/user-attachments/assets/33bfc894-4cb2-41d0-8433-8528c0d9530d) <br />
  This means the PHP script is trying to connect to MySQL as root, but either the password is incorrect, or the root user is not allowed to connect from the web server (localhost via PHP). Double-check the MySQL root password
  ```
  mysql -u root -p
  ```
  Try with the password `your_mysql_password` <br />
  ![image](https://github.com/user-attachments/assets/0326ac5a-b407-4bed-a42f-6d9bece0869b) <br />
  Apparently access is denied due to typo error during initial password setup

- To fully reset MySQL root password, first thing to do is to stop the MySQL service
  ```
  sudo systemctl stop mysql
  ```
- Then start MySQL in safe mode to skip permission checks 
  ```
  sudo mysqld_safe --skip-grant-tables &
  ```
- Open another terminal and connect to MySQL
  ```
  mysql -u root
  ```

- Set a new password then exit
  ```
  FLUSH PRIVILEGES;
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'newpassword';
  ```

- Stop the safe-mode MySQL and restart the service normally
  ```
  sudo killall mysqld
  sudo systemctl start mysql
  ```
  Update `index.php` with the new password and test the new password
  ```
  mysql -u root -p
  ```
- Now open `index.php` in web browser to see if it now works as expected


- Export the DB for future migration
  ```
  mysqldump -u root -p testdb > testdb.sql
  ```

## Configuring VPC and Subnets
- Create a custom VPC. CIDR block: 10.0.0.0/16
- Create subnets
  Public subnet (e.g., 10.0.1.0/24)
  Private subnet (e.g., 10.0.2.0/24)
- Create Internet Gateway
  Attach it to the VPC
- Create NAT Gateway (Optional for RDS outbound access)
  Launch in public subnet, requires Elastic IP
- Create Route Tables
  Public route table: route 0.0.0.0/0 to Internet Gateway
  Associate with public subnet
  Private route table: route 0.0.0.0/0 to NAT Gateway (optional)


## Configuring Security Groups and IAM
- Security Groups
  Web-SG (for EC2):
  Allow HTTP (80), HTTPS (443) from anywhere (0.0.0.0/0)
  Allow SSH (22) from your IP

  DB-SG (for RDS):
  Allow MySQL (3306) only from Web-SG

- Create IAM Role for EC2
  Permissions: AmazonS3FullAccess (or fine-tune later)
  Attach to EC2 instance


## Provisioning EC2 Instance and Deploying Web App
- Launch EC2 Instance

  AMI: Amazon Linux 2 or Ubuntu
  
  Instance Type: t2.micro
  
  Subnet: public subnet
  
  Security Group: Web-SG
  
  IAM Role: attach previously created role
  
  Elastic IP (optional for static IP)

- SSH into EC2 and install LAMP
  ```
  sudo yum update -y  # or sudo apt update
  sudo yum install httpd php php-mysqlnd -y  # or apache2, php, php-mysql
  sudo systemctl start httpd
  sudo systemctl enable httpd
  ```

- Copy your PHP app to EC2

  Use `scp` or copy-paste code
  
  Place it in `/var/www/html`


## Provisioning RDS MySQL

- Launch RDS instance

  Engine: MySQL
  
  DB instance size: db.t2.micro
  
  Subnet group: private subnet
  
  Security group: DB-SG
  
  Enable public access: No
  
  Enable auto backups

- Note the endpoint, e.g., mydb.xxxxxxx.rds.amazonaws.com
- In EC2, connect to RDS to test
  ```
  mysql -u admin -p -h <rds-endpoint>
  ```
- Import the DB dump
  ```
  mysql -u admin -p -h <rds-endpoint> testdb < testdb.sql
  ```
- Update the web app DB host to point to RDS


## Scripting Backup to S3
- Create S3 bucket
  Name: `lamp-db-backups-yourname`
- Backup script (on EC2)
  ```
  #!/bin/bash
  DATE=$(date +%F)
  mysqldump -u admin -p'yourpass' -h <rds-endpoint> testdb > /tmp/testdb-$DATE.sql
  aws s3 cp /tmp/testdb-$DATE.sql s3://lamp-db-backups-yourname/
  find /tmp -name "testdb-*.sql" -type f -mtime +7 -exec rm {} \;
  ```

- Make script executable and test
  ```
  chmod +x backup.sh
  ./backup.sh
  ```

- An optional step is to add a cron job that runs daily at 2 am
  ```
  crontab -e
  0 2 * * * /home/ec2-user/backup.sh
  ```

## Monitoring, Scaling, and Load Balancing
- Enable CloudWatch Alarms
  EC2 CPU > 70%
  
  RDS free storage < 20%

- (Optional) Create ALB
  Target group: EC2 instance
  
  Listener: HTTP (80)
  
  DNS: Route 53 or use ALB DNS

- (Optional) Launch 2nd EC2 in different AZ
  Attach to same target group for scaling demo










