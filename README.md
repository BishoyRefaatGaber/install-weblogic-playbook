```markdown
# WebLogic Installation Using Ansible

This document explains the process of installing Oracle WebLogic using an Ansible script. The installation is divided into several stages, including setting up Java (JDK), configuring the environment, running the WebLogic installer in silent mode, and configuring the WebLogic domain using WLST. Each section includes the necessary commands and explanations to help you understand and customize the installation process.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [JDK Installation](#jdk-installation)
- [Preparing the WebLogic User](#preparing-the-weblogic-user)
- [Environment Variables and Directories](#environment-variables-and-directories)
- [WebLogic Installer Setup](#weblogic-installer-setup)
- [Silent Installation of WebLogic](#silent-installation-of-weblogic)
- [Domain Configuration Using WLST](#domain-configuration-using-wlst)
- [Deploying to Managed Nodes](#deploying-to-managed-nodes)
- [Notes and Customization](#notes-and-customization)

## Overview

This guide walks you through the automated installation of Oracle WebLogic using an Ansible script. The process includes:
- Installing Java and setting up the environment.
- Creating a dedicated non-root user (`weblogic`) for the WebLogic installation.
- Downloading and extracting the WebLogic installer.
- Running the silent installation with response files.
- Configuring an Admin Server and domain through WLST (WebLogic Scripting Tool).
- Packaging and deploying configurations to managed nodes.

## Prerequisites

Before proceeding, make sure that:
- You have administrative privileges to install JDK and configure system users.
- The Oracle installation process is executed under a non-root user (for the Oracle database, if applicable).
- The required files (JDK RPM, WebLogic installer ZIP, and installer JAR) are accessible.

## JDK Installation

Install the Java Development Kit (JDK) required by WebLogic:

```bash
# Download the JDK RPM package
wget https://download.oracle.com/otn/java/jdk/8u202-b08/1961070e4c9b4e26a04e7f5a083f551e/jdk-8u202-linux-x64.rpm

# Install the JDK package
rpm -ivh jdk-8u202-linux-x64.rpm
```

Then, update the root user’s environment:

```bash
vi /root/.bash_profile
# Add the following lines:
export JAVA_HOME=/usr/java/jdk1.8.0_202-amd64
PATH=$JAVA_HOME/bin:$PATH:$HOME/bin
export PATH
```

## Preparing the WebLogic User

Since Oracle installations must run using a non-root user, create a user for WebLogic:

```bash
# Create a new user and set the password
useradd weblogic
passwd weblogic

# Switch to the new weblogic user
su -l weblogic
```

## Environment Variables and Directories

For a smooth installation, define the following variables that point to key directories:

- **ORACLE_BASE:** Default Oracle installer directory location.
- **ORACLE_HOME:** Directory for Oracle database installation or Oracle client.
- **MW HOME:** Middleware installation directory.
- **WLS HOME / WL HOME:** Directories for the managed and admin servers.
- **DOMAIN BASE / DOMAIN HOME:** Global and specific domain configuration directories.

Edit the weblogic user’s bash profile:

```bash
vi /home/weblogic/.bash_profile

# Set base directories and Java environment
export ORACLE_BASE=/home/weblogic/wls/oracle
export ORACLE_HOME=$ORACLE_BASE/product/fmw14
export MW_HOME=$ORACLE_HOME
export WLS_HOME=$MW_HOME/wlserver
export WL_HOME=$WLS_HOME  # Simplified variable usage

# Domain paths
export DOMAIN_BASE=$ORACLE_BASE/config/domains
export DOMAIN_HOME=$DOMAIN_BASE/TEST

# Java setup
export JAVA_HOME=/usr/java/jdk1.8.0_202-amd64
export PATH=$JAVA_HOME/bin:$PATH:$HOME/bin
```

Load the profile:

```bash
source /home/weblogic/.bash_profile
```

Create the required directories:

```bash
mkdir -p $ORACLE_BASE
mkdir -p $DOMAIN_BASE
mkdir -p $ORACLE_HOME
mkdir -p $ORACLE_BASE/config/applications
mkdir -p /home/weblogic/wls/oraInventory  # Inventory directory for WebLogic installer
```

## WebLogic Installer Setup

Change to the directory where you wish to execute the installation:

```bash
cd wls
```

Create the **Inventory Pointer File** `oraInst.loc` with:

```bash
vi oraInst.loc
# Include:
inventory_loc=/home/weblogic/wls/oraInventory
inst_group=weblogic
```

Create a **Response File** `wls.rsp` for silent installation:

```bash
vi wls.rsp
```

Add the following content to the response file:

```ini
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
ORACLE_HOME=/home/weblogic/wls/oracle/product/fmw14
INSTALL_TYPE=WebLogic Server
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
```

Set the proper permissions on the inventory directory:

```bash
chmod -R 775 /home/weblogic/wls/oraInventory
chown -R weblogic:weblogic /home/weblogic/wls/oraInventory

