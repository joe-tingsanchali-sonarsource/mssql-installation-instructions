# SonarQube Server with Microsoft SQL Server Installation Guide

This guide provides comprehensive instructions for installing and configuring SonarQube Server with Microsoft SQL Server as the database backend.

## Table of Contents

- [Prerequisites](#prerequisites)
- [System Requirements](#system-requirements)
- [Database Installation and Configuration](#database-installation-and-configuration)
- [SonarQube Server Installation](#sonarqube-server-installation)
- [Configuration](#configuration)
- [Starting SonarQube](#starting-sonarqube)
- [Post-Installation](#post-installation)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Prerequisites

### Software Requirements

- **Microsoft SQL Server**: Version 2016, 2017, 2019, or 2022
  - Express Edition is supported
  - Both Windows Authentication and SQL Server Authentication are supported
- **Java**: Oracle JRE or OpenJDK version 21 (Java 17 is deprecated)
- **Operating System**: 
  - Linux (x64, AArch64)
  - Windows (x64)
  - macOS (x64, AArch64)

### Hardware Requirements

#### Small-Scale Installation (up to 1M lines of code)
- **RAM**: 4GB minimum
- **CPU**: 2 cores (64-bit system)
- **Disk Space**: 30GB minimum
- **Free Disk Space**: Maintain at least 10% free

#### Large-Scale Installation (up to 50M lines of code)
- **RAM**: 16GB minimum
- **CPU**: 8 cores (64-bit system)
- **Disk Space**: Varies based on codebase size
- **Free Disk Space**: Maintain at least 10% free

> **Important**: For production installations, it is strongly recommended to host the database on a separate machine from the SonarQube Server host, with low latency between both hosts.

---

## Database Installation and Configuration

### Step 1: Install Microsoft SQL Server

1. Download and install Microsoft SQL Server from the [Microsoft website](https://www.microsoft.com/sql-server/)
2. Follow the installation wizard to complete the setup
3. Note the server name and instance name:
   - **Default Instance**: Uses port 1433
   - **Named Instance** (e.g., SQLEXPRESS): Uses dynamic ports or custom port

#### Enable TCP/IP Protocol (Required for SonarQube)

After installation, you must enable TCP/IP connections:

1. Open **SQL Server Configuration Manager**:
   - Press `Win + R` and type: `SQLServerManager16.msc` (for SQL Server 2022)
   - For other versions: 2019 = 15, 2017 = 14, 2016 = 13
   - Or search "SQL Server Configuration Manager" in Start Menu

2. **Enable TCP/IP Protocol**:
   - Expand **SQL Server Network Configuration**
   - Click **Protocols for [INSTANCE_NAME]** (e.g., "Protocols for SQLEXPRESS")
   - Right-click **TCP/IP** and select **Enable**

3. **Configure Static Port 1433** (Optional but Recommended):
   
   By default, SQL Server Express uses dynamic ports. Setting a static port simplifies configuration and eliminates the need for SQL Server Browser.
   
   a. In **SQL Server Configuration Manager**, under **Protocols for SQLEXPRESS**:
      - Right-click **TCP/IP** and select **Properties**
      - Go to the **IP Addresses** tab
      - Scroll down to the **IPAll** section at the bottom
   
   b. Configure the ports:
      - **TCP Dynamic Ports**: Clear this field (delete any value, leave it blank)
      - **TCP Port**: Enter `1433`
   
   c. Click **OK**
   
   d. You will need to restart SQL Server for changes to take effect (see Step 6 below)
   
   > **Note**: If you configure a static port 1433, you can use the simpler connection string:
   > ```properties
   > sonar.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true
   > ```
   > Instead of using the instance name `\\SQLEXPRESS`.

4. **Enable SQL Server Browser** (Critical for named instances with dynamic ports):
   
   > **Note**: If you configured static port 1433 in Step 3, SQL Server Browser is optional. However, it's still recommended for easier instance discovery.
   
   SQL Server Browser is often **disabled** by default. You must enable it:
   
   **Method A: Using Services (Recommended)**
   - Press `Win + R`, type `services.msc` and press Enter
   - Scroll down and find **SQL Server Browser**
   - Right-click **SQL Server Browser** and select **Properties**
   - Change **Startup type** from **Disabled** to **Automatic**
   - Click **Apply**
   - Click the **Start** button
   - Click **OK**
   
   **Method B: Using SQL Server Configuration Manager**
   - Go to **SQL Server Services**
   - Right-click **SQL Server Browser** and select **Properties**
   - Go to the **Service** tab
   - Set **Start Mode** to **Automatic**
   - Click **OK**, then right-click and select **Start**

5. **Enable SQL Server Authentication (Mixed Mode)**:
   
   SQL Server Express defaults to Windows Authentication only. You must enable Mixed Mode for SonarQube:
   
   - Open **SQL Server Management Studio (SSMS)**
   - Connect using **Windows Authentication** to `localhost\SQLEXPRESS`
   - Right-click the server name (at top of Object Explorer)
   - Select **Properties**
   - Go to **Security** page
   - Under "Server authentication", select **SQL Server and Windows Authentication mode**
   - Click **OK**

6. **Restart SQL Server**:
   - In **SQL Server Services** (or `services.msc`), right-click **SQL Server (SQLEXPRESS)**
   - Select **Restart**

> **Important Notes**:
> - Without TCP/IP enabled, you'll see errors like "Connection refused" or "TCP/IP connection has failed"
> - Without Mixed Mode authentication, you'll see "Login failed for user 'sonarqube'"
> - Without SQL Server Browser running (if using dynamic ports), you'll see "SocketTimeoutException: Receive timed out"
> - **Tip**: Configuring static port 1433 (Step 3) eliminates the need for SQL Server Browser and simplifies troubleshooting

### Step 2: Connect to SQL Server Using SSMS

#### Connecting with SQL Server Management Studio (SSMS) 2022

1. **Open SQL Server Management Studio (SSMS)**

2. **In the "Connect to Server" dialog, enter the following:**
   - **Server type**: Database Engine
   - **Server name**: `localhost\SQLEXPRESS` (for SQL Server Express 2022)
     - Alternative formats: `.\SQLEXPRESS` or `(local)\SQLEXPRESS`
   - **Authentication**: Windows Authentication (default) or SQL Server Authentication

3. **Configure Certificate Trust for SQL Server 2022**
   
   SQL Server 2022 enforces encryption by default but uses a self-signed certificate. You'll need to trust this certificate:
   
   - Click **Options >>** at the bottom of the dialog
   - Go to the **Connection Properties** tab
   - Click **View all SQLServer properties** or look for **Advanced** button
   - Find and set **Trust Server Certificate**: `True` (check the box)
   
   **Alternative method**: In the **Additional Connection Parameters** field, add:
   ```
   TrustServerCertificate=True;
   ```

4. **Click Connect**

> **Troubleshooting Certificate Error**: If you encounter this error when connecting:
> 
> ```
> A connection was successfully established with the server, but then an error 
> occurred during the login process. (provider: SSL Provider, error: 0 - 
> The certificate chain was issued by an authority that is not trusted.) 
> (Microsoft SQL Server, Error: -2146893019)
> ```
> 
> **Solution**: Ensure you've set `Trust Server Certificate` to `True` in the connection properties as described in Step 3 above.

### Step 3: Create the SonarQube Database

Once connected to your SQL Server instance, execute the following steps:

#### 3.1 Create a New Database

```sql
CREATE DATABASE sonarqube;
```

#### 3.2 Set Database Collation

**CRITICAL**: Collation **MUST** be case-sensitive (CS) and accent-sensitive (AS).

```sql
ALTER DATABASE sonarqube 
COLLATE SQL_Latin1_General_CP1_CS_AS;
```

#### 3.3 Enable READ_COMMITTED_SNAPSHOT

**CRITICAL**: `READ_COMMITTED_SNAPSHOT` **MUST** be enabled to prevent deadlocks under heavy loads.

Check if it's already enabled:

```sql
SELECT is_read_committed_snapshot_on 
FROM sys.databases 
WHERE name='sonarqube';
```

If the result is `0` (false), enable it:

```sql
ALTER DATABASE sonarqube 
SET READ_COMMITTED_SNAPSHOT ON 
WITH ROLLBACK IMMEDIATE;
```

### Step 4: Create a SQL Server User for SonarQube

> **Prerequisites**: Ensure you've enabled **SQL Server Authentication (Mixed Mode)** in Step 1 before proceeding.

#### Option A: Using SQL Server Authentication (Recommended for cross-platform)

```sql
-- Check if the login already exists (optional)
SELECT name FROM sys.server_principals WHERE name = 'sonarqube';

-- Create login
CREATE LOGIN sonarqube WITH PASSWORD = 'YourStrongPassword123!';

-- Switch to the sonarqube database
USE sonarqube;

-- Check if user exists in database (optional)
SELECT name FROM sys.database_principals WHERE name = 'sonarqube';

-- Create user in the database
CREATE USER sonarqube FOR LOGIN sonarqube;

-- Grant necessary permissions
ALTER ROLE db_owner ADD MEMBER sonarqube;

-- Verify the user has permissions (optional)
SELECT 
    dp.name AS UserName,
    r.name AS RoleName
FROM sys.database_role_members drm
JOIN sys.database_principals dp ON drm.member_principal_id = dp.principal_id
JOIN sys.database_principals r ON drm.role_principal_id = r.principal_id
WHERE dp.name = 'sonarqube';
```

#### Test the SQL Server Authentication

Before configuring SonarQube, test the login in SSMS:

1. Disconnect from SQL Server in SSMS
2. Reconnect with:
   - **Server name**: `localhost\SQLEXPRESS`
   - **Authentication**: SQL Server Authentication
   - **Login**: `sonarqube`
   - **Password**: `YourStrongPassword123!`
   - **Trust Server Certificate**: Check this box

If the connection succeeds, proceed to SonarQube configuration.

#### Option B: Using Windows Authentication (Windows only)

```sql
-- Create user from Windows account
CREATE LOGIN [DOMAIN\username] FROM WINDOWS;

USE sonarqube;
CREATE USER [DOMAIN\username] FOR LOGIN [DOMAIN\username];

-- Grant necessary permissions
ALTER ROLE db_owner ADD MEMBER [DOMAIN\username];
```

---

## SonarQube Server Installation

### Step 1: Download SonarQube Server

1. Visit the [SonarQube Downloads page](https://www.sonarsource.com/products/sonarqube/downloads/)
2. Download the latest SonarQube Server Community, Developer, Enterprise, or Data Center Edition ZIP file

### Step 2: Extract the ZIP File

Extract the downloaded ZIP file to your desired installation directory:

- **Linux/macOS**: `/opt/sonarqube` (recommended)
- **Windows**: `C:\sonarqube` (recommended)

```bash
# Linux/macOS example
sudo unzip sonarqube-*.zip -d /opt/
sudo mv /opt/sonarqube-* /opt/sonarqube
```

```powershell
# Windows PowerShell example
Expand-Archive -Path sonarqube-*.zip -DestinationPath C:\
Rename-Item -Path C:\sonarqube-* -NewName C:\sonarqube
```

---

## Configuration

### Step 1: Configure Database Connection

Edit the `sonar.properties` file located in `<sonarqube_home>/conf/sonar.properties`:

#### For SQL Server Express (Named Instance with Dynamic Port):

```properties
# Database connection string for SQL Server Express with dynamic port
# Note: Use double backslash (\\) for instance name in properties file
# Requires SQL Server Browser service to be running
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true

# Database credentials
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourStrongPassword123!
```

#### For SQL Server Express (Named Instance with Static Port 1433):

```properties
# Database connection string for SQL Server Express with static port 1433
# Note: If you configured TCP Port 1433 in SQL Server Configuration Manager (Step 3)
sonar.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true

# Database credentials
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourStrongPassword123!
```

#### For SQL Server Default Instance:

```properties
# Database connection string for default instance
sonar.jdbc.url=jdbc:sqlserver://localhost;databaseName=sonarqube;trustServerCertificate=true

# Database credentials
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourStrongPassword123!
```

#### For Windows Integrated Security:

**SQL Server Express (Named Instance):**
```properties
# Database connection string with integrated security
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;integratedSecurity=true;trustServerCertificate=true

# DO NOT set username and password when using integrated security
# sonar.jdbc.username=
# sonar.jdbc.password=
```

**SQL Server Default Instance:**
```properties
# Database connection string with integrated security
sonar.jdbc.url=jdbc:sqlserver://localhost;databaseName=sonarqube;integratedSecurity=true;trustServerCertificate=true

# DO NOT set username and password when using integrated security
# sonar.jdbc.username=
# sonar.jdbc.password=
```

#### For Remote SQL Server:

**Default Instance:**
```properties
# Replace 'localhost' with your SQL Server hostname or IP address
sonar.jdbc.url=jdbc:sqlserver://sql-server.example.com:1433;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourStrongPassword123!
```

**Named Instance:**
```properties
# For remote named instance (e.g., SQLEXPRESS)
sonar.jdbc.url=jdbc:sqlserver://sql-server.example.com\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourStrongPassword123!
```

### Step 2: Configure Encryption Settings (Optional but Recommended)

#### If SQL Server doesn't support encryption:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost;databaseName=sonarqube;encrypt=false
```

#### If SQL Server requires encryption but without certificate validation:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost;databaseName=sonarqube;trustServerCertificate=true
```

### Step 3: Configure Integrated Security (Windows Only)

If using Windows Authentication with Integrated Security:

1. Download the [Microsoft SQL JDBC Auth package v12.10.2](https://github.com/microsoft/mssql-jdbc/releases/download/v12.10.2/mssql-jdbc_auth.zip)

2. Extract `mssql-jdbc_auth-12.10.2.x64.dll` from the downloaded ZIP

3. Copy the DLL to a folder in your system's PATH environment variable, such as:
   - `C:\Windows\System32`
   - Or add a custom folder to PATH

4. If running SonarQube as a Windows service, ensure the service account has permission to connect to SQL Server

### Step 4: Configure Web Server Settings (Optional)

In `sonar.properties`, you can customize:

```properties
# Web server port (default is 9000)
sonar.web.port=9000

# Web server host (default is 0.0.0.0)
sonar.web.host=0.0.0.0

# Web context path (default is root)
# sonar.web.context=/sonarqube
```

### Step 5: Configure JVM Options (Optional)

For optimal performance, you may want to adjust JVM settings in `<sonarqube_home>/conf/sonar.properties`:

```properties
# Web Server JVM options
sonar.web.javaOpts=-Xmx512m -Xms128m -XX:+HeapDumpOnOutOfMemoryError

# Compute Engine JVM options
sonar.ce.javaOpts=-Xmx512m -Xms128m -XX:+HeapDumpOnOutOfMemoryError

# Elasticsearch JVM options
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
```

---

## Starting SonarQube

### Linux/macOS

```bash
# Navigate to the SonarQube bin directory
cd /opt/sonarqube/bin/linux-x86-64/

# Start SonarQube
./sonar.sh start

# Check status
./sonar.sh status

# View logs
tail -f /opt/sonarqube/logs/sonar.log
```

### Windows

```powershell
# Navigate to the SonarQube bin directory
cd C:\sonarqube\bin\windows-x86-64\

# Start SonarQube
StartSonar.bat

# Or install as a Windows service
InstallNTService.bat
net start SonarQube
```

### Using Console Mode (for troubleshooting)

```bash
# Linux/macOS
/opt/sonarqube/bin/linux-x86-64/sonar.sh console

# Windows
C:\sonarqube\bin\windows-x86-64\StartSonar.bat
```

---

## Post-Installation

### Step 1: Access the Web Interface

1. Open a web browser and navigate to: `http://localhost:9000`
2. Wait for SonarQube to fully start (this may take a few minutes on first startup)

### Step 2: Log In with Default Credentials

- **Username**: `admin`
- **Password**: `admin`

> **Important**: You will be prompted to change the password on first login.

### Step 3: Configure Security Settings

After logging in:

1. Change the default admin password
2. Review global permissions at **Administration > Security > Global Permissions**
3. Remove the **Execute Analysis** permission from the `sonar-users` group if needed
4. Configure additional users and groups as required

### Step 4: Install Language Plugins (if needed)

1. Navigate to **Administration > Marketplace**
2. Browse and install plugins for additional language support
3. Restart SonarQube after installing plugins

### Step 5: Configure Your First Project

1. Click **Create Project** or navigate to **Administration > Projects > Management**
2. Follow the wizard to set up your first project
3. Generate an authentication token for analysis
4. Configure your build tool or CI/CD pipeline

---

## Troubleshooting

### Quick Fixes for Common SQL Server Express Errors

#### Error 1: Certificate Chain Not Trusted

**Error Message:**
```
The certificate chain was issued by an authority that is not trusted.
(Microsoft SQL Server, Error: -2146893019)
```

**Fix**: Add `trustServerCertificate=true` to connection string:
```properties
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
```

#### Error 2: Connection Refused on Port 1433

**Error Message:**
```
The TCP/IP connection to the host localhost, port 1433 has failed. 
Error: "Connection refused: getsockopt.
```

**Fix Option 1**: Use named instance in connection string:
```properties
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
```

**Fix Option 2 (Recommended)**: Configure SQL Server Express to use port 1433:
1. Open **SQL Server Configuration Manager**
2. **Protocols for SQLEXPRESS** → Right-click **TCP/IP** → **Properties**
3. Go to **IP Addresses** tab → Scroll to **IPAll** section
4. Clear **TCP Dynamic Ports**, set **TCP Port** to `1433`
5. Restart SQL Server service
6. Use connection string:
```properties
sonar.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true
```

#### Error 3: Connection Timeout / SQL Server Browser Not Running

**Error Message:**
```
The connection to the host localhost, named instance sqlexpress failed. 
Error: "java.net.SocketTimeoutException: Receive timed out"
```

**Fix Option 1**: Enable and start SQL Server Browser service:
1. Press `Win + R`, type `services.msc` and press Enter
2. Find **SQL Server Browser**
3. Right-click → **Properties** → Change **Startup type** to **Automatic**
4. Click **Apply** → Click **Start** → Click **OK**
5. Restart SonarQube

**Fix Option 2 (Recommended)**: Configure static port 1433 to avoid needing SQL Server Browser:
1. Open **SQL Server Configuration Manager**
2. **Protocols for SQLEXPRESS** → Right-click **TCP/IP** → **Properties**
3. Go to **IP Addresses** tab → Scroll to **IPAll** section
4. Clear **TCP Dynamic Ports**, set **TCP Port** to `1433`
5. Restart SQL Server service
6. Use connection string: `jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true`

#### Error 4: Login Failed for User

**Error Message:**
```
Login failed for user 'sonarqube'.
```

**Fix**: Enable SQL Server Authentication (Mixed Mode):
1. Open SSMS with **Windows Authentication** (`localhost\SQLEXPRESS`)
2. Right-click server → **Properties** → **Security**
3. Select **SQL Server and Windows Authentication mode**
4. Click **OK**
5. Restart SQL Server service in `services.msc`
6. In SSMS, run:
   ```sql
   CREATE LOGIN sonarqube WITH PASSWORD = 'YourStrongPassword123!';
   USE sonarqube;
   CREATE USER sonarqube FOR LOGIN sonarqube;
   ALTER ROLE db_owner ADD MEMBER sonarqube;
   ```

#### Error 5: Login Failed for Integrated Security (Windows Only)

**Error Message:**
```
Cannot open database "sonarqube" requested by the login. The login failed.
```

**Fix**: Grant permissions to the account running SonarQube (usually Local System Account):
1. Identify the account: Usually `NT AUTHORITY\SYSTEM` (Local System Account)
2. In SSMS, run:
   ```sql
   CREATE LOGIN [NT AUTHORITY\SYSTEM] FROM WINDOWS;
   USE sonarqube;
   CREATE USER [NT AUTHORITY\SYSTEM] FOR LOGIN [NT AUTHORITY\SYSTEM];
   ALTER ROLE db_owner ADD MEMBER [NT AUTHORITY\SYSTEM];
   ```
3. Ensure `integratedSecurity=true` is in the connection string and **username/password are commented out**.

> **Note**: All five issues are common with SQL Server Express 2022. 
> 
> **Recommended setup** (simplest configuration):
> - Configure static port 1433 in SQL Server Configuration Manager (see Error 2/3 Fix Option 2)
> - Trust the certificate: `trustServerCertificate=true`
> - Enable Mixed Mode authentication and create the SQL login
> - Use connection string: `jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true`
>
> **Alternative setup** (using named instance):
> - Use named instance: `localhost\\SQLEXPRESS`
> - Trust the certificate: `trustServerCertificate=true`
> - Enable SQL Server Browser service
> - Enable Mixed Mode authentication and create the SQL login
> - Use connection string: `jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true`

### Common Issues

#### 1. SonarQube Won't Start

**Check the logs:**
```bash
# Linux/macOS
tail -f /opt/sonarqube/logs/sonar.log

# Windows
type C:\sonarqube\logs\sonar.log
```

**Common causes:**
- Java not installed or wrong version
- Port 9000 already in use
- Insufficient permissions
- Database connection issues

#### 2. Database Connection Errors

##### Error: "TCP/IP connection has failed" or "Connection refused"

This is the most common error, especially with SQL Server Express.

**Error Messages You May See:**

```
The TCP/IP connection to the host localhost, port 1433 has failed. 
Error: "Connection refused: getsockopt. Verify the connection properties. 
Make sure that an instance of SQL Server is running on the host and accepting 
TCP/IP connections at the port. Make sure that TCP connections to the port are 
not blocked by a firewall."
```

Or:

```
The connection to the host localhost, named instance sqlexpress failed. 
Error: "java.net.SocketTimeoutException: Receive timed out". 
Verify the server and instance names and check that no firewall is blocking 
UDP traffic to port 1434. For SQL Server 2005 or later, verify that the 
SQL Server Browser Service is running on the host.
```

**Solution 1: Use Named Instance in Connection String**

If using SQL Server Express, specify the instance name:

```properties
# Use double backslash for instance name
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
```

**Solution 2: Enable and Start SQL Server Browser**

If you see error: "The connection to the host localhost, named instance sqlexpress failed" or "SocketTimeoutException: Receive timed out", the SQL Server Browser service is disabled:

1. Press `Win + R`, type `services.msc` and press Enter
2. Find **SQL Server Browser**
3. Right-click and select **Properties**
4. Change **Startup type** from **Disabled** to **Automatic**
5. Click **Apply**
6. Click the **Start** button
7. Click **OK**

**Solution 3: Enable TCP/IP Protocol**

1. Open **SQL Server Configuration Manager**
   - Press `Win + R`, type: `SQLServerManager16.msc` (SQL Server 2022)
   - Or search "SQL Server Configuration Manager"

2. Expand **SQL Server Network Configuration**
3. Click **Protocols for [INSTANCE_NAME]**
4. Right-click **TCP/IP** and select **Enable**

5. Restart SQL Server service:
   - Go to **SQL Server Services**
   - Right-click **SQL Server (SQLEXPRESS)** and select **Restart**

**Solution 4: Configure Static Port 1433 (Recommended Alternative)**

Instead of using SQL Server Browser, configure SQL Server Express to use static port 1433:

1. Open **SQL Server Configuration Manager**
2. Go to **SQL Server Network Configuration** → **Protocols for SQLEXPRESS**
3. Right-click **TCP/IP** and select **Properties**
4. Go to **IP Addresses** tab
5. Scroll to bottom to **IPAll** section
6. Configure:
   - **TCP Dynamic Ports**: Clear this field (leave blank)
   - **TCP Port**: Enter `1433`
7. Click **OK**
8. Restart SQL Server service:
   - Go to **SQL Server Services**
   - Right-click **SQL Server (SQLEXPRESS)** and select **Restart**

Then use this simpler connection string:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true
```

> **Benefits**: No need for SQL Server Browser, simpler connection string, consistent port across environments.

**Solution 5: Use Dynamic Port Connection (If Static Port Not Possible)**

If you cannot set a static port, find and use the current dynamic port:

1. Open **SQL Server Configuration Manager**
2. Go to **SQL Server Network Configuration** → **Protocols for SQLEXPRESS**
3. Right-click **TCP/IP** and select **Properties**
4. Go to **IP Addresses** tab
5. Scroll to bottom to **IPAll** section
6. Note the port number in **TCP Dynamic Ports** (e.g., `49172`)

Then use this connection string:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost:49172;databaseName=sonarqube;trustServerCertificate=true
```

Replace `49172` with your actual port number.

**Solution 6: Check Firewall**
- Ensure SQL Server port is not blocked
- For named instances with SQL Server Browser, allow UDP port 1434
- For static port configuration, allow TCP port 1433
- For dynamic ports, allow the specific TCP port shown in IPAll

**Verify connection string:**
```bash
# Test connectivity from command line
# For named instance:
sqlcmd -S localhost\SQLEXPRESS -d sonarqube -U sonarqube -P YourPassword

# For default instance:
sqlcmd -S localhost -d sonarqube -U sonarqube -P YourPassword
```

##### Error: "Certificate chain was issued by an authority that is not trusted"

**Error Message:**

```
A connection was successfully established with the server, but then an error 
occurred during the login process. (provider: SSL Provider, error: 0 - 
The certificate chain was issued by an authority that is not trusted.) 
(Microsoft SQL Server, Error: -2146893019)
```

**Solution**: Add `trustServerCertificate=true` to your connection string:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
```

This is required for SQL Server 2022 which enforces encryption by default but uses a self-signed certificate.

##### Error: "Login failed for user 'sonarqube'"

**Error Message:**

```
Login failed for user 'sonarqube'. 
ClientConnectionId:e6cf02b8-cc7f-476c-90b4-ece9ef791027
```

This error occurs when:
- SQL Server Authentication (Mixed Mode) is not enabled
- The `sonarqube` login doesn't exist
- The password is incorrect
- The user isn't mapped to the database

**Solution - Step 1: Enable Mixed Mode Authentication**

1. Open **SSMS** and connect using **Windows Authentication**:
   - Server name: `localhost\SQLEXPRESS`
   - Authentication: Windows Authentication

2. Enable **SQL Server and Windows Authentication mode**:
   - Right-click the server name (at top of Object Explorer)
   - Select **Properties**
   - Go to **Security** page
   - Under "Server authentication", select **SQL Server and Windows Authentication mode**
   - Click **OK**

3. Restart SQL Server:
   - Press `Win + R`, type `services.msc` and press Enter
   - Find **SQL Server (SQLEXPRESS)**
   - Right-click and select **Restart**

**Solution - Step 2: Create or Verify the SonarQube Login**

Connect to SSMS (Windows Authentication) and run:

```sql
-- Check if the login already exists
SELECT name FROM sys.server_principals WHERE name = 'sonarqube';

-- If it doesn't exist, create the login
CREATE LOGIN sonarqube WITH PASSWORD = 'YourStrongPassword123!';

-- Switch to the sonarqube database
USE sonarqube;

-- Check if user exists in database
SELECT name FROM sys.database_principals WHERE name = 'sonarqube';

-- If it doesn't exist, create the user
CREATE USER sonarqube FOR LOGIN sonarqube;

-- Grant db_owner permissions
ALTER ROLE db_owner ADD MEMBER sonarqube;

-- Verify the user has permissions
SELECT 
    dp.name AS UserName,
    r.name AS RoleName
FROM sys.database_role_members drm
JOIN sys.database_principals dp ON drm.member_principal_id = dp.principal_id
JOIN sys.database_principals r ON drm.role_principal_id = r.principal_id
WHERE dp.name = 'sonarqube';
```

**Solution - Step 3: Test the SQL Server Login**

Before trying SonarQube again, test the login in SSMS:

1. Disconnect from the server
2. Reconnect with:git s
   - Server name: `localhost\SQLEXPRESS`
   - Authentication: **SQL Server Authentication**
   - Login: `sonarqube`
   - Password: `YourStrongPassword123!`

If the connection succeeds, your SonarQube connection will work.

**Solution - Step 4: Verify sonar.properties**

Ensure the password matches exactly:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourStrongPassword123!
```

##### Error: "Cannot open database requested by the login" (Integrated Security)

**Error Message:**

```
Cannot open database "sonarqube" requested by the login. The login failed.
```

This error occurs with integrated security when the Windows account running the SonarQube service does not have permissions in SQL Server.

**Solution - Step 1: Identify the Windows Account**

1. Press `Win + R`, type `services.msc` and press Enter.
2. Find the **SonarQube** service.
3. Right-click → **Properties** → **Log On** tab.
4. Note the account:
   - If it says **Local System account**, the user is `NT AUTHORITY\SYSTEM`.
   - If it says **This account**, it's a specific user (e.g., `DOMAIN\username`).

**Solution - Step 2: Grant Permissions in SQL Server**

Connect to SSMS (Windows Authentication) and run the following (replace `[NT AUTHORITY\SYSTEM]` with your account if different):

```sql
-- Create login for the account
CREATE LOGIN [NT AUTHORITY\SYSTEM] FROM WINDOWS;

-- Switch to the sonarqube database
USE sonarqube;

-- Create user for the login
CREATE USER [NT AUTHORITY\SYSTEM] FOR LOGIN [NT AUTHORITY\SYSTEM];

-- Grant db_owner permissions
ALTER ROLE db_owner ADD MEMBER [NT AUTHORITY\SYSTEM];
```

**Solution - Step 3: Verify sonar.properties**

Ensure `integratedSecurity=true` is used and credentials are removed:

```properties
sonar.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=sonarqube;integratedSecurity=true;trustServerCertificate=true

# DO NOT set username and password
# sonar.jdbc.username=
# sonar.jdbc.password=
```

#### 3. Elasticsearch Won't Start

**Common causes:**
- Insufficient disk space (needs at least 10% free)
- Wrong file permissions
- Insufficient RAM

**Linux-specific prerequisites:**
```bash
# Check and set required limits
sysctl vm.max_map_count
sudo sysctl -w vm.max_map_count=524288

# Make permanent
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
```

#### 4. Performance Issues

**Increase JVM heap sizes:**
```properties
# In sonar.properties
sonar.web.javaOpts=-Xmx1024m -Xms256m
sonar.ce.javaOpts=-Xmx1024m -Xms256m
sonar.search.javaOpts=-Xmx1024m -Xms1024m
```

**Use SSD storage:**
- Elasticsearch performance heavily depends on disk I/O
- SSDs provide significantly better performance than spinning disks

#### 5. READ_COMMITTED_SNAPSHOT Error

If you see deadlock errors, verify the snapshot isolation is enabled:

```sql
SELECT is_read_committed_snapshot_on 
FROM sys.databases 
WHERE name='sonarqube';
```

If it returns 0, enable it as shown in the Database Configuration section.

---

## Database Connection Examples

### SQL Server Express with Static Port 1433 (Recommended)

```properties
# SQL Server Express with static port 1433
# RECOMMENDED: Simplest configuration, no SQL Server Browser needed
# Configure static port in SQL Server Configuration Manager (see Installation Step 3)
sonar.jdbc.url=jdbc:sqlserver://localhost:1433;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourPassword123!
```

### SQL Server Express (Named Instance with Dynamic Port)

```properties
# SQL Server Express with named instance and dynamic port
# IMPORTANT: Use double backslash (\\) for instance name
# IMPORTANT: Requires SQL Server Browser service to be running
# IMPORTANT: Include trustServerCertificate=true for SQL Server 2022
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourPassword123!
```

### SQL Server Default Instance

```properties
# Local SQL Server with default instance (uses port 1433 by default)
sonar.jdbc.url=jdbc:sqlserver://localhost;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourPassword123!
```

### Remote Server with Custom Port

```properties
# Remote SQL Server on custom port
sonar.jdbc.url=jdbc:sqlserver://db-server.example.com:1434;databaseName=sonarqube;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourPassword123!
```

### Encryption Enabled with Trusted Certificate

```properties
# Encrypted connection with certificate validation disabled
sonar.jdbc.url=jdbc:sqlserver://localhost;databaseName=sonarqube;encrypt=true;trustServerCertificate=true
sonar.jdbc.username=sonarqube
sonar.jdbc.password=YourPassword123!
```

### Using Environment Variables (Recommended for Production)

Instead of storing credentials in `sonar.properties`, use environment variables:

```bash
# Linux/macOS - Default Instance
export SONAR_JDBC_URL="jdbc:sqlserver://localhost;databaseName=sonarqube;trustServerCertificate=true"
export SONAR_JDBC_USERNAME="sonarqube"
export SONAR_JDBC_PASSWORD="YourPassword123!"

# Linux/macOS - Named Instance (SQL Server Express)
export SONAR_JDBC_URL="jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true"
export SONAR_JDBC_USERNAME="sonarqube"
export SONAR_JDBC_PASSWORD="YourPassword123!"

# Windows PowerShell - Default Instance
$env:SONAR_JDBC_URL="jdbc:sqlserver://localhost;databaseName=sonarqube;trustServerCertificate=true"
$env:SONAR_JDBC_USERNAME="sonarqube"
$env:SONAR_JDBC_PASSWORD="YourPassword123!"

# Windows PowerShell - Named Instance (SQL Server Express)
$env:SONAR_JDBC_URL="jdbc:sqlserver://localhost\SQLEXPRESS;databaseName=sonarqube;trustServerCertificate=true"
$env:SONAR_JDBC_USERNAME="sonarqube"
$env:SONAR_JDBC_PASSWORD="YourPassword123!"
```

> **Note**: In PowerShell, use single backslash `\SQLEXPRESS`. In properties files, use double backslash `\\SQLEXPRESS`.

---

## Security Best Practices

### 1. Database Security

- Use strong passwords for SQL Server accounts
- Limit database user permissions to only what's necessary
- Enable SQL Server encryption (TLS/SSL)
- Keep SQL Server patched and up to date
- Use firewall rules to restrict database access

### 2. SonarQube Security

- Change the default admin password immediately
- Use strong authentication mechanisms (LDAP, SAML, etc.)
- Enable HTTPS for the web interface (use a reverse proxy)
- Review and configure global permissions
- Regularly update SonarQube to the latest version
- Configure regular database backups

### 3. Network Security

- Use a reverse proxy (NGINX, Apache) in front of SonarQube
- Enable HTTPS with valid SSL certificates
- Restrict access to port 9000 to trusted networks only
- Consider using a VPN for remote access

---

## Backup and Maintenance

### Database Backup

```sql
-- Create a full backup
BACKUP DATABASE sonarqube 
TO DISK = 'C:\Backups\sonarqube_backup.bak'
WITH FORMAT, 
     MEDIANAME = 'SonarQubeBackup',
     NAME = 'Full Backup of SonarQube';
```

### SonarQube Data Backup

Backup the following directories:
- `<sonarqube_home>/data/` - Elasticsearch data
- `<sonarqube_home>/extensions/` - Installed plugins
- `<sonarqube_home>/conf/sonar.properties` - Configuration

### Updating SonarQube

1. Backup the database and SonarQube directories
2. Download the new version
3. Stop the current SonarQube instance
4. Replace the binaries (keep `data`, `extensions`, and `conf` folders)
5. Start SonarQube - it will automatically upgrade the database schema
6. Check logs for any errors

---

## References

### Official Documentation

- [SonarQube Server Documentation](https://docs.sonarsource.com/sonarqube-server)
- [Installing Database](https://docs.sonarsource.com/sonarqube-server/server-installation/installing-the-database)
- [Server Host Requirements](https://docs.sonarsource.com/sonarqube-server/server-installation/server-host-requirements)
- [Configuration Properties](https://docs.sonarsource.com/sonarqube-server/server-installation/system-properties/common-properties)
- [Installing from ZIP](https://docs.sonarsource.com/sonarqube-server/server-installation/from-zip-file)

### Microsoft SQL Server

- [Microsoft SQL Server Downloads](https://www.microsoft.com/sql-server/sql-server-downloads)
- [Microsoft SQL JDBC Driver](https://docs.microsoft.com/sql/connect/jdbc/microsoft-jdbc-driver-for-sql-server)
- [SQL Server Authentication Modes](https://docs.microsoft.com/sql/relational-databases/security/choose-an-authentication-mode)

### Community Support

- [SonarSource Community Forum](https://community.sonarsource.com/)
- [SonarQube GitHub Repository](https://github.com/SonarSource/sonarqube)

---

## Version Information

This guide is based on:
- **SonarQube Server**: Version 2025.6 (latest as of January 2026)
- **Microsoft SQL Server**: Versions 2016, 2017, 2019, 2022
- **Microsoft JDBC Driver**: Version 12.10.2

---

## License

SonarQube Server is available in multiple editions:
- **Community Edition**: Open source (LGPL v3)
- **Developer Edition**: Commercial license
- **Enterprise Edition**: Commercial license
- **Data Center Edition**: Commercial license

Visit [SonarSource Plans and Pricing](https://www.sonarsource.com/plans-and-pricing/) for more information.

---

## Support

For technical support and questions:
- **Community Forum**: https://community.sonarsource.com/
- **Documentation**: https://docs.sonarsource.com/sonarqube-server
- **Enterprise Support**: Available with commercial licenses

---

*Last Updated: January 2026*
