JBoss EAP 7.4 to EAP 8.1 Server Migration Standard Operating Procedure (SOP)
1. Directory Structure & File System Layout (RPM Installation)
Red Hat EAP 8.1 installed via RPM adheres to the Linux Filesystem Hierarchy Standard (FHS). Binaries and libraries are kept separate from configurations and runtime data to ensure patches do not overwrite customizations.
1.1 Directory Layout
Component	EAP 8.1 Path	Description
`JBOSS\_HOME`	`/opt/rh/eap8/root/usr/share/wildfly`	Read-only binaries, scripts, and base modules.
Configuration	`/etc/opt/rh/eap8/wildfly/standalone/`	System configuration files (`standalone.xml`). Modify files here, not under `/opt`.
Runtime Data	`/var/opt/rh/eap8/wildfly/standalone/`	Runtime server data, deployments, and temporary caches.
Logs	`/var/log/opt/rh/eap8/wildfly/`	Server logs (`server.log`).
Systemd Service	`eap8-standalone.service`	Managed system daemon service.
> \*\*Note on Symlinks:\*\* Red Hat creates symbolic links inside `/opt/rh/eap8/root/usr/share/wildfly/standalone` pointing to `/etc/opt/rh/eap8/...` and `/var/...`. Always edit configurations in `/etc` to prevent changes from being wiped during future `dnf update` operations.
---
2. Environment Preparation & Strict Security Permissions
In accordance with Red Hat and enterprise Linux security standards, file ownership is split between `root` (for binaries and configuration definitions) and `jboss` (for logs and runtime data). This prevents a compromised `jboss` application process from altering server binaries or system configurations.
2.1 File Ownership & Permissions Matrix
Path / Directory	Target Ownership	Permissions	Description & Rationale
`/opt/rh/eap8/`	`root:jboss`	`755`	Core binaries and base modules. Read-only for `jboss` to prevent unauthorized binary modification.
`/etc/opt/rh/eap8/`	`root:jboss`	`770`	Configuration files (`standalone.xml`). `root` owns the files while `jboss` group has read/write access for administrative operations.
`/var/log/opt/rh/eap8/`	`jboss:jboss`	`770`	Application and server log output directory. Full `jboss` write access required.
`/var/opt/rh/eap8/`	`jboss:jboss`	`770`	Runtime deployment data, temporary caches, and system indexes. Full `jboss` write access required.
`/etc/systemd/system/eap8-standalone.service`	`root:root`	`644`	Systemd unit file. Must strictly be owned by `root:root` for OS integrity.
2.2 Setup Commands
Apply Strict Ownership & Permission Sets:
# 1. Server Binaries: Read-Only for jboss user
sudo chown -R root:jboss /opt/rh/eap8/
sudo chmod -R 755 /opt/rh/eap8/
# 2. Configurations: Read/Write for jboss group
sudo chown -R root:jboss /etc/opt/rh/eap8/
sudo chmod -R 770 /etc/opt/rh/eap8/
# 3. Runtime Data & Logs: Full jboss owner rights
sudo chown -R jboss:jboss /var/log/opt/rh/eap8/
sudo chown -R jboss:jboss /var/opt/rh/eap8/
sudo chmod -R 770 /var/log/opt/rh/eap8/
sudo chmod -R 770 /var/opt/rh/eap8/
Extract Server Migration Tool: Extract the standalone Migration Tool ZIP archive to `/opt/` and assign `root:jboss` ownership:
sudo unzip jboss-server-migration-*.zip -d /opt/
sudo chown -R root:jboss /opt/jboss-server-migration
sudo chmod -R 755 /opt/jboss-server-migration
Stop Source and Target Services:
sudo systemctl stop eap7-standalone 2>/dev/null || true
sudo systemctl stop eap8-standalone
---
3. Server Configuration Migration (Interactive Mode)
3.1 Execute the Tool
Run the script as the `jboss` user without the `--batch` flag:
sudo -u jboss /opt/jboss-server-migration/jboss-server-migration.sh \
--source /opt/jboss-eap-7.4 \
--target /opt/rh/eap8/root/usr/share/wildfly
3.2 Interactive Prompts Walkthrough
During execution, the script prompts for decisions. Pressing `Enter` accepts the default choice inside the brackets `\[y]`.
Prompt Rules:
Press `Enter` or type `y` to accept recommended subsystem updates.
Type `n` if you wish to skip migrating a specific subsystem.
Press `Ctrl + C` to cancel execution.
3.3 Log Locations & Verification
Terminal Output: Displays real-time `INFO`, `WARN`, `ERROR`, and `SUCCESS` indicators.
Migration Log File: Detailed logs are saved at:
/opt/jboss-server-migration/logs/migration.log
Target File Output: Confirm the updated configuration file is written to `/etc`:
ls -la /etc/opt/rh/eap8/wildfly/standalone/configuration/standalone.xml
---
4. Post-Migration Server Configuration
4.1 Management Credentials (Local Fallback)
Add an administrative user to the local Elytron properties realm:
sudo -u jboss /opt/rh/eap8/root/usr/share/wildfly/bin/add-user.sh \
-u admin -p 'YourSecurePassword123!' -g admin
4.2 Port Offset, JVM Customization & OS Limits (Systemd Override)
To prevent patch overwrites during future `dnf update` operations, configure port offsets, JVM options, and system limits using systemd overrides:
sudo systemctl edit eap8-standalone
Add:
[Service]
Environment="WILDFLY_OPTS=-Djboss.socket.binding.port-offset=100"
Environment="JAVA_OPTS=-Xms2g -Xmx4g -XX:+UseG1GC"
LimitNOFILE=65536
LimitNPROC=4096
4.3 Secure Admin Console via Elytron HTTPS (Company-Issued CA)
Phase 1: Generate Private Key & CSR
cd /etc/opt/rh/eap8/wildfly/standalone/configuration/
# 1. Generate 2048-bit RSA Private Key
sudo -u jboss openssl genrsa -out mgmt-server.key 2048
sudo chmod 600 mgmt-server.key
# 2. Generate CSR for PKI submission
sudo -u jboss openssl req -new -key mgmt-server.key -out mgmt-server.csr \
-subj "/C=US/ST=State/L=City/O=App-NameName/OU=IT Infrastructure/CN=jboss-mgmt.App-Name.internal"
Phase 2: Create PKCS12 Bundle (`mgmt.p12`)
Once the signed server certificate (`mgmt-server.crt`), intermediate (`intermediate.crt`), and root CA (`root-ca.crt`) are returned from your internal App-Name PKI:
# Bundle certificate chain
cat mgmt-server.crt intermediate.crt root-ca.crt > mgmt-chain.crt
# Convert to PKCS12 format
sudo -u jboss openssl pkcs12 -export \
-in mgmt-chain.crt \
-inkey mgmt-server.key \
-out mgmt.p12 \
-name mgmt-ssl \
-passout pass:KeystorePass123!
sudo chown root:jboss mgmt.p12
sudo chmod 640 mgmt.p12
Phase 3: Bind to Elytron Management Interface via CLI
sudo -u jboss /opt/rh/eap8/root/usr/share/wildfly/bin/jboss-cli.sh --connect
# 1. Add Keystore
/subsystem=elytron/key-store=mgmtKS:add(path=mgmt.p12, relative-to=jboss.server.config.dir, credential-reference={clear-text="KeystorePass123!"}, type=PKCS12)
# 2. Add Key Manager
/subsystem=elytron/key-manager=mgmtKM:add(key-store=mgmtKS, credential-reference={clear-text="KeystorePass123!"}, alias=mgmt-ssl)
# 3. Add Server SSL Context
/subsystem=elytron/server-ssl-context=mgmtSSLContext:add(key-manager=mgmtKM)
# 4. Attach SSL Context to Management HTTP Interface
/core-service=management/management-interface=http-interface:write-attribute(name=ssl-context, value=mgmtSSLContext)
/core-service=management/management-interface=http-interface:write-attribute(name=secure-socket-binding, value=management-https)
reload
4.4 SELinux Contexts & File Security
Apply SELinux certificates context for RHEL 9/10 compliance:
sudo semanage fcontext -a -t cert_t "/etc/opt/rh/eap8/wildfly/standalone/configuration/.*"
sudo restorecon -Rv /etc/opt/rh/eap8/wildfly/standalone/configuration/
4.5 Active Directory / LDAP Management Authentication
Phase 1: Store AD Bind Account Password in Elytron Store
/subsystem=elytron/credential-store=App-NameCS:add-alias(alias=ad_bind_password, secret-value="ADServiceBindPass#2026")
Phase 2: Configure Directory Context and LDAP Realm
/subsystem=elytron/dir-context=adContext:add( \
url="ldaps://ad.App-Name.internal:636", \
principal="cn=jboss_bind,ou=ServiceAccounts,dc=App-Name,dc=internal", \
credential-reference={store=App-NameCS, alias=ad_bind_password})
/subsystem=elytron/ldap-realm=adRealm:add( \
dir-context=adContext, \
identity-mapping={ \
    rdn-identifier="sAMAccountName", \\
    search-path="ou=AdminUsers,dc=App-Name,dc=internal", \\\\

    attribute-mapping=\\\[{from="memberOf", filter="(\&(objectClass=group)(member={0}))", filter-base-dn="ou=Groups,dc=App-Name,dc=internal", to="roles"}\\] \\\\



})
Phase 3: Create Security Domain & Group Mappings
/subsystem=elytron/security-domain=adMgmtDomain:add( \
default-realm=adRealm, \
realms=[{realm=adRealm}], \
permission-mapper=default-permission-mapper)
/subsystem=elytron/role-mapping-realm=adRoleMap:add( \
realm=adRealm, \
role-map={ \
    "SuperUser"=\["CN=JBoss\_Admins,OU=Groups,DC=App-Name,DC=internal"\] \\

})
Phase 4: Attach Domain to Management HTTP Interface
/subsystem=elytron/http-authentication-factory=management-http-authentication:write-attribute( \
name=security-domain, \
value=adMgmtDomain)
reload
---
4.6 DataSource & IBM MQ Credential Store Integration
Eliminate plain-text credentials in `standalone.xml` by binding DataSources and IBM MQ resource adapters to Elytron Credential Stores.
Step 1: Add Credentials to `elytron.p12`
# 1. Store DataSource DB Password
/subsystem=elytron/credential-store=App-NameCS:add-alias(alias=ds_app_password, secret-value="SecureDbPass#2026")
# 2. Store IBM MQ Service Password
/subsystem=elytron/credential-store=App-NameCS:add-alias(alias=ibmmq_app_pass, secret-value="SecureMqPassword#2026")
Step 2: Configure DataSource Credential Reference in `standalone.xml`
In `/etc/opt/rh/eap8/wildfly/standalone/configuration/standalone.xml`:
<subsystem xmlns="urn:jboss:domain:datasources:7.0">
    \<datasources\>
        \\<datasource jndi-name="java:jboss/datasources/AppDS" pool-name="AppDS" enabled="true"\\>

            \\<connection-url\\>jdbc:oracle:thin:@oracle-db.App-Name.internal:1521/appdb\\</connection-url\\>

            \\<driver\\>oracle\\</driver\\>

            \\<security\\>

                \\<user-name\\>app\\\_user\\</user-name\\>

                \\<credential-reference store="App-NameCS" alias="ds\\\_app\\\_password"/\\>

            \\</security\\>

        \\</datasource\\>

    \\</datasources\\>