# Ensure that the weblogic group exists
getent group weblogic || sudo groupadd weblogic
```

## Silent Installation of WebLogic

Navigate to the Oracle base directory and download the WebLogic installer ZIP:

```bash
cd $ORACLE_BASE
wget https://www.oracle.com/webapps/redirect/signon?nexturl=https://download.oracle.com/otn/nt/middleware/14c/14110/fmw_14.1.1.0.0_wls_lite_Disk1_1of1.zip
unzip fmw_14.1.1.0.0_wls_lite_Disk1_1of1.zip
```

Run the installer jar in silent mode with the response and inventory pointer files:

```bash
java -jar ./fmw_14.1.1.0.0_wls.jar -silent -responseFile /home/weblogic/wls/wls.rsp -invPtrLoc /home/weblogic/wls/oraInst.loc
```

## Domain Configuration Using WLST

After installation, you need to configure your domain and Admin Server using WLST. Follow these steps:

1. Change to the common bin directory and initialize the environment:

    ```bash
    cd $WL_HOME/common/bin/
    /home/weblogic/wls/oracle/product/fmw14/oracle_common/common/bin/commEnv.sh
    ./wlst.sh
    ```

2. Once in WLST, execute the following commands:

    ```wlst
    # Start by reading the domain template
    wls:/offline> readTemplate('/home/weblogic/wls/oracle/product/fmw14/wlserver/common/templates/wls/wls.jar')

    # Configure the Admin Server
    wls:/offline/base_domain> cd('Servers/AdminServer')
    wls:/offline/base_domain/Server/AdminServer> set('ListenAddress','<server-IP>')  # Replace <server-IP>
    wls:/offline/base_domain/Server/AdminServer> set('ListenPort',7001)

    # Create SSL configuration for the Admin Server
    wls:/offline/base_domain/Server/AdminServer> create('AdminServer','SSL')
    wls:/offline/base_domain/Server/AdminServer> cd('SSL/AdminServer')
    wls:/offline/base_domain/Server/AdminServer/SSL/AdminServer> set('Enabled','true')
    wls:/offline/base_domain/Server/AdminServer/SSL/AdminServer> set('ListenPort',7002)

    # Configure the security settings
    wls:/offline/base_domain/Server/AdminServer/SSL/AdminServer> cd('/')
    wls:/offline/base_domain> cd('Security/base_domain/User/weblogic')
    wls:/offline/base_domain/Security/base_domain/User/weblogic> cmo.setPassword('Test1234')

    # Finalize the domain creation
    wls:/offline/base_domain/Security/base_domain/User/weblogic> setOption('OverwriteDomain','true')
    wls:/offline/base_domain> writeDomain('/home/weblogic/wls/oracle/config/domains/TEST')

    # Cleanup and exit WLST
    wls:/offline/TEST/Security/TEST/User/weblogic> closeTemplate()
    wls:/offline> exit()
    ```

3. Start the Admin Server:

    ```bash
    cd $DOMAIN_HOME/bin/
    ./startWebLogic.sh &
    ```

    You can verify that the server is listening on the correct port by:

    ```bash
    netstat -apn | grep -i :70
    ```

## Deploying to Managed Nodes

After configuring your Admin Server, you can copy the WebLogic configuration to managed server nodes:

1. **Pack the Domain Configuration:**

    ```bash
    $WL_HOME/common/bin/pack.sh -domain=/home/weblogic/wls/oracle/config/domains/TEST
    $WL_HOME/common/bin/pack.sh -domain=$DOMAIN_HOME -template=$WL_HOME/common/templates/domains/TEST_template.jar -template_name=TEST -managed=true
    ```

2. **Copy the Template File:**

    ```bash
    scp -r /home/weblogic/wls/oracle/product/fmw12/wlserver/common/templates/domains/TEST_template.jar user@<managed-node-IP>:/home/weblogic/wls/
    ```

3. **On the Managed Server Node, Unpack the Domain:**

    ```bash
    cd $WL_HOME
    $WL_HOME/common/bin/unpack.sh -template=/home/weblogic/wls/TEST_template.jar -domain=$DOMAIN_HOME
    ```

4. **Start the Managed Server:**

    ```bash
    cd $DOMAIN_HOME/bin/
    ./stopManagedWebLogic.sh Node_Server01 t3://<admin-server-IP>:7001 weblogic Test1234
    ./startManagedWebLogic.sh Node_Server01 t3://<admin-server-IP>:7001 &
    ```

## Notes and Customization

- **Environment Variables:** Modify the variables in the bash profiles as needed for your installation paths.
- **Response Files:** Adjust the `wls.rsp` file parameters if additional installation options are required.
- **WLST Scripting:** Replace `<server-IP>` and `<admin-server-IP>` with your actual IP addresses.
- **Security:** Change default passwords after installation for enhanced security.
- **Ansible Integration:** Incorporate these shell commands into your Ansible playbooks and roles to automate the entire installation process.
``` 

Simply paste the above into your Markdown file in one block.