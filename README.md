# Lift-and-Shift a LAMP Web App to EC2 and RDS


1. []()


## Build and Test the LAMP App Locally
- In the Lubuntu VM, install Apache, MySQL and PHP
  ```
  sudo apt update
  sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql -y
  ```

- Secure MySQL
  ```
  sudo mysql_secure_installation
  ```

- Create a sample LAMP app
  ```
  cd /var/www/html/
  touch index.php
  ```
  Use the following PHP code below
  ```
  <?php
  $conn = new mysqli("localhost", "root", "your_mysql_password", "testdb");
  if ($conn->connect_error) {
      die("Connection failed: " . $conn->connect_error);
  }
  echo "Connected to MySQL successfully!";
  ?>
  ```

- Create MySQL DB
  ```
  sudo mysql -u root -p
  CREATE DATABASE testdb;
  ```

- Test the web app. Open a web browser and go to `http://<your-lubuntu-ip>/index.php`. The MySQL connection message should be displayed

- Export the DB for future migration
  ```
  mysqldump -u root -p testdb > testdb.sql
  ```

## VPC and Subnets
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


## Security Groups and IAM
- Security Groups
  Web-SG (for EC2):
  Allow HTTP (80), HTTPS (443) from anywhere (0.0.0.0/0)
  Allow SSH (22) from your IP

  DB-SG (for RDS):
  Allow MySQL (3306) only from Web-SG

- Create IAM Role for EC2
  Permissions: AmazonS3FullAccess (or fine-tune later)
  Attach to EC2 instance


## Provision EC2 Instance and Deploy Web App
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


## Provision RDS MySQL

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


## Backup to S3
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

## Monitoring, Scaling, and Load Balancer
- Enable CloudWatch Alarms
  EC2 CPU > 70%
  
  RDS free storage < 20%

- (Optional) Create ALB
  Target group: EC2 instance
  
  Listener: HTTP (80)
  
  DNS: Route 53 or use ALB DNS

- (Optional) Launch 2nd EC2 in different AZ
  Attach to same target group for scaling demo