</subsystem>
Step 3: Configure IBM MQ Resource Adapter Credential Reference
<subsystem xmlns="urn:jboss:domain:resource-adapters:7.0">
    \<resource-adapters\>
        \\<resource-adapter id="wmq.jms.ra.rar"\\>

            \\<archive\\>wmq.jms.ra.rar\\</archive\\>

            \\<connection-configurations\\>

                \\<connection-definition class-name="com.ibm.mq.connector.outbound.ManagedConnectionFactoryImpl" jndi-name="java:/jms/wmqCF" pool-name="wmqCF"\\>

                    \\<security\\>

                        \\<credential-reference store="App-NameCS" alias="ibmmq\\\_app\\\_pass"/\\>

                    \\</security\\>

                \\</connection-definition\\>

            \\</connection-configurations\\>

        \\</resource-adapter\\>

    \\</resource-adapters\\>



</subsystem>
---
4.7 Switching Between OpenJDK 17 and OpenJDK 21
JBoss EAP 8.1 is fully supported on both OpenJDK 17 and OpenJDK 21. If the development team requests switching Java runtimes:
Install OpenJDK Packages:
sudo dnf install -y java-17-openjdk-headless java-21-openjdk-headless
Configure Service-Specific `JAVA\_HOME` via Systemd Override:
sudo systemctl edit eap8-standalone
For OpenJDK 17:
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
For OpenJDK 21:
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk"

