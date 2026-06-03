# 🛢️ AWS Database Private Access and Connectivity

This guide is a deep, implementation-focused reference for designing, deploying, securing, and troubleshooting **private connectivity to AWS database services**.

It is written for cloud engineers, platform teams, network engineers, DBAs, SREs, and security architects who need to keep database traffic off the public internet while still enabling application, administrative, and hybrid access.

> Replace example VPC IDs, subnet IDs, security groups, account IDs, ARNs, KMS keys, DNS zones, Regions, and credentials before running commands.

## 📚 Table of Contents

1. [AWS Database Services Overview](#1-aws-database-services-overview)
2. [Setting Up RDS (Basic to Advanced)](#2-setting-up-rds-basic-to-advanced)
3. [Private Access via VPC](#3-private-access-via-vpc)
4. [VPC Endpoints for AWS Database Services](#4-vpc-endpoints-for-aws-database-services)
5. [Connecting from On-Premises](#5-connecting-from-on-premises)
6. [Bastion Host / Systems Manager Access](#6-bastion-host--systems-manager-access)
7. [Aurora Serverless & Global Database](#7-aurora-serverless--global-database)
8. [DynamoDB Access Patterns](#8-dynamodb-access-patterns)
9. [Database Security](#9-database-security)
10. [Database Troubleshooting](#10-database-troubleshooting)
11. [Appendix A: Connectivity Validation Checklist](#11-appendix-a-connectivity-validation-checklist)
12. [Appendix B: Operational Command Library](#12-appendix-b-operational-command-library)
13. [Appendix C: FAQ](#13-appendix-c-faq)
14. [Appendix D: Engine-by-Engine Connectivity Reference](#14-appendix-d-engine-by-engine-connectivity-reference)
15. [Appendix E: Infrastructure as Code Patterns](#15-appendix-e-infrastructure-as-code-patterns)
16. [Appendix F: Real-World Architecture Scenarios](#16-appendix-f-real-world-architecture-scenarios)
17. [Appendix G: Extended Troubleshooting Matrix](#17-appendix-g-extended-troubleshooting-matrix)
18. [Appendix H: Pre-Production Review Worksheet](#18-appendix-h-pre-production-review-worksheet)
19. [Appendix I: Cutover and Rollback Questions](#19-appendix-i-cutover-and-rollback-questions)

## 1. AWS Database Services Overview

Private access strategy starts with choosing the right data service. Each AWS database product has different network behavior, authentication patterns, failover models, and endpoint types.

The services below are the most common building blocks for modern AWS data platforms:

- **Amazon RDS** for managed relational engines with familiar operational models.
- **Amazon Aurora** for cloud-native relational databases with separated compute and distributed storage.
- **Amazon DynamoDB** for serverless NoSQL key-value and document workloads.
- **Amazon ElastiCache** for Redis OSS, Valkey, or Memcached caching tiers inside a VPC.
- **Amazon DocumentDB** for MongoDB-compatible document workloads inside a VPC.
- **Amazon Neptune** for graph workloads.
- **Amazon Redshift** for analytics and warehousing.
- **Amazon MemoryDB** for durable in-memory Redis-compatible workloads.

| Service | Model | Primary access pattern | Typical network model | Private access notes |
| --- | --- | --- | --- | --- |
| Amazon RDS | Managed relational | TCP client to DB endpoint | DB endpoint in VPC | Best practice is private subnets, SG allowlists, optional RDS Proxy |
| Amazon Aurora | Cloud-native relational | Cluster / instance endpoint | DB endpoint in VPC | Private-only clusters are common; supports Global Database and Data API in some patterns |
| Amazon DynamoDB | Serverless NoSQL | HTTPS API | Public AWS endpoint or VPC endpoint | Use gateway or interface VPC endpoints and IAM policies |
| Amazon ElastiCache | Managed cache | TCP to cache endpoint | Private IPs in VPC | Nodes are private; use SGs and subnets; API can be reached with interface endpoints |
| Amazon DocumentDB | Document DB | Mongo-compatible endpoint | Private IPs in VPC | Private subnet placement, SGs, TLS, Secrets Manager recommended |
| Amazon Neptune | Graph DB | Gremlin / openCypher / SPARQL endpoint | Private IPs in VPC | Route design and SG scoping matter because traffic stays in VPC |
| Amazon Redshift | Data warehouse | JDBC/ODBC/SQL endpoint | Private or public endpoint | Prefer private cluster, Redshift-managed VPC endpoints where relevant |
| Amazon MemoryDB | Durable in-memory | Redis-compatible endpoint | Private IPs in VPC | Designed for private application access with SG-based controls |

### Amazon RDS

- **Best for:** traditional line-of-business applications, packaged software, ERP integrations, and transactional systems that require managed MySQL, PostgreSQL, MariaDB, Oracle, or SQL Server.
- **Engines or protocol family:** MySQL, PostgreSQL, MariaDB, Oracle, SQL Server.
- **Private connectivity guidance:** place instances in private subnets, restrict ingress with security groups, and expose administration through SSM or bastion hosts rather than public access.
- **Common authentication modes:** native DB users, IAM DB authentication for supported engines, Secrets Manager rotation, Kerberos/AD for certain engines.
- **Operational features to plan around:** automated backups, Multi-AZ, read replicas, Performance Insights, Enhanced Monitoring.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon Aurora

- **Best for:** high-throughput relational systems, SaaS platforms, microservices, and global applications that want fast failover and storage-layer replication.
- **Engines or protocol family:** Aurora MySQL, Aurora PostgreSQL.
- **Private connectivity guidance:** use private cluster endpoints and reader endpoints inside the application VPC or through Transit Gateway / hybrid routing.
- **Common authentication modes:** database users, IAM DB auth for supported versions, Secrets Manager, Data API in serverless-compatible patterns.
- **Operational features to plan around:** cluster endpoints, reader endpoints, Global Database, Serverless v2, fast restore and clone capabilities.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon DynamoDB

- **Best for:** high-scale key-value, event-driven, gaming, session, cart, and serverless workloads with predictable access patterns.
- **Engines or protocol family:** N/A.
- **Private connectivity guidance:** data plane access can stay private through gateway or interface VPC endpoints, including on-premises access through PrivateLink where supported.
- **Common authentication modes:** IAM identity-based policies, resource policies, condition keys, VPC endpoint policies.
- **Operational features to plan around:** Global Tables, on-demand capacity, streams, TTL, DAX, backups, PITR.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon ElastiCache

- **Best for:** sub-millisecond cache, session store, pub/sub, real-time ranking, feature stores, and rate limiting.
- **Engines or protocol family:** Valkey, Redis OSS, Memcached.
- **Private connectivity guidance:** cache nodes are private by design inside selected subnets; control plane API access can use interface endpoints where supported.
- **Common authentication modes:** security groups, AUTH token, ACLs for Redis/Valkey, IAM integration depending on feature set.
- **Operational features to plan around:** replication groups, automatic failover, snapshots, cluster mode, serverless cache options in newer patterns.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon DocumentDB

- **Best for:** document-centric applications that prefer MongoDB APIs without self-managing replicas and backups.
- **Engines or protocol family:** DocumentDB engine.
- **Private connectivity guidance:** cluster endpoints are private and typically consumed from application subnets through SG allowlists.
- **Common authentication modes:** database credentials, TLS, Secrets Manager.
- **Operational features to plan around:** replicas, storage scaling, snapshots, monitoring via CloudWatch.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon Neptune

- **Best for:** fraud graphs, knowledge graphs, identity graphs, network topology, and recommendation workloads.
- **Engines or protocol family:** Neptune engine.
- **Private connectivity guidance:** endpoints are private in a VPC; routing and SGs must allow graph clients from app or analytics tiers.
- **Common authentication modes:** IAM auth, SigV4 for some endpoints, SGs, TLS.
- **Operational features to plan around:** replicas, snapshots, cluster failover, Streams and integration to analytics pipelines.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon Redshift

- **Best for:** BI, data warehousing, data lake query federation, ELT pipelines, and dashboarding workloads.
- **Engines or protocol family:** Redshift / Redshift Serverless.
- **Private connectivity guidance:** prefer private clusters and control access via SGs, subnet groups, and routing from analytics VPCs or shared services VPCs.
- **Common authentication modes:** database credentials, IAM federation, temporary credentials, Secrets Manager.
- **Operational features to plan around:** workgroups, snapshots, concurrency scaling, RA3 managed storage, data sharing.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Amazon MemoryDB

- **Best for:** durable in-memory primary databases, gaming state, session state, and microservices needing fast reads/writes with persistence.
- **Engines or protocol family:** Redis-compatible.
- **Private connectivity guidance:** use private subnets and SG-based access; no public routing should be required.
- **Common authentication modes:** ACLs, users, TLS, SGs.
- **Operational features to plan around:** multi-AZ durability, snapshots, shard scaling, failover.
- **When not to choose it:** avoid it when the access pattern, transaction model, or analytics model does not fit the service's design center.

### Comparison Guidance

- Choose **RDS** when vendor compatibility or an existing relational engine is the top requirement.
- Choose **Aurora** when you want managed relational scale with fast failover and a storage subsystem designed for AWS-native resilience.
- Choose **DynamoDB** when the application can be modeled around partition keys, sort keys, and explicit access patterns.
- Choose **ElastiCache** when the database is not the system of record and the main goal is low-latency cache or ephemeral state.
- Choose **DocumentDB** when teams need MongoDB-compatible drivers but do not want to run MongoDB themselves.
- Choose **Neptune** when graph traversal is a first-class query pattern.
- Choose **Redshift** when analytic concurrency, columnar storage, and warehouse SQL matter more than OLTP transactions.
- Choose **MemoryDB** when the in-memory tier is also required to be durable and highly available.

| Scenario | Recommended service | Why private connectivity matters |
| --- | --- | --- |
| Three-tier internal HR application | RDS PostgreSQL or RDS SQL Server | Sensitive employee records should only be reachable from app subnets and admin access paths |
| Global SaaS with read-heavy APIs | Aurora PostgreSQL with reader endpoints | Private readers reduce exposure while allowing multi-AZ app scaling |
| Gaming leaderboard with unpredictable traffic | DynamoDB + DAX | Private endpoints and IAM policies keep low-latency API calls off the internet |
| Session cache for ecommerce | ElastiCache for Redis/Valkey | Cache nodes remain inside the VPC and only app tiers can reach them |
| Fraud graph analytics | Neptune | Graph endpoints should be reachable from analytics tools over private network paths |
| Internal warehouse for finance | Redshift | Analytic clusters often contain regulated data and should be reachable only from corp networks or BI subnets |

## 2. Setting Up RDS (Basic to Advanced)

### RDS Provisioning Workflow

```mermaid
flowchart TD
    A[Choose engine and version] --> B[Create DB subnet group]
    B --> C[Create security group rules]
    C --> D[Pick instance class and storage]
    D --> E[Enable backups and monitoring]
    E --> F[Decide on private access only]
    F --> G[Launch DB instance]
    G --> H[Validate DNS, routing, SGs, and TLS]
    H --> I[Store secrets in Secrets Manager]
    I --> J[Connect apps or RDS Proxy]

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class A,B,C,D,E,F,G,H,I,J aws;
```

### Step 1: Plan prerequisites

1. Choose the target engine and version that is supported by your applications, drivers, and compliance standards.
2. Decide whether the database will be **single-AZ**, **Multi-AZ**, or backed by **read replicas**.
3. Select at least two private subnets in separate Availability Zones for the DB subnet group.
4. Create or identify an application security group that should be allowed to connect to the database.
5. Create a dedicated database security group with only the required inbound port open from trusted source security groups.
6. Decide whether credentials will be static, rotated through Secrets Manager, or replaced by IAM database authentication.
7. Pick instance class family and storage type based on latency, throughput, and budget goals.
8. Define backup retention, maintenance window, log exports, deletion protection, and KMS encryption requirements.
9. Plan how administrators will reach the database: Session Manager port forwarding, bastion host, VPN, or Direct Connect.
10. Record the DNS zones, VPC IDs, route tables, and NACLs that could influence private reachability.

### Creating an RDS instance in the AWS Console

1. Open **Amazon RDS** in the target Region.
2. Choose **Create database**.
3. Pick **Standard create** to expose the full set of engine and networking controls.
4. Select the engine family: MySQL, PostgreSQL, MariaDB, Oracle, or SQL Server.
5. Choose the engine version that matches application support requirements.
6. Select a template such as Dev/Test or Production; production templates surface HA defaults.
7. Enter the DB instance identifier and the master username.
8. Choose **Manage master credentials in AWS Secrets Manager** if you want AWS-managed rotation at creation time.
9. Select an instance class such as `db.t4g.medium`, `db.r6g.large`, or `db.m6i.xlarge`.
10. Set storage size and choose `gp3`, `io1/io2`, or magnetic if legacy compatibility is required.
11. Enable storage autoscaling if you want AWS to grow storage up to a defined ceiling.
12. Under **Connectivity**, choose the target VPC.
13. Choose a **DB subnet group** that points to private subnets in multiple AZs.
14. Set **Public access** to **No** for private-only connectivity.
15. Attach the dedicated DB security group.
16. Set the database port or keep the engine default such as 5432 or 3306.
17. Enable Enhanced Monitoring and Performance Insights if deeper diagnostics are needed.
18. Enable automatic backups and set retention, backup window, and log exports.
19. Select a KMS key for encryption at rest.
20. Optionally enable Multi-AZ for production HA.
21. Review maintenance, deletion protection, and tags.
22. Create the database and wait until the status is **available**.
23. Copy the endpoint, port, CA bundle reference, and Secret ARN for application onboarding.
24. Validate private DNS from a compute instance in the same VPC before handing the endpoint to developers.

### AWS CLI: RDS instance creation

```bash
aws rds create-db-subnet-group       --db-subnet-group-name prod-db-subnets       --db-subnet-group-description "Private database subnets"       --subnet-ids subnet-aaa111 subnet-bbb222 subnet-ccc333

aws ec2 create-security-group       --group-name prod-db-sg       --description "Database access from app tier"       --vpc-id vpc-1234567890abcdef0

aws ec2 authorize-security-group-ingress       --group-id sg-db123456       --ip-permissions '[
    {
      "IpProtocol": "tcp",
      "FromPort": 5432,
      "ToPort": 5432,
      "UserIdGroupPairs": [{"GroupId": "sg-app123456"}]
    }
  ]'

aws rds create-db-instance       --db-instance-identifier prod-postgres-01       --engine postgres       --engine-version 15.5       --db-instance-class db.r6g.large       --allocated-storage 200       --storage-type gp3       --master-username masteruser       --manage-master-user-password       --db-subnet-group-name prod-db-subnets       --vpc-security-group-ids sg-db123456       --backup-retention-period 7       --multi-az       --storage-encrypted       --kms-key-id arn:aws:kms:us-east-1:111122223333:key/abcd-1234       --publicly-accessible false       --enable-performance-insights       --monitoring-interval 60       --monitoring-role-arn arn:aws:iam::111122223333:role/rds-monitoring-role       --deletion-protection       --tags Key=Environment,Value=prod Key=Owner,Value=data-platform
```

### Terraform: private RDS deployment

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

variable "vpc_id" {}
variable "private_subnet_ids" {
  type = list(string)
}
variable "app_sg_id" {}

resource "aws_db_subnet_group" "prod" {
  name       = "prod-db-subnets"
  subnet_ids = var.private_subnet_ids
  tags = {
    Name        = "prod-db-subnets"
    Environment = "prod"
  }
}

resource "aws_security_group" "db" {
  name        = "prod-db-sg"
  description = "Allow PostgreSQL from application tier"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_sg_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prod-db-sg"
  }
}

resource "aws_db_instance" "postgres" {
  identifier                              = "prod-postgres-01"
  engine                                  = "postgres"
  engine_version                          = "15.5"
  instance_class                          = "db.r6g.large"
  allocated_storage                       = 200
  storage_type                            = "gp3"
  db_subnet_group_name                    = aws_db_subnet_group.prod.name
  vpc_security_group_ids                  = [aws_security_group.db.id]
  username                                = "masteruser"
  manage_master_user_password             = true
  multi_az                                = true
  publicly_accessible                     = false
  storage_encrypted                       = true
  backup_retention_period                 = 7
  performance_insights_enabled            = true
  monitoring_interval                     = 60
  deletion_protection                     = true
  skip_final_snapshot                     = false
  final_snapshot_identifier               = "prod-postgres-01-final"
  enabled_cloudwatch_logs_exports         = ["postgresql", "upgrade"]
  auto_minor_version_upgrade              = true
  iam_database_authentication_enabled     = true
  apply_immediately                       = false

  tags = {
    Environment = "prod"
    Service     = "billing"
  }
}
```

### CloudFormation: private RDS deployment

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Private PostgreSQL RDS deployment

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  PrivateSubnetA:
    Type: AWS::EC2::Subnet::Id
  PrivateSubnetB:
    Type: AWS::EC2::Subnet::Id
  AppSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id

Resources:
  DbSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow PostgreSQL from application tier
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref AppSecurityGroupId
      Tags:
        - Key: Name
          Value: prod-db-sg

  DbSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Private DB subnets
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
      DBSubnetGroupName: prod-db-subnets

  DbInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: prod-postgres-01
      Engine: postgres
      EngineVersion: '15.5'
      DBInstanceClass: db.r6g.large
      AllocatedStorage: '200'
      StorageType: gp3
      PubliclyAccessible: false
      MultiAZ: true
      BackupRetentionPeriod: 7
      DBSubnetGroupName: !Ref DbSubnetGroup
      VPCSecurityGroups:
        - !GetAtt DbSecurityGroup.GroupId
      ManageMasterUserPassword: true
      MasterUsername: masteruser
      StorageEncrypted: true
      EnablePerformanceInsights: true
      MonitoringInterval: 60
      EnableIAMDatabaseAuthentication: true
      DeletionProtection: true
      Tags:
        - Key: Environment
          Value: prod
        - Key: Service
          Value: billing

Outputs:
  DbEndpoint:
    Value: !GetAtt DbInstance.Endpoint.Address
  DbPort:
    Value: !GetAtt DbInstance.Endpoint.Port
```

### Instance class families

| Family | Good fit | Trade-off |
| --- | --- | --- |
| db.t4g / db.t3 | small dev/test or bursty apps | low cost but not ideal for steady high throughput |
| db.m6i / db.m7g | balanced general-purpose workloads | good default family for many applications |
| db.r6g / db.r7g | memory-heavy relational workloads | higher cost but strong cache and join performance |
| db.x2g / db.x2iedn | very large memory needs | specialized sizing and cost profile |
| db.c6i (engine dependent) | compute-heavy workloads | less memory per vCPU than memory families |

### Storage types

| Storage type | When to use | Notes |
| --- | --- | --- |
| gp3 | default for most new deployments | predictable baseline performance, decoupled size and throughput knobs |
| io1 / io2 | latency-sensitive or high-IOPS workloads | provisioned IOPS and usually higher durability expectations |
| magnetic | legacy only | older option with lower performance and rarely the best choice for new systems |

### Multi-AZ deployments

- Use Multi-AZ when the database is business-critical and you want a synchronous standby in another Availability Zone.
- Multi-AZ is for **availability**, not primary read scaling.
- Expect some additional write latency because commits are synchronously replicated to the standby.
- Failover re-points the managed endpoint; applications should reconnect using DNS rather than pinned IPs.
- Run game-day tests so application connection pools are validated during controlled failover events.

```bash
aws rds modify-db-instance       --db-instance-identifier prod-postgres-01       --multi-az       --apply-immediately
```

### Read replicas

- Use read replicas to offload read traffic, reporting jobs, BI extracts, or regional read latency needs.
- Replication is asynchronous, so monitor lag if stale reads could cause customer impact.
- Reader applications should use dedicated reader endpoints or service-discovery patterns instead of manually rotating hostnames.
- Promoting a replica is a recovery or migration technique, but it changes replication relationships and requires post-cutover validation.

```bash
aws rds create-db-instance-read-replica       --db-instance-identifier prod-postgres-01-rr1       --source-db-instance-identifier prod-postgres-01       --db-instance-class db.r6g.large       --publicly-accessible false
```

### RDS high-availability and read-scale flow

```mermaid
flowchart LR
    App[Application tier] --> Proxy[RDS Proxy optional]
    Proxy --> Primary[(Primary DB)]
    Primary --> Standby[(Synchronous standby)]
    Primary --> RR1[(Read replica 1)]
    Primary --> RR2[(Read replica 2)]
    RR1 --> Reports[Reporting jobs]
    RR2 --> Analytics[Read-only APIs]

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class App,Proxy,Primary,Standby,RR1,RR2,Reports,Analytics aws;
```

### Connection strings for common languages and tools

#### PostgreSQL URI

```text
postgresql://masteruser:REPLACE_PASSWORD@prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com:5432/appdb?sslmode=require
```

#### MySQL URI

```text
mysql://masteruser:REPLACE_PASSWORD@prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com:3306/appdb?ssl-mode=REQUIRED
```

#### Python (psycopg)

```python
import psycopg

conn = psycopg.connect(
    host="prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com",
    port=5432,
    dbname="appdb",
    user="masteruser",
    password="REPLACE_PASSWORD",
    sslmode="require",
)
```

#### Python (PyMySQL)

```python
import pymysql

conn = pymysql.connect(
    host="prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com",
    user="masteruser",
    password="REPLACE_PASSWORD",
    database="appdb",
    port=3306,
    ssl={"ssl": {}}
)
```

#### Node.js (pg)

```javascript
const { Client } = require('pg');

const client = new Client({
  host: 'prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com',
  port: 5432,
  database: 'appdb',
  user: 'masteruser',
  password: process.env.DB_PASSWORD,
  ssl: { rejectUnauthorized: true }
});
```

#### Node.js (mysql2)

```javascript
const mysql = require('mysql2/promise');

const conn = await mysql.createConnection({
  host: 'prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com',
  user: 'masteruser',
  password: process.env.DB_PASSWORD,
  database: 'appdb',
  port: 3306,
  ssl: {}
});
```

#### Java JDBC (PostgreSQL)

```java
String url = "jdbc:postgresql://prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com:5432/appdb?sslmode=require";
Properties props = new Properties();
props.setProperty("user", "masteruser");
props.setProperty("password", System.getenv("DB_PASSWORD"));
Connection conn = DriverManager.getConnection(url, props);
```

#### Java JDBC (MySQL)

```java
String url = "jdbc:mysql://prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com:3306/appdb?useSSL=true&requireSSL=true";
Connection conn = DriverManager.getConnection(url, "masteruser", System.getenv("DB_PASSWORD"));
```

#### Go (database/sql + pgx)

```go
connString := "host=prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com port=5432 dbname=appdb user=masteruser password=REPLACE_PASSWORD sslmode=require"
db, err := sql.Open("pgx", connString)
if err != nil {
    panic(err)
}
```

#### Go (database/sql + mysql)

```go
dsn := "masteruser:REPLACE_PASSWORD@tcp(prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com:3306)/appdb?tls=true"
db, err := sql.Open("mysql", dsn)
if err != nil {
    panic(err)
}
```

#### C# (.NET Npgsql)

```csharp
var cs = "Host=prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com;Port=5432;Database=appdb;Username=masteruser;Password=" + Environment.GetEnvironmentVariable("DB_PASSWORD") + ";SSL Mode=Require;Trust Server Certificate=false";
await using var conn = new NpgsqlConnection(cs);
await conn.OpenAsync();
```

#### C# (.NET MySqlConnector)

```csharp
var cs = "Server=prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com;Port=3306;Database=appdb;User ID=masteruser;Password=" + Environment.GetEnvironmentVariable("DB_PASSWORD") + ";SslMode=Required";
await using var conn = new MySqlConnection(cs);
await conn.OpenAsync();
```

#### PHP PDO PostgreSQL

```php
$dsn = 'pgsql:host=prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com;port=5432;dbname=appdb;sslmode=require';
$pdo = new PDO($dsn, 'masteruser', getenv('DB_PASSWORD'));
```

#### Ruby pg gem

```ruby
conn = PG.connect(
  host: 'prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com',
  port: 5432,
  dbname: 'appdb',
  user: 'masteruser',
  password: ENV['DB_PASSWORD'],
  sslmode: 'require'
)
```

#### psql command line

```bash
export PGPASSWORD='REPLACE_PASSWORD'
psql "host=prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com port=5432 dbname=appdb user=masteruser sslmode=require"
```

#### mysql command line

```bash
mysql           --host=prod-mysql-01.abcdefghijk.us-east-1.rds.amazonaws.com           --port=3306           --user=masteruser           --password           --ssl-mode=REQUIRED appdb
```

#### PowerShell SqlClient example

```powershell
$connectionString = "Server=tcp:prod-sqlserver-01.abcdefghijk.us-east-1.rds.amazonaws.com,1433;Database=AppDb;User ID=masteruser;Password=$env:DB_PASSWORD;Encrypt=True;TrustServerCertificate=False;"
$connection = New-Object System.Data.SqlClient.SqlConnection $connectionString
$connection.Open()
```

### Real-world RDS rollout scenarios

#### Internal payroll application

- Use Multi-AZ PostgreSQL in private subnets, Secrets Manager rotation, and Session Manager port forwarding for DBA access.
- Define explicit source security groups instead of broad CIDR ranges whenever possible.
- Document rollback, maintenance, and failover test procedures before production onboarding.
- Validate application connection pools, TLS settings, and secret rotation behavior during integration testing.

#### SaaS billing API

- Use Aurora or RDS PostgreSQL with RDS Proxy, reader replicas for reporting, and SG-based access from ECS tasks.
- Define explicit source security groups instead of broad CIDR ranges whenever possible.
- Document rollback, maintenance, and failover test procedures before production onboarding.
- Validate application connection pools, TLS settings, and secret rotation behavior during integration testing.

#### Legacy vendor app

- Use RDS SQL Server or Oracle with a dedicated admin subnet, Active Directory integration where required, and backup validation.
- Define explicit source security groups instead of broad CIDR ranges whenever possible.
- Document rollback, maintenance, and failover test procedures before production onboarding.
- Validate application connection pools, TLS settings, and secret rotation behavior during integration testing.

#### Cross-account analytics consumer

- Publish read replicas or shared access through a peered/TGW-connected analytics VPC, while keeping the primary isolated.
- Define explicit source security groups instead of broad CIDR ranges whenever possible.
- Document rollback, maintenance, and failover test procedures before production onboarding.
- Validate application connection pools, TLS settings, and secret rotation behavior during integration testing.

#### Migration landing zone

- Restore from snapshot into a private subnet, validate schema and performance, then cut over app DNS or connection configuration.
- Define explicit source security groups instead of broad CIDR ranges whenever possible.
- Document rollback, maintenance, and failover test procedures before production onboarding.
- Validate application connection pools, TLS settings, and secret rotation behavior during integration testing.

### Post-deployment validation checklist

1. Confirm the DB endpoint resolves to a private IP from application subnets.
2. Verify the application security group can reach the DB port and that no other sources are allowed.
3. Connect with the required TLS mode and confirm certificate validation behavior.
4. Run a basic read/write smoke test with the production connection string format.
5. Check CloudWatch metrics, Enhanced Monitoring, and Performance Insights for the new instance.
6. Test password retrieval or secret rotation if Secrets Manager is in use.
7. Test admin access path through SSM or bastion, not by temporarily enabling public access.
8. Record the endpoint, port, subnet group, SGs, KMS key, backup retention, and failover posture in operations documentation.

## 3. Private Access via VPC

### Reference VPC Architecture

```mermaid
flowchart TB
    Internet[Internet users] --> ALB[Public ALB]
    ALB --> App1[App subnet AZ-A]
    ALB --> App2[App subnet AZ-B]
    App1 --> DB1[(RDS subnet AZ-A)]
    App2 --> DB2[(RDS subnet AZ-B)]
    Admin[Admin via VPN or SSM] --> Bastion[Optional bastion]
    Bastion --> DB1
    Bastion --> DB2
    App1 --> VPCE[Interface and gateway endpoints]
    App2 --> VPCE

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class Internet,ALB,App1,App2,DB1,DB2,Admin,Bastion,VPCE aws;
```

### Core principles

- Put stateful database services in **private subnets** with no direct route from the internet.
- Give application tiers and admin paths **just enough network reachability** using security groups and routing.
- Use **DB subnet groups** to anchor RDS, Aurora, DocumentDB, Neptune, and related managed services to the correct subnets.
- Keep **public access disabled** unless there is a compelling, approved exception with compensating controls.
- Treat DNS, route tables, and NACLs as first-class parts of private connectivity design; security groups alone are not enough.

### RDS in private subnets

1. Create private subnets in at least two Availability Zones.
2. Associate those subnets with route tables that do not point directly to an internet gateway.
3. If outbound package updates are required for helper EC2 instances, use NAT gateways only for those application or admin subnets, not because the DB needs internet access.
4. Build a DB subnet group from the private subnets.
5. Launch the RDS instance with **PubliclyAccessible=false**.
6. Allow only application SGs, bastion SGs, or specific hybrid CIDRs through the DB security group.
7. Use private hosted zones or standard VPC DNS support so clients resolve the endpoint correctly.

### DB subnet groups

- A DB subnet group is a named collection of subnets that RDS or Aurora uses for placement and failover.
- Use at least two subnets in different AZs for production.
- Do not mix public and private subnets in the same DB subnet group unless there is a very specific and approved reason.
- Subnet group design should reflect the same blast radius and compliance boundary as the application consuming the database.
- When shared VPC or multi-account patterns are used, document subnet ownership clearly so route and ACL changes do not break availability.

### Security groups for database access

| Source | Protocol/port | Allowed? | Reason |
| --- | --- | --- | --- |
| Application SG | TCP 5432 to PostgreSQL | Yes | Main app traffic path |
| Bastion SG | TCP 5432 to PostgreSQL | Yes, only if bastion is used | Admin connectivity path |
| 0.0.0.0/0 | TCP 5432 | No | Overly broad and defeats private-access posture |
| Corporate CIDR over VPN | TCP 5432 | Conditional | Useful for hybrid admin or BI access when routed privately |
| Monitoring SG | TCP DB port | Conditional | Only if self-managed collectors or migration tools require direct DB access |

- Prefer **SG-to-SG references** over CIDR ranges for AWS-native workloads.
- Use separate security groups for application traffic and administrative traffic when operationally practical.
- Keep egress restrictive where organizational policy requires it, but remember return traffic and service integrations may need outbound permissions.

### Network ACL considerations

- NACLs are stateless, so you must allow both inbound request ports and outbound ephemeral response ports.
- In tightly controlled environments, document the client ephemeral port range used by your operating systems and enforce it consistently.
- If you experience intermittent timeouts despite correct SG rules, NACLs are a common root cause.
- Use NACLs as a coarse subnet guardrail, not the primary database access control layer.

```text
Example NACL pattern for PostgreSQL in a private DB subnet:
- Inbound allow TCP 5432 from application subnet CIDRs
- Outbound allow TCP 1024-65535 to application subnet CIDRs
- Inbound allow ephemeral return range from load or admin subnet if required by tooling
- Deny everything else by default rule order
```

### Routing and DNS details

- RDS endpoints are DNS names; applications should never hard-code resolved IP addresses.
- Ensure VPC DNS hostnames and DNS resolution are enabled.
- If on-premises clients must resolve private RDS names, use Route 53 Resolver inbound endpoints and forwarding rules where needed.
- For multi-VPC environments, verify that the chosen connectivity method preserves DNS reachability and not just IP routing.
- In split-horizon DNS designs, test from each network segment because the same hostname can resolve differently depending on the resolver path.

### Terraform snippet: VPC-side controls for a private database tier

```hcl
resource "aws_route_table" "private" {
  vpc_id = var.vpc_id
  tags = {
    Name = "private-app-rt"
  }
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = var.vpc_id
}

resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = var.vpc_id

  ingress {
    protocol        = "tcp"
    from_port       = 5432
    to_port         = 5432
    security_groups = [aws_security_group.app.id]
  }
}
```

### Private access design review questions

- Which subnets host the database and are they private in every AZ?
- Which application components actually require direct database connectivity?
- Can admin access be moved to Session Manager port forwarding instead of SSH ingress?
- Are route tables, TGW attachments, and resolver rules aligned for every source network?
- Is the database endpoint private-only and protected by SG allowlists?
- How will failover affect DNS, client reconnect logic, and monitoring?
- What evidence proves that no public path is required for routine operations?

## 4. VPC Endpoints for AWS Database Services

### Private endpoint architecture

```mermaid
flowchart LR
    App[App subnet] --> IF1[Interface endpoint: RDS API]
    App --> IF2[Interface endpoint: Secrets Manager]
    App --> IF3[Interface endpoint: KMS]
    App --> GW1[Gateway endpoint: DynamoDB]
    Hybrid[On-prem via VPN/DX] --> IF4[Interface endpoint: DynamoDB]
    Admin[Shared services VPC] --> IF1
    Cache[ElastiCache nodes] --> App

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class App,IF1,IF2,IF3,GW1,Hybrid,IF4,Admin,Cache aws;
```

### Why endpoints matter

- VPC endpoints let workloads reach AWS services without traversing the public internet.
- They simplify private API access from workloads that do not have internet egress.
- They make it easier to enforce tighter egress controls, reduce NAT dependency, and attach endpoint policies.
- They are especially helpful in regulated environments that require clear proof that service API calls stay on the AWS network.

### Interface endpoints for the RDS API

- An interface endpoint for **Amazon RDS API** is used for **control-plane** operations such as creating, modifying, or describing RDS resources.
- It is **not** required for application traffic to the actual DB instance endpoint.
- Use it when automation in a private subnet needs to call the RDS API without NAT or internet egress.
- Attach a security group to the interface endpoint ENIs and allow HTTPS 443 from the source subnets or SGs.

```bash
aws ec2 create-vpc-endpoint       --vpc-id vpc-1234567890abcdef0       --service-name com.amazonaws.us-east-1.rds       --vpc-endpoint-type Interface       --subnet-ids subnet-aaa111 subnet-bbb222       --security-group-ids sg-vpce123456       --private-dns-enabled
```

### Gateway and interface endpoints for DynamoDB

- DynamoDB supports **gateway endpoints** and also **interface endpoints (AWS PrivateLink)** in supported Regions.
- Gateway endpoints are often the simplest option for workloads running inside the same VPC because they are route-table based and cost-effective.
- Interface endpoints are useful when you need private connectivity from on-premises, another VPC, or more granular endpoint policy and DNS behavior.
- You can run gateway and interface endpoints together in the same VPC if needed.

```bash
aws ec2 create-vpc-endpoint       --vpc-id vpc-1234567890abcdef0       --service-name com.amazonaws.us-east-1.dynamodb       --vpc-endpoint-type Gateway       --route-table-ids rtb-aaa111 rtb-bbb222

aws ec2 create-vpc-endpoint       --vpc-id vpc-1234567890abcdef0       --service-name com.amazonaws.us-east-1.dynamodb       --vpc-endpoint-type Interface       --subnet-ids subnet-aaa111 subnet-bbb222       --security-group-ids sg-vpce123456       --private-dns-enabled
```

### PrivateLink and ElastiCache

- ElastiCache **data-plane** connectivity is usually direct private access to cache node or cluster endpoints inside your VPC.
- AWS also documents **interface VPC endpoints for the ElastiCache API** (control plane) through AWS PrivateLink.
- Do not confuse API endpoint private access with application traffic to cache nodes; application traffic still targets the cache endpoints themselves.
- When using ElastiCache serverless or advanced multi-VPC designs, validate the exact endpoint behavior for your engine and Region before rollout.

### DNS resolution in the VPC

- Enable **Private DNS** for interface endpoints when you want the standard AWS service hostname to resolve to the private endpoint in the VPC.
- Test DNS from each subnet or hybrid source because split routing or custom DNS forwarders can change results.
- For on-premises workloads, use Route 53 Resolver inbound endpoints or equivalent DNS forwarding so PrivateLink names resolve correctly.
- Document whether applications will call the standard AWS hostname or the endpoint-specific DNS name.

### Endpoint policy examples

```json
{
  "Statement": [
    {
      "Sid": "AllowDescribeOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "rds:DescribeDBSubnetGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

### When to use which endpoint type

| Use case | Recommended endpoint | Why |
| --- | --- | --- |
| Private automation to manage RDS resources | RDS API interface endpoint | Keeps control-plane calls off NAT/internet |
| EC2 app in same VPC calling DynamoDB | DynamoDB gateway endpoint | Simple routing and cost-effective |
| On-prem app calling DynamoDB privately | DynamoDB interface endpoint | PrivateLink works over VPN or Direct Connect |
| Private subnet app retrieving DB secrets | Secrets Manager interface endpoint | Removes internet egress requirement for secret reads |
| Private subnet app using KMS for envelope encryption | KMS interface endpoint | Keeps encryption API traffic private |

## 5. Connecting from On-Premises

### Hybrid database connectivity topology

```mermaid
flowchart LR
    Corp[Corporate network] --> VPN[Site-to-Site VPN]
    Corp --> DX[Direct Connect]
    VPN --> TGW[Transit Gateway]
    DX --> TGW
    TGW --> Shared[Shared services VPC]
    TGW --> AppVPC[Application VPC]
    TGW --> DbVPC[Database VPC]
    Shared --> DNS[Route 53 Resolver endpoints]
    AppVPC --> DbVPC
    Corp --> DNS
    DbVPC --> RDS[(Private RDS/Aurora)]
    DbVPC --> DDB[PrivateLink or gateway-enabled DynamoDB path]

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class Corp,VPN,DX,TGW,Shared,AppVPC,DbVPC,DNS,RDS,DDB aws;
```

### AWS Site-to-Site VPN

- Site-to-Site VPN is often the fastest path to private hybrid database access for pilots, branch offices, or backup connectivity.
- Advertise only the routes that on-premises clients actually need to reach app or database subnets.
- If dynamic routing is used, validate propagated routes in the VPC route tables and TGW route tables.
- Plan for MTU and throughput characteristics; long-running replication or bulk exports may need testing to avoid fragmentation or throughput surprises.

### AWS Direct Connect

- Direct Connect is the preferred option for stable, high-throughput, low-variance hybrid data traffic.
- Use private virtual interfaces to reach VPCs over Direct Connect Gateway and Transit Gateway where appropriate.
- For critical database administration or bulk ETL, many organizations pair Direct Connect with VPN as encrypted backup connectivity.
- Even with DX, keep security groups narrow and avoid treating the corporate network as fully trusted.

### Transit Gateway for multi-VPC database access

- Transit Gateway centralizes routing between application VPCs, shared services VPCs, inspection VPCs, and database VPCs.
- It reduces peering sprawl when many workloads need controlled access to a shared data layer.
- Use separate route tables to segment which spokes can reach which database networks.
- Add appliance or inspection VPCs only if latency and failure domains are understood, because extra hops can complicate troubleshooting.

### DNS resolution for private RDS endpoints

1. Ensure the on-premises DNS path can resolve AWS private hostnames or forward matching zones to Route 53 Resolver inbound endpoints.
2. Create inbound resolver endpoints in a shared services or networking VPC.
3. Update on-prem DNS conditional forwarders for the relevant private zones or AWS service zones.
4. Test name resolution and compare the answer from on-premises versus EC2 in the target VPC.
5. If custom private hosted zones are used for database aliases, associate them with the correct VPCs and document ownership.

```bash
aws route53resolver create-resolver-endpoint       --creator-request-id corp-dns-inbound-01       --name corp-inbound-r53       --direction INBOUND       --security-group-ids sg-resolver123456       --ip-addresses SubnetId=subnet-aaa111,Ip=10.0.10.10 SubnetId=subnet-bbb222,Ip=10.0.20.10
```

### Hybrid design patterns

#### Corporate BI tool to Redshift

- Use Direct Connect + TGW + private Redshift endpoint + corporate DNS forwarders.
- Validate both routing and DNS before production cutover.
- Record latency expectations and acceptable failover behavior for each path.

#### On-prem batch jobs to RDS

- Use VPN or DX, allow only job server CIDRs or SG-connected proxies, and schedule maintenance-aware windows.
- Validate both routing and DNS before production cutover.
- Record latency expectations and acceptable failover behavior for each path.

#### Shared services DBA access

- Use SSM-managed jump hosts in a shared services VPC and route privately through TGW to database VPCs.
- Validate both routing and DNS before production cutover.
- Record latency expectations and acceptable failover behavior for each path.

#### Hybrid API to DynamoDB

- Use DynamoDB interface endpoints when private HTTPS access from on-premises is required.
- Validate both routing and DNS before production cutover.
- Record latency expectations and acceptable failover behavior for each path.

#### Disaster recovery control plane

- Pre-stage routing, resolver rules, and failover runbooks so hybrid traffic can switch Regions predictably.
- Validate both routing and DNS before production cutover.
- Record latency expectations and acceptable failover behavior for each path.

### Hybrid connectivity validation

- Run `nslookup`, `dig`, or `Resolve-DnsName` from on-premises and from AWS to compare results.
- Test TCP connectivity with `nc`, `telnet`, `Test-NetConnection`, or application-native clients.
- Capture VPC Flow Logs during validation so deny events can be analyzed later if needed.
- Review BGP advertisements and route propagation when traffic reaches the VPC but not the DB subnet.

## 6. Bastion Host / Systems Manager Access

### Administrative access patterns

```mermaid
flowchart TD
    AdminLaptop[Administrator laptop] --> SSM[AWS Systems Manager Session Manager]
    AdminLaptop --> SSH[SSH to bastion]
    SSM --> ManagedEC2[Managed EC2 in private subnet]
    SSH --> Bastion[Bastion host]
    ManagedEC2 --> DB[(Private database)]
    Bastion --> DB

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class AdminLaptop,SSM,SSH,ManagedEC2,Bastion,DB aws;
```

### EC2 bastion host setup for DB access

1. Launch a small hardened EC2 instance in a dedicated admin subnet or private subnet with controlled ingress.
2. Attach an IAM role if the host will also fetch secrets, write logs, or integrate with Session Manager.
3. Install database clients such as `psql`, `mysql`, or SQL Server tools.
4. Restrict inbound SSH to a small trusted corporate CIDR or remove SSH entirely if Session Manager will be used.
5. Allow egress from the bastion SG to the DB SG on the required database port.
6. Enable OS patching, centralized logging, and MFA-backed access processes.

### AWS Systems Manager Session Manager (bastion-less)

- Session Manager is often the best modern answer because it removes inbound SSH requirements from the VPC.
- Managed instances need the SSM agent, an IAM role, and network reachability to SSM, EC2 Messages, and SSMMessages endpoints or NAT.
- In locked-down networks, create interface VPC endpoints for `ssm`, `ec2messages`, and `ssmmessages` so management stays private.
- You can use Session Manager for interactive shell sessions, SSH over SSM, or port forwarding to a remote database endpoint.

```bash
aws ssm start-session       --target i-0123456789abcdef0
```

### SSH tunneling through a bastion

```bash
ssh -i ~/.ssh/prod-bastion.pem       -L 5432:prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com:5432       ec2-user@bastion.corp.example.com

psql "host=127.0.0.1 port=5432 dbname=appdb user=masteruser sslmode=require"
```

### Session Manager port forwarding to a database

```bash
aws ssm start-session       --target i-0123456789abcdef0       --document-name AWS-StartPortForwardingSessionToRemoteHost       --parameters '{
    "host": ["prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com"],
    "portNumber": ["5432"],
    "localPortNumber": ["15432"]
  }'

psql "host=127.0.0.1 port=15432 dbname=appdb user=masteruser sslmode=require"
```

### Hardening recommendations for admin access

- Prefer Session Manager over public bastions whenever possible.
- If a bastion is required, avoid database client credentials stored on disk; fetch them just-in-time from Secrets Manager.
- Log session activity and integrate with SIEM tooling.
- Use separate admin hosts for production and non-production to reduce blast radius.
- Require MFA-backed federated access to the AWS account and SSM session permissions.

### Real-world administrative access scenarios

#### Emergency schema fix

- Use Session Manager port forwarding to a controlled EC2 instance in the same private subnet family as the application tier.
- Keep the access window short and revoke elevated privileges immediately after the task.

#### Vendor DBA access

- Provide a time-bound federated role, SSM access to a managed jump host, and a rotated secret scoped only to the required database.
- Keep the access window short and revoke elevated privileges immediately after the task.

#### Blue/green cutover

- Use bastion or SSM to compare old and new instances, run checksums, and verify replication lag before promoting traffic.
- Keep the access window short and revoke elevated privileges immediately after the task.

#### Audit review

- Use recorded sessions, CloudTrail, and database audit logs to show that admin access remained private and attributable.
- Keep the access window short and revoke elevated privileges immediately after the task.

## 7. Aurora Serverless & Global Database

### Aurora topology reference

```mermaid
flowchart LR
    App1[App in Region A] --> Writer[(Aurora writer endpoint)]
    App2[App in Region A] --> Reader[(Aurora reader endpoint)]
    Writer --> Storage[(Distributed Aurora storage)]
    Storage --> Reader
    Writer --> Global[(Global Database secondary region)]
    Serverless[Serverless v2 capacity] --> Writer
    DataAPI[Data API] --> Writer

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class App1,App2,Writer,Reader,Storage,Global,Serverless,DataAPI aws;
```

### Aurora Serverless v2 setup

- Aurora Serverless v2 automatically adjusts capacity in small increments based on workload demand.
- It is useful for variable workloads, development environments that still require Aurora features, and some SaaS workloads with uneven demand curves.
- Design private connectivity exactly as you would for provisioned Aurora: private subnets, SGs, endpoint planning, and secret management still matter.

```bash
aws rds create-db-cluster       --db-cluster-identifier prod-aurora-pg       --engine aurora-postgresql       --engine-version 15.4       --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=8       --db-subnet-group-name prod-db-subnets       --vpc-security-group-ids sg-db123456       --storage-encrypted       --manage-master-user-password       --enable-http-endpoint

aws rds create-db-instance       --db-cluster-identifier prod-aurora-pg       --db-instance-identifier prod-aurora-pg-1       --db-instance-class db.serverless       --engine aurora-postgresql
```

### Terraform: Aurora Serverless v2

```hcl
resource "aws_rds_cluster" "aurora" {
  cluster_identifier              = "prod-aurora-pg"
  engine                          = "aurora-postgresql"
  engine_version                  = "15.4"
  db_subnet_group_name            = aws_db_subnet_group.prod.name
  vpc_security_group_ids          = [aws_security_group.db.id]
  manage_master_user_password     = true
  storage_encrypted               = true
  iam_database_authentication_enabled = true
  serverlessv2_scaling_configuration {
    min_capacity = 0.5
    max_capacity = 8
  }
  enable_http_endpoint = true
}

resource "aws_rds_cluster_instance" "aurora_writer" {
  identifier         = "prod-aurora-pg-1"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora.engine
}
```

### Aurora Global Database for multi-region

- Aurora Global Database replicates storage changes from the primary Region to secondary Regions with low lag.
- Use it for cross-region disaster recovery, regional read scaling, or controlled active/passive designs.
- Application failover planning must include DNS, secret replication, SG parity, and client reconnect behavior.
- Do not treat Global Database as a substitute for application-level resiliency testing; run actual cross-region failover exercises.

```bash
aws rds create-global-cluster       --global-cluster-identifier global-orders-db       --source-db-cluster-identifier arn:aws:rds:us-east-1:111122223333:cluster:prod-aurora-pg
```

### Data API for serverless access

- The Data API provides HTTPS-based access for supported Aurora serverless-compatible patterns.
- It can simplify Lambda or low-connection-count environments where native client management is inconvenient.
- Because it is an API-based model, authorization, throttling, and network paths differ from direct TCP connections.
- Keep Secrets Manager and IAM permissions tightly scoped because the Data API often becomes part of automation workflows.

```bash
aws rds-data execute-statement       --resource-arn arn:aws:rds:us-east-1:111122223333:cluster:prod-aurora-pg       --secret-arn arn:aws:secretsmanager:us-east-1:111122223333:secret:prod-aurora       --database appdb       --sql "select now();"
```

### Aurora connection management

- Use the **cluster endpoint** for writes and the **reader endpoint** for read-only or read-preferred traffic.
- Use **RDS Proxy** when the client environment creates connection storms, such as Lambda or bursty container scaling.
- For serverless capacity, test scale-up reaction time under realistic concurrency and transaction duration.
- Monitor DatabaseConnections, ServerlessDatabaseCapacity, failover events, and replica lag in CloudWatch.

## 8. DynamoDB Access Patterns

### Table design and access

- Start with access patterns first: item lookups, range queries, adjacency lists, and aggregate projections.
- Design partition keys to distribute load; design sort keys to support ordered retrieval and narrow-range queries.
- Use GSIs and LSIs intentionally because every index changes cost, throughput, and write amplification.
- Model for application questions, not for a relational normalization ideal.

| Pattern | Recommended modeling idea | Private access note |
| --- | --- | --- |
| User profile by ID | PK = USER#<id> | Works well through gateway or interface VPC endpoints |
| Orders by customer over time | PK = CUSTOMER#<id>, SK = ORDER#<timestamp> | On-premises access may favor interface endpoints |
| Many-to-many lookup | Adjacency item collections or duplicated relationship items | IAM policy should restrict table and index use |
| Low-latency hot key reads | Add DAX for read acceleration | Keep DAX nodes in private subnets and SG allowlists |

### VPC endpoints for DynamoDB

- Use a **gateway endpoint** when workloads are in the same VPC and you want a simple private route-table-based solution.
- Use an **interface endpoint** through AWS PrivateLink when you need private connectivity from on-premises, across certain network boundaries, or more explicit DNS control.
- Apply endpoint policies to reduce misuse from broad IAM permissions.
- Test SDK behavior, DNS resolution, and failover from each compute environment using the table APIs you actually depend on.

### IAM-based access control

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:UpdateItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:111122223333:table/Orders",
        "arn:aws:dynamodb:us-east-1:111122223333:table/Orders/index/*"
      ],
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:Attributes": ["PK", "SK", "status", "total"]
        }
      }
    }
  ]
}
```

### DAX (DynamoDB Accelerator) setup

- DAX is an in-memory cache for DynamoDB that improves read latency for suitable workloads.
- Deploy DAX nodes in private subnets and control access with dedicated SG rules from the application tier.
- DAX does not change table design fundamentals; a poor partition key design will still surface scalability issues.
- Use it for read-heavy, eventually consistent access patterns where microsecond-level caching is useful.

```bash
aws dax create-cluster       --cluster-name orders-dax       --node-type dax.r5.large       --replication-factor 3       --subnet-group-name prod-dax-subnets       --security-group-ids sg-dax123456       --iam-role-arn arn:aws:iam::111122223333:role/dax-service-role
```

### Global Tables

- Global Tables replicate data across Regions for low-latency reads and regional resilience.
- Coordinate IAM policies, endpoint strategy, and operational playbooks across Regions.
- Design for conflict resolution semantics and application idempotency.
- Global Tables help with private regional access, but they do not remove the need for DNS, endpoint, or network planning in hybrid scenarios.

### DynamoDB real-world patterns

#### Serverless checkout

- Lambda in private subnets uses a DynamoDB VPC endpoint and IAM role with table- and index-scoped permissions.
- Test retry behavior, throttling, and region-failover implications with production-like traffic.

#### IoT ingestion

- A fleet writes device state to DynamoDB, then analytics jobs read projections through tightly scoped roles.
- Test retry behavior, throttling, and region-failover implications with production-like traffic.

#### Hybrid inventory sync

- On-prem ERP connects through Direct Connect and a DynamoDB interface endpoint for private HTTPS calls.
- Test retry behavior, throttling, and region-failover implications with production-like traffic.

#### Read acceleration

- API tier reads session or catalog data through DAX while writes go directly to DynamoDB.
- Test retry behavior, throttling, and region-failover implications with production-like traffic.

## 9. Database Security

### Security architecture reference

```mermaid
flowchart TD
    IAM[IAM roles and policies] --> App[Application tier]
    Secrets[Secrets Manager rotation] --> App
    KMS[KMS encryption keys] --> RDS[(RDS / Aurora)]
    KMS --> DDB[DynamoDB]
    TLS[TLS in transit] --> App
    App --> Proxy[RDS Proxy]
    Proxy --> RDS
    Audit[Database Activity Streams / logs] --> SIEM[Security monitoring]
    SG[Security groups and NACLs] --> RDS
    SG --> Proxy

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E;
    class IAM,App,Secrets,KMS,RDS,DDB,TLS,Proxy,Audit,SIEM,SG aws;
```

### IAM database authentication

- IAM database authentication can replace static passwords for supported RDS and Aurora MySQL/PostgreSQL patterns.
- Clients request a short-lived authentication token signed with IAM permissions.
- This reduces long-lived password sprawl, but it adds token-generation logic and expiration handling to clients or admin workflows.
- Combine it with TLS because IAM DB auth requires encrypted connections in supported engines.

```bash
aws rds generate-db-auth-token       --hostname prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com       --port 5432       --region us-east-1       --username iam_db_user
```

### Secrets Manager for credentials rotation

- Use Secrets Manager when you want managed storage, rotation, versioning, and auditability for database credentials.
- Prefer applications that fetch secrets at startup and refresh safely instead of embedding passwords in AMIs, images, or IaC outputs.
- Private subnet applications should also have VPC interface endpoints for Secrets Manager if internet egress is not allowed.
- Test rotation with the real client libraries because some connection pools cache credentials aggressively.

```bash
aws secretsmanager create-secret       --name prod/postgres/master       --description "Primary PostgreSQL credentials"       --secret-string '{"username":"masteruser","password":"REPLACE_PASSWORD"}'
```

### KMS encryption at rest

- Enable storage encryption for RDS, Aurora, Redshift, DocumentDB, and other supported services.
- Use customer-managed keys when key policy control, rotation alignment, or cross-account usage requirements exist.
- Ensure key policies and grants allow the service and operations teams to use the key without over-broad access.
- Remember that encrypted snapshots, replicas, and restores inherit KMS design implications.

### SSL/TLS in transit

- Require TLS for database connections wherever the engine and client support it.
- Distribute the correct CA bundle and rotate trust stores before certificate authority deadlines.
- Validate server identity when possible instead of merely encrypting without certificate checks.
- Include TLS checks in health tests, not just application connection tests.

### RDS Proxy for connection pooling

- RDS Proxy helps absorb connection storms and pool database connections efficiently.
- It is especially useful for Lambda, bursty ECS services, or fleets that autoscale rapidly.
- Keep the proxy in private subnets and lock its SG so only expected application tiers can connect.
- Monitor borrow latency, connection utilization, and target health to ensure the proxy improves rather than obscures behavior.

```bash
aws rds create-db-proxy       --db-proxy-name prod-postgres-proxy       --engine-family POSTGRESQL       --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:us-east-1:111122223333:secret:prod/postgres/master","IAMAuth":"REQUIRED"}]'       --role-arn arn:aws:iam::111122223333:role/rds-proxy-role       --vpc-subnet-ids subnet-aaa111 subnet-bbb222       --vpc-security-group-ids sg-proxy123456
```

### Database Activity Streams and audit logging

- Use Database Activity Streams, engine-native logs, CloudWatch log exports, or partner tools when near-real-time audit visibility is needed.
- Activity Streams are valuable for regulated workloads because they capture database events with low tampering risk.
- Integrate audit outputs with SIEM pipelines and keep retention aligned with policy.
- Tune noisy events so the audit channel remains actionable and affordable.

| Control | Applies to | Primary risk reduced |
| --- | --- | --- |
| Private subnets | RDS, Aurora, DocumentDB, Neptune, ElastiCache, MemoryDB, Redshift | Direct public exposure |
| Security group allowlists | Most VPC-hosted DB services | Unauthorized east-west access |
| IAM DB auth | Supported RDS/Aurora engines | Static credential sprawl |
| Secrets Manager rotation | RDS/Aurora/DocumentDB/Redshift and apps | Stale credentials and poor secret hygiene |
| KMS encryption | Most AWS database services | Data exposure from compromised storage media or snapshots |
| TLS in transit | Most DB engines and APIs | Credential sniffing or plaintext data exposure |
| RDS Proxy | RDS/Aurora | Connection exhaustion and poor secret handling |
| Activity Streams / audit logs | Supported engines | Undetected misuse or weak forensics |

### Security implementation checklist

- Public access disabled for production relational databases unless a formal exception exists.
- Security groups use SG references or tightly scoped CIDRs only.
- Secrets Manager or IAM DB auth chosen instead of hard-coded credentials.
- KMS key ownership and policy documented.
- TLS requirement documented and tested from every client type.
- Database logging and audit coverage mapped to compliance requirements.
- RDS Proxy evaluated for bursty clients.
- Privileged admin path isolated through SSM or hardened bastions.
- Snapshots, cross-region copies, and restores inherit the right encryption posture.
- Runbooks include secret rotation, certificate rotation, and failover testing.

## 10. Database Troubleshooting

### Troubleshooting sequence

1. Confirm the application is using the correct endpoint, port, and database name.
2. Check whether the issue is DNS resolution, TCP reachability, authentication, TLS negotiation, or server-side load.
3. Validate route tables, Transit Gateway paths, VPN/DX propagation, and NACL behavior for the source subnet or network.
4. Inspect security groups on both the source resource and the database or proxy.
5. Check CloudWatch metrics, Performance Insights, Enhanced Monitoring, engine logs, and VPC Flow Logs.
6. Reproduce with a native client from a known-good source in the same network path.
7. Capture timestamps and request IDs so CloudTrail, Flow Logs, and monitoring data line up for the same failure window.

### Performance Insights

- Performance Insights helps identify wait events, top SQL, DB load, and bottlenecks.
- Use it when connections succeed but the application reports latency, lock contention, or throughput collapse.
- Correlate PI findings with instance class sizing, IOPS, storage throughput, and application connection patterns.

### Enhanced Monitoring

- Enhanced Monitoring exposes near-real-time OS metrics for supported RDS engines.
- It is helpful for diagnosing CPU steal, memory pressure, process behavior, or kernel-level symptoms that standard CloudWatch metrics do not expose clearly.
- Send the monitoring role output to CloudWatch Logs for retention and analysis.

### Common errors and solutions

| Error or symptom | Likely cause | What to do next |
| --- | --- | --- |
| `could not translate host name` | DNS resolution problem | Check VPC DNS settings, Route 53 Resolver rules, and on-prem forwarders |
| `connection timed out` | Routing, SG, NACL, or VPN/TGW issue | Test with `nc`, inspect VPC Flow Logs, verify route tables |
| `password authentication failed` | Bad secret or rotated credential not reloaded | Fetch current secret version and retest |
| `SSL is required` | Client attempted plaintext session | Enable TLS and supply CA bundle if needed |
| `certificate verify failed` | Outdated trust store or CA mismatch | Update CA bundle and trust settings |
| High replica lag | Read replica saturated or network/storage limits | Check workload burst, IOPS, long transactions, replica sizing |
| Frequent failovers | Infrastructure or maintenance events | Review RDS events, maintenance history, AZ stability, and application retries |
| Connection storms | Too many short-lived clients | Add RDS Proxy or pool connections |
| Slow queries after deploy | Bad query plan or missing index | Use Performance Insights, EXPLAIN, and engine logs |
| Intermittent hybrid failures | Route propagation or DNS split-horizon inconsistency | Compare results from multiple source networks and verify TGW policies |
| DynamoDB throttling | Capacity or hot partition issue | Examine access pattern, adaptive capacity, and retry/backoff behavior |
| DAX cache miss surge | Warm-up or key skew | Review TTL, hot-key design, and cluster health |
| Proxy target unhealthy | Backend DB stress or bad auth config | Inspect RDS Proxy metrics and target group configuration |
| `Too many connections` | DB max connections reached | Pool connections, resize instance, or proxy the workload |
| Long failover recovery in app | Client not reconnecting on DNS change | Lower DNS cache TTL in app/runtime and test failover logic |
| Cannot reach DB from bastion | Bastion SG or route mismatch | Test bastion-to-DB port, inspect SG references |
| Secrets rotation broke app | App cached old credentials | Implement refresh logic and rotation-aware retries |
| Data API access denied | IAM or Secret ARN mismatch | Review resource ARN, secret ARN, and policy scope |
| `no pg_hba.conf entry` equivalent managed symptom | Engine/user/network auth mismatch | Validate engine-specific auth rules, TLS mode, and username |
| High storage latency | Under-provisioned storage or burst depletion | Move to gp3/io1/io2 and review throughput settings |

### Troubleshooting commands

```bash
nslookup prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com
dig +short prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com
nc -vz prod-postgres-01.abcdefghijk.us-east-1.rds.amazonaws.com 5432
aws rds describe-db-instances --db-instance-identifier prod-postgres-01
aws rds describe-events --source-type db-instance --source-identifier prod-postgres-01 --duration 60
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=prod-postgres-01 --statistics Average Maximum --period 60 --start-time 2025-01-01T00:00:00Z --end-time 2025-01-01T01:00:00Z
aws logs start-query --log-group-name /aws/rds/instance/prod-postgres-01/postgresql --start-time 1735689600 --end-time 1735693200 --query-string "fields @timestamp, @message | sort @timestamp desc | limit 20"
```

### Scenario-based debugging playbooks

#### App in ECS cannot connect to PostgreSQL

1. Verify task ENI security group egress and DB SG ingress.
2. Check whether the task subnet route table and NACL allow return traffic.
3. Test DNS resolution inside the task or an equivalent debug container.
4. Review RDS Proxy if the app is configured to use one.

#### On-prem tool resolves the endpoint but cannot connect

1. Check VPN/DX routing, TGW propagation, and firewall policies.
2. Validate DNS forwards to Route 53 Resolver inbound endpoints.
3. Use packet captures or firewall logs on the on-prem side if AWS paths look healthy.

#### Database is reachable but slow

1. Inspect wait events in Performance Insights.
2. Check for lock contention, checkpoint spikes, vacuum backlog, or storage saturation.
3. Correlate with deploy, batch, and backup windows.

#### Lambda flood creates connection exhaustion

1. Use RDS Proxy or redesign connection reuse.
2. Cap concurrency where appropriate.
3. Monitor database and proxy connection metrics during peak events.

## 11. Appendix A: Connectivity Validation Checklist

- [ ] 1. Endpoint resolves privately from app subnets
- [ ] 2. Endpoint resolves correctly from hybrid networks
- [ ] 3. DB SG ingress restricted to required sources only
- [ ] 4. NACLs allow stateful return traffic
- [ ] 5. Public access disabled
- [ ] 6. Secrets Manager or IAM auth path works
- [ ] 7. TLS handshake succeeds
- [ ] 8. RDS events reviewed after creation
- [ ] 9. CloudWatch metrics visible
- [ ] 10. Enhanced Monitoring enabled where required
- [ ] 11. Performance Insights enabled where required
- [ ] 12. Backups and retention verified
- [ ] 13. Deletion protection enabled for prod
- [ ] 14. DB subnet group spans at least two AZs
- [ ] 15. Failover runbook documented
- [ ] 16. Read replica lag alarm configured if replicas exist
- [ ] 17. DNS forwarders documented
- [ ] 18. Transit Gateway routes validated
- [ ] 19. VPN or Direct Connect failover tested
- [ ] 20. Session Manager endpoints available in private admin subnets
- [ ] 21. Bastion hardening verified if used
- [ ] 22. Audit logging retention set
- [ ] 23. KMS key policy reviewed
- [ ] 24. Snapshot encryption inheritance verified
- [ ] 25. Client trust stores updated

## 12. Appendix B: Operational Command Library

- `aws rds describe-db-instances`
- `aws rds describe-db-clusters`
- `aws rds describe-db-subnet-groups`
- `aws rds describe-db-proxies`
- `aws rds describe-pending-maintenance-actions`
- `aws ec2 describe-security-groups --group-ids sg-db123456`
- `aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-1234567890abcdef0`
- `aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-1234567890abcdef0`
- `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-1234567890abcdef0`
- `aws route53resolver list-resolver-endpoints`
- `aws route53resolver list-resolver-rules`
- `aws dynamodb describe-table --table-name Orders`
- `aws dax describe-clusters`
- `aws elasticache describe-serverless-caches`
- `aws elasticache describe-replication-groups`
- `aws redshift describe-clusters`
- `aws docdb describe-db-clusters`
- `aws neptune describe-db-clusters`
- `aws cloudwatch describe-alarms --alarm-name-prefix db-`
- `aws logs describe-log-groups --log-group-name-prefix /aws/rds/`

## 13. Appendix C: FAQ

**Q: Do I need an RDS interface endpoint to connect to my database privately?**
A: No. The RDS API interface endpoint is for control-plane API calls. Application traffic still goes to the DB endpoint in the VPC.

**Q: Should I ever enable public access on a production database?**
A: Usually no. Prefer private subnets, VPN/DX, SSM, bastions, or proxies for access.

**Q: Are security groups enough by themselves?**
A: No. Routing, DNS, NACLs, endpoint policies, IAM, and certificate settings also affect private connectivity.

**Q: Can on-premises apps use private DynamoDB access?**
A: Yes, interface endpoints for DynamoDB over PrivateLink are designed for that use case in supported Regions.

**Q: Does Multi-AZ replace backups?**
A: No. Multi-AZ improves availability; backups and restore testing are still required.

**Q: When should I use RDS Proxy?**
A: When connection storms, Lambda bursts, or high client churn threaten database connection limits.

**Q: Why does failover break some apps?**
A: Applications often cache DNS too long or fail to retry cleanly after transient disconnects.

**Q: What is the first thing to check in a timeout?**
A: DNS resolution, SG ingress, route tables, NACLs, and whether the source network can actually reach the DB subnet.

**Q: Is a bastion still necessary if Session Manager works?**
A: Often no. Session Manager can remove inbound SSH and simplify private admin access.

**Q: How do I prove traffic stayed private?**
A: Use architecture evidence, route and endpoint design, VPC Flow Logs, CloudTrail, and absence of public exposure settings.

## 14. Appendix D: Engine-by-Engine Connectivity Reference

| Service | Default port/protocol | Primary endpoint type | Private path guidance | Admin tip |
| --- | --- | --- | --- | --- |
| RDS PostgreSQL | 5432/TCP | Instance endpoint | Private subnets + SG allowlist + optional RDS Proxy | Use `psql` through SSM forwarding |
| RDS MySQL/MariaDB | 3306/TCP | Instance endpoint | Private subnets + SG allowlist | Use TLS and Secrets Manager rotation |
| RDS SQL Server | 1433/TCP | Instance endpoint | Hybrid access often uses DX/VPN + SSM admin host | Validate AD/Kerberos dependencies |
| Aurora | 3306 or 5432/TCP | Cluster + reader endpoints | Private-only clusters are common | Use cluster endpoint for writes |
| DynamoDB | 443/HTTPS | Regional service endpoint | Gateway or interface endpoint | Use IAM + endpoint policies |
| ElastiCache | 6379/6380 or 11211/TCP | Node/cluster endpoint | Private VPC-only data path | Distinguish API endpoint from cache endpoint |
| DocumentDB | 27017/TCP | Cluster endpoint | Private subnets + SG allowlist + TLS | Use Mongo-compatible tooling over private paths |
| Neptune | 8182/HTTPS or WebSocket | Cluster endpoint | Private SG-controlled access | Sign requests with IAM where applicable |
| Redshift | 5439/TCP | Cluster/workgroup endpoint | Private analytics VPC or shared services path | Use temp credentials or SSO |
| MemoryDB | 6379/TCP | Cluster endpoint | Private in-memory primary database path | Test client TLS and ACLs |

### RDS PostgreSQL

- **Default client port:** 5432.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws rds describe-db-instances --db-instance-identifier prod-postgres-01
psql "host=prod-postgres-01.example.rds.amazonaws.com port=5432 dbname=appdb user=app sslmode=require"
```

### RDS MySQL/MariaDB

- **Default client port:** 3306.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws rds describe-db-instances --db-instance-identifier prod-mysql-01
mysql --host=prod-mysql-01.example.rds.amazonaws.com --port=3306 --ssl-mode=REQUIRED -u app -p appdb
```

### RDS SQL Server

- **Default client port:** 1433.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws rds describe-db-instances --db-instance-identifier prod-sql-01
sqlcmd -S prod-sql-01.example.rds.amazonaws.com,1433 -d AppDb -U app -P REPLACE_PASSWORD -N -C
```

### Aurora PostgreSQL

- **Default client port:** 5432.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws rds describe-db-clusters --db-cluster-identifier prod-aurora-pg
psql "host=prod-aurora.cluster-xyz.us-east-1.rds.amazonaws.com port=5432 dbname=appdb user=app sslmode=require"
```

### Aurora MySQL

- **Default client port:** 3306.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws rds describe-db-clusters --db-cluster-identifier prod-aurora-mysql
mysql --host=prod-aurora.cluster-xyz.us-east-1.rds.amazonaws.com --port=3306 --ssl-mode=REQUIRED -u app -p appdb
```

### DynamoDB

- **Default client port:** 443.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws dynamodb describe-table --table-name Orders
aws dynamodb get-item --table-name Orders --key "{"PK":{"S":"ORDER#1001"},"SK":{"S":"META"}}"
```

### ElastiCache Redis/Valkey

- **Default client port:** 6379/6380.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws elasticache describe-replication-groups --replication-group-id orders-cache
redis-cli -h orders-cache.xxxxxx.use1.cache.amazonaws.com -p 6379 --tls
```

### DocumentDB

- **Default client port:** 27017.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws docdb describe-db-clusters --db-cluster-identifier prod-docdb
mongosh "mongodb://app:REPLACE_PASSWORD@prod-docdb.cluster-xyz.us-east-1.docdb.amazonaws.com:27017/appdb?tls=true&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false"
```

### Neptune

- **Default client port:** 8182.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws neptune describe-db-clusters --db-cluster-identifier fraud-graph
curl -s https://neptune.cluster-xyz.us-east-1.neptune.amazonaws.com:8182/status
```

### Redshift

- **Default client port:** 5439.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws redshift describe-clusters --cluster-identifier analytics-prod
psql "host=analytics-prod.xxxxxx.us-east-1.redshift.amazonaws.com port=5439 dbname=analytics user=bi sslmode=require"
```

### MemoryDB

- **Default client port:** 6379.
- **Preferred private pattern:** keep the data path inside the VPC or over approved hybrid connectivity.
- **Identity model:** combine network controls with engine-native auth, IAM, or Secrets Manager as appropriate.
- **DNS note:** test endpoint resolution from every source network, not just from an EC2 instance in the database VPC.
- **Troubleshooting shortcut:** validate DNS first, then TCP connectivity, then authentication, then engine load.
- **Operational caution:** do not treat a successful `describe` API call as proof that the data plane is reachable.

```bash
aws memorydb describe-clusters --cluster-name session-db
redis-cli -h session-db.xxxxxx.memorydb.us-east-1.amazonaws.com -p 6379 --tls
```

## 15. Appendix E: Infrastructure as Code Patterns

### Terraform pattern: SG-only ingress for app-to-DB

- Use this pattern when you want the control in versioned infrastructure code rather than a one-off console action.
- Review dependencies such as subnet IDs, SG IDs, endpoint policies, and IAM roles before applying it.
- Pair the code with post-apply validation so the configuration is proven, not merely deployed.

```hcl
resource "aws_security_group_rule" "app_to_db" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.db.id
  source_security_group_id = aws_security_group.app.id
}
```

### Terraform pattern: DynamoDB gateway endpoint

- Use this pattern when you want the control in versioned infrastructure code rather than a one-off console action.
- Review dependencies such as subnet IDs, SG IDs, endpoint policies, and IAM roles before applying it.
- Pair the code with post-apply validation so the configuration is proven, not merely deployed.

```hcl
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = var.vpc_id
  service_name      = "com.amazonaws.us-east-1.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = var.private_route_table_ids
}
```

### Terraform pattern: RDS API interface endpoint

- Use this pattern when you want the control in versioned infrastructure code rather than a one-off console action.
- Review dependencies such as subnet IDs, SG IDs, endpoint policies, and IAM roles before applying it.
- Pair the code with post-apply validation so the configuration is proven, not merely deployed.

```hcl
resource "aws_vpc_endpoint" "rds_api" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.us-east-1.rds"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpce.id]
}
```

### CloudFormation pattern: Session Manager private endpoints

- Use this pattern when you want the control in versioned infrastructure code rather than a one-off console action.
- Review dependencies such as subnet IDs, SG IDs, endpoint policies, and IAM roles before applying it.
- Pair the code with post-apply validation so the configuration is proven, not merely deployed.

```yaml
Resources:
  SsmEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ServiceName: !Sub com.amazonaws.${AWS::Region}.ssm
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      VpcId: !Ref VpcId
      SubnetIds: [!Ref PrivateSubnetA, !Ref PrivateSubnetB]
      SecurityGroupIds: [!Ref EndpointSecurityGroup]
```

### CloudFormation pattern: Route 53 Resolver inbound endpoint

- Use this pattern when you want the control in versioned infrastructure code rather than a one-off console action.
- Review dependencies such as subnet IDs, SG IDs, endpoint policies, and IAM roles before applying it.
- Pair the code with post-apply validation so the configuration is proven, not merely deployed.

```yaml
Resources:
  InboundResolver:
    Type: AWS::Route53Resolver::ResolverEndpoint
    Properties:
      Direction: INBOUND
      Name: corp-inbound-resolver
      SecurityGroupIds: [!Ref ResolverSecurityGroup]
      IpAddresses:
        - SubnetId: !Ref SharedSubnetA
        - SubnetId: !Ref SharedSubnetB
```

### Terraform pattern: RDS Proxy target group

- Use this pattern when you want the control in versioned infrastructure code rather than a one-off console action.
- Review dependencies such as subnet IDs, SG IDs, endpoint policies, and IAM roles before applying it.
- Pair the code with post-apply validation so the configuration is proven, not merely deployed.

```hcl
resource "aws_db_proxy_default_target_group" "default" {
  db_proxy_name = aws_db_proxy.main.name
  connection_pool_config {
    max_connections_percent      = 80
    max_idle_connections_percent = 40
    connection_borrow_timeout    = 120
  }
}
```

## 16. Appendix F: Real-World Architecture Scenarios

### Scenario 1: Internal finance application with RDS PostgreSQL and zero public access

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 2: Healthcare records platform with Multi-AZ Aurora and mandatory TLS

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 3: Hybrid Oracle migration using Direct Connect and Route 53 Resolver

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 4: Shared services database VPC serving multiple application VPCs over Transit Gateway

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 5: Retail platform using DynamoDB + DAX + private app subnets

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 6: Fraud analytics graph on Neptune accessed from a data science VPC

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 7: Global SaaS using Aurora Global Database with private regional app tiers

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 8: Redis-heavy microservices using ElastiCache with SG-based service isolation

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 9: BI warehouse on Redshift for corporate analysts over private connectivity

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

### Scenario 10: Audit-sensitive environment using Session Manager-only administration

- Primary goal: deliver private data access without creating an exception-based operations model.
- Network pattern: isolate the database tier in private subnets and define every allowed source path explicitly.
- Identity pattern: prefer short-lived or centrally rotated credentials over static passwords in code.
- Observability pattern: alarm on connectivity failure, latency, replica lag, and unusual admin access.
- Risk to watch: DNS drift, SG drift, hybrid route changes, or hidden certificate dependencies.
- Validation step: test from the exact runtime or admin path that production users will use.

## 17. Appendix G: Extended Troubleshooting Matrix

| Failure domain | Example symptom | Best first check | Helpful AWS data source |
| --- | --- | --- | --- |
| DNS | Hostname resolves on EC2 but not on-prem | Resolver forwarding and private hosted zone association | Route 53 Resolver logs / on-prem DNS logs |
| Routing | Timeout from app subnet only | Route tables and TGW propagation | VPC route tables / TGW route tables |
| Security groups | Timeout from one ECS service only | Source SG reference on DB SG | EC2/ECS ENI SGs |
| NACLs | Intermittent timeout with correct SGs | Ephemeral return ports | VPC Flow Logs |
| TLS | Certificate verify failure after CA rotation | Client trust store | Application logs / engine logs |
| Authentication | Password rejected after rotation | Secret version in runtime | Secrets Manager / app logs |
| Engine load | Connection succeeds but queries stall | DB load and wait events | Performance Insights |
| Hybrid DNS | On-prem resolves public answer unexpectedly | Conditional forwarder path | Resolver endpoint config |
| Proxy | RDS Proxy endpoint connects slowly | Target health and borrow latency | Proxy metrics |
| DynamoDB | Private app gets AccessDenied | IAM + endpoint policy + region | CloudTrail / IAM policy simulator |

### Workflow: DNS problems

1. Query from source network and compare answer to expected private result.
2. Verify resolver endpoints, forwarding rules, and VPC DNS settings.
3. Check whether the app caches stale answers.

### Workflow: Network timeouts

1. Use native TCP tests from the actual source host or container.
2. Inspect Flow Logs for ACCEPT or REJECT.
3. Confirm route symmetry for hybrid paths.

### Workflow: Authentication failures

1. Fetch the current secret version.
2. Confirm username mapping and IAM DB auth settings.
3. Retest with a native client to isolate application pooling behavior.

### Workflow: Slow reads or writes

1. Check Performance Insights wait events.
2. Review storage latency and IOPS.
3. Inspect long transactions, locking, and batch job overlap.

### Workflow: Hybrid path instability

1. Compare VPN and DX routes.
2. Review firewall or SD-WAN policies on the on-prem side.
3. Test Route 53 Resolver health and failover paths.

## 18. Appendix H: Pre-Production Review Worksheet

### Networking review

- [ ] Networking 1: Private subnets exist in at least two AZs
- [ ] Networking 2: Route tables reviewed
- [ ] Networking 3: NACLs reviewed
- [ ] Networking 4: VPC endpoint needs identified
- [ ] Networking 5: Hybrid paths tested
- [ ] Networking 6: DNS ownership documented

### Identity review

- [ ] Identity 1: Secret source chosen
- [ ] Identity 2: Rotation tested
- [ ] Identity 3: IAM DB auth evaluated
- [ ] Identity 4: Least privilege IAM validated
- [ ] Identity 5: Admin access approvals documented
- [ ] Identity 6: MFA-backed access enforced

### Availability review

- [ ] Availability 1: Multi-AZ decision documented
- [ ] Availability 2: Replica strategy documented
- [ ] Availability 3: Backup retention approved
- [ ] Availability 4: Restore tested
- [ ] Availability 5: Failover test executed
- [ ] Availability 6: Monitoring thresholds approved

### Security review

- [ ] Security 1: Public access disabled
- [ ] Security 2: TLS validated
- [ ] Security 3: KMS key selected
- [ ] Security 4: Audit logs retained
- [ ] Security 5: RDS Proxy evaluated
- [ ] Security 6: Bastion elimination considered

### Operations review

- [ ] Operations 1: Runbooks written
- [ ] Operations 2: On-call ownership assigned
- [ ] Operations 3: Patch window agreed
- [ ] Operations 4: Maintenance notifications wired
- [ ] Operations 5: Capacity review completed
- [ ] Operations 6: Escalation process documented

## 19. Appendix I: Cutover and Rollback Questions

### Readiness

- [ ] Readiness 1: Have application owners validated the exact production connection strings?
- [ ] Readiness 2: Have all secrets, IAM roles, and certificates been tested in the target environment?
- [ ] Readiness 3: Are maintenance windows, freeze windows, and stakeholder communications aligned?
- [ ] Readiness 4: Has a known-good performance baseline been captured before cutover?
- [ ] Readiness 5: Is there a single owner for go/no-go decisions?

### Network path

- [ ] Network path 1: Has DNS been switched in a non-production rehearsal already?
- [ ] Network path 2: Are SGs, NACLs, routes, and TGW attachments version-controlled and approved?
- [ ] Network path 3: Have hybrid users tested through the same VPN or DX path they will use after cutover?
- [ ] Network path 4: Are interface and gateway endpoints in place where the app depends on private AWS APIs?
- [ ] Network path 5: Is packet rejection telemetry enabled through Flow Logs or equivalent tools?

### Database safety

- [ ] Database safety 1: Is a final snapshot or restorable backup available?
- [ ] Database safety 2: Have replication lag and consistency checks passed?
- [ ] Database safety 3: Are long-running transactions or jobs paused as required?
- [ ] Database safety 4: Have RPO/RTO expectations been restated to stakeholders?
- [ ] Database safety 5: Is the rollback trigger threshold clearly defined?

### Application behavior

- [ ] Application behavior 1: Do connection pools reconnect correctly after DNS change or failover?
- [ ] Application behavior 2: Have health checks been validated end-to-end?
- [ ] Application behavior 3: Are feature flags or read-only modes available if needed?
- [ ] Application behavior 4: Can the application tolerate stale reads or temporary disconnects during transition?
- [ ] Application behavior 5: Are synthetic transactions ready for smoke testing?

### Rollback

- [ ] Rollback 1: Is rollback DNS pre-planned?
- [ ] Rollback 2: Are previous security groups and routes preserved for the rollback window?
- [ ] Rollback 3: Are old credentials still usable for emergency fallback if policy allows?
- [ ] Rollback 4: Has the team practiced rollback decision-making, not just rollback mechanics?
- [ ] Rollback 5: Is post-cutover evidence collection assigned to named owners?