Reload and Restart:
sudo systemctl daemon-reload
sudo systemctl restart eap8-standalone
Verification:
head -n 30 /var/log/opt/rh/eap8/wildfly/standalone/server.log | grep "Java Version"
---
5. Supplement: Third-Party JDBC Driver & IBM MQ Resource Adapter Management
5.1 How It Varies from JBoss EAP 7.4
JBoss EAP 7.4: JDBC driver JARs or `.rar` connector files were often dropped directly into `standalone/deployments/` or manually configured under loose flat directory structures.
JBoss EAP 8.1 (RPM): RPM installations strictly enforce modular isolation and FHS file separation. Binaries and custom modules reside in read-only system paths (`/opt/rh/eap8/...`), while active runtime deployments (`.rar` files) reside in `/var/opt/rh/eap8/...` under the `jboss:jboss` security context. This prevents binary modifications during RPM patch cycles (`dnf update`).
5.2 Part A: Deploying and Configuring a JDBC Driver (e.g., Oracle)
Create Module Directory Structure:
sudo mkdir -p /opt/rh/eap8/root/usr/share/wildfly/modules/system/layers/base/com/oracle/jdbc/main/
Place Driver JAR & Set Permissions:
sudo cp ojdbc11.jar /opt/rh/eap8/root/usr/share/wildfly/modules/system/layers/base/com/oracle/jdbc/main/
sudo chown root:jboss /opt/rh/eap8/root/usr/share/wildfly/modules/system/layers/base/com/oracle/jdbc/main/*
sudo chmod 644 /opt/rh/eap8/root/usr/share/wildfly/modules/system/layers/base/com/oracle/jdbc/main/*
Create `module.xml` Descriptor:
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.9" name="com.oracle.jdbc">
\<resources\>
           \\<resource-root path="ojdbc11.jar"/\\>  
     
       \\</resources\\>  
     
       \\<dependencies\\>  
     
           \\<module name="javax.api"/\\>  
     
           \\<module name="javax.transaction.api"/\\>  
     
       \\</dependencies\\>  
     


</module>
Register in `standalone.xml`:
<subsystem xmlns="urn:jboss:domain:datasources:7.0">
\<datasources\>
           \\<drivers\\>  
     
               \\<driver name="oracle" module="com.oracle.jdbc"\\>  
     
                   \\<driver-class\\>oracle.jdbc.OracleDriver\\</driver-class\\>  
     
               \\</driver\\>  
     
           \\</drivers\\>  
     
       \\</datasources\\>  
     


</subsystem>
---
5.3 Part B: Deploying and Configuring IBM MQ Resource Adapter (`wmq.jms.ra.rar`)
Place `.rar` File in Deployments Directory:
sudo cp wmq.jms.ra.rar /var/opt/rh/eap8/wildfly/standalone/deployments/
sudo chown jboss:jboss /var/opt/rh/eap8/wildfly/standalone/deployments/wmq.jms.ra.rar
sudo chmod 640 /var/opt/rh/eap8/wildfly/standalone/deployments/wmq.jms.ra.rar
Reference the RAR in `standalone.xml`:
<subsystem xmlns="urn:jboss:domain:resource-adapters:7.0">
\<resource-adapters\>
           \\<resource-adapter id="wmq.jms.ra.rar"\\>  
     
               \\<archive\\>wmq.jms.ra.rar\\</archive\\>  
     
               \\<transaction-support\\>XATransaction\\</transaction-support\\>  
     
               \\<connection-configurations\\>  
     
                   \\<connection-definition class-name="com.ibm.mq.connector.outbound.ManagedConnectionFactoryImpl" jndi-name="java:/jms/wmqCF" pool-name="wmqCF" enabled="true"\\>  
     
                       \\<config-property name="channel"\\>SYSTEM.DEF.SVRCONN\\</config-property\\>  
     
                       \\<config-property name="hostName"\\>mq-broker.App-Name.internal\\</config-property\\>  
     
                       \\<config-property name="port"\\>1414\\</config-property\\>  
     
                       \\<config-property name="queueManager"\\>QM\\\_App-Name\\\_PROD\\</config-property\\>  
     
                       \\<config-property name="transportType"\\>CLIENT\\</config-property\\>  
     
                       \\<security\\>  
     
                           \\<credential-reference store="App-NameCS" alias="ibmmq\\\_app\\\_pass"/\\>  
     
                       \\</security\\>  
     
                   \\</connection-definition\\>  
     
               \\</connection-configurations\\>  
     
           \\</resource-adapter\\>  
     
       \\</resource-adapters\\>  
     


</subsystem>
---
6. Service Startup & Post-Migration Verification
Start Service:
sudo systemctl daemon-reload
sudo systemctl enable eap8-standalone
sudo systemctl start eap8-standalone
Verify Service Status:
sudo systemctl status eap8-standalone
Verify RPM Package Integrity:
sudo rpm -V eap8-standalone
