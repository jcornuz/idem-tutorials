# HOWTO Install and Configure a Shibboleth IdP v3.4.x on Debian-Ubuntu Linux with Apache2 + Jetty9 

<img width="120px" src="https://wiki.idem.garr.it/IDEM_Approved.png" />

## Table of Contents

1. [Hardware Requirements](#hardware-requirements)
2. [Software that will be installed](#software-that-will-be-installed)
3. [Other Requirements](#other-requirements)
4. [Install Instructions](#install-instructions)
   1. [Install software requirements](#install-software-requirements)
   2. [Configure the environment](#configure-the-environment)
   3. [Install Jetty 9 Web Server](#install-jetty-9-web-server)
   4. [Install Shibboleth Identity Provider 3.4.x](#install-shibboleth-identity-provider-v34x)
5. [Configuration Instructions](#configuration-instructions)
   1. [Configure SSL on Apache2 (front-end of Jetty)](#configure-ssl-on-apache2-front-end-of-jetty)
   2. [Configure Jetty](#configure-jetty)
   3. [Configure Shibboleth Identity Provider StorageRecords (User Consent)](#configure-shibboleth-identity-provider-storagerecords-user-consent)
      1. [Default - Not Recommended](#default---not-recommended)
      2. [HTML Local Storage - Recommended](#html-local-storage---recommended)
      3. [JPA Storage Service - using a database](#jpa-storage-service---using-a-database)
   4. [Configure Shibboleth Identity Provider to release the persistent-id](#configure-shibboleth-identity-provider-to-release-the-persistent-id)
      1. [Computed mode - Default & Recommended](#computed-mode---default--recommended)
      2. [Stored Mode - using a database](#stored-mode---using-a-database)
   5. [Configure Logout](#configure-logout)
   6. [Configure the directory (openLDAP or AD) connection](#configure-the-directory-openldap-or-ad-connection)
   7. [Configure the attribute resolver (sample)](#configure-the-attribute-resolver-sample)
   8. [Configure IdP Logging](#configure-idp-logging)
   9. [Translate IdP messages into preferred language](#translate-idp-messages-into-preferred-language)
   10. [Disable SAML1 Deprecated Protocol](#disable-saml1-deprecated-protocol)
   11. [(ONLY FOR IDEM Federation members) Register the IdP on the Federation](#only-for-idem-federation-members-register-the-idp-on-the-federation)
   12. [Configure attribute filter policies for the REFEDS Research and Scholarship and the GEANT Data Protection Code of Conduct Entity Categories](#configure-attribute-filter-policies-for-the-refeds-research-and-scholarship-and-the-geant-data-protection-code-of-conduct-entity-categories)
   13. [(ONLY FOR IDP TRAINING AT CYNET) Register the IdP on the Training Test Federation](#only-for-idp-training-at-cynet-register-the-idp-on-the-training-test-federation)
   14. [(ONLY FOR IDP TRAINING AT CYNET) Configure Attribute Filters to release all attributes to all federation resources](#only-for-idp-training-at-cynet-configure-attribute-filters-to-release-all-attributes-to-all-federation-resources)
   15. [(ONLY FOR IDP TRAINING AT CYNET) Configure Attribute Filters to release recommended attributes for eduGAIN](#only-for-idp-training-at-cynet-configure-attribute-filters-to-release-recommended-attributes-for-edugain)
6. [Appendix A: (ONLY FOR IDEM Federation members) Configure Attribute Filters to release the mandatory attributes to the IDEM Default Resources](#appendix-a-only-for-idem-federation-members-configure-attribute-filters-to-release-the-mandatory-attributes-to-the-idem-default-resources)
7. [Appendix B: (ONLY FOR IDEM Federation members) Configure Attribute Filters to release the mandatory attributes to the IDEM Production Resources](#appendix-b-only-for-idem-federation-members-configure-attribute-filters-to-release-the-mandatory-attributes-to-the-idem-production-resources)
8. [Appendix C: Import metadata from previous IDP v2.x](#appendix-c-import-metadata-from-previous-idp-v2x)
9. [Appendix D: Import persistent-id from a previous database](#appendix-d-import-persistent-id-from-a-previous-database)
10. [Appendix E: Useful logs to find problems](#appendix-e-useful-logs-to-find-problems)
11. [Authors](#authors)

## Hardware Requirements

 * CPU: 2 Core
 * RAM: 4 GB
 * HDD: 20 GB
 * OS: Debian 9 / Debian 10 / Ubuntu 18.04 

## Software that will be installed

 * ca-certificates
 * ntp
 * vim
 * Amazon Corretto 8 JDK (recommended) or OpenJDK (latest)
 * jetty 9.4.x
 * apache2 (>= 2.4)
 * openssl
 * libservlet3.1-java
 * default-mysql-server (if JPAStorageService is used)
 * libmariadb-java (if JPAStorageService is used)
 * libcommons-dbcp-java (if JPAStorageService is used)
 * libcommons-pool-java (if JPAStorageService is used)

## Other Requirements

 * Put HTTPS credentials in the right place:
   * HTTPS Server Certificate (Public Key) inside `/etc/ssl/certs` 
   * HTTPS Server Key (Private Key) inside `/etc/ssl/private`
   * Download TCS CA Cert into `/etc/ssl/certs`
     - `wget -O /etc/ssl/certs/TERENA_SSL_CA_3.pem https://www.terena.org/activities/tcs/repository-g3/TERENA_SSL_CA_3.pem`

## Install Instructions

### Install software requirements

1. Become ROOT:
   * `sudo su -`

2. Change the default mirror to the GARR ones on `/etc/apt/sources.list` (OPTIONAL):
   * `debian.mirror.garr.it` (Debian)
   * `ubuntu.mirror.garr.it` (Ubuntu)
   
3. Update packages:
   * `apt update && apt-get upgrade -y --no-install-recommends`
  
4. Install the required packages:
   * Amazon Corretto OpenJDK (Recommended): 
     * `apt install vim wget ca-certificates openssl apache2 ntp libservlet3.1-java --no-install-recommends`
     * `wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add -`
     * `apt-get install software-properties-common`
     * `add-apt-repository 'deb https://apt.corretto.aws stable main'`
     * `apt-get update; sudo apt-get install -y java-1.8.0-amazon-corretto-jdk`
     * `java -version`

   * OpenJDK: 
     * `apt install vim wget default-jdk ca-certificates openssl apache2 ntp libservlet3.1-java --no-install-recommends`

5. Check that Java is working:
   * `update-alternatives --config java` 
   
   (It will returns "There is only one alternative in link group java (providing /usr/bin/java):" )

### Configure the environment

1. Become ROOT:
   * `sudo su -`
   
2. Be sure that your firewall **is not blocking** the traffic on port **443** and **80** for the IdP server.

3. Set the IdP hostname:
   * `vim /etc/hosts`

     ```bash
     <YOUR SERVER PUBLIC IP ADDRESS> idp.example.org idp
     ```
     (*Replace `idp.example.org` with your IdP Full Qualified Domain Name*)

4. Set the variable `JAVA_HOME` in `/etc/environment`:
   * Set JAVA_HOME:
     * OpenJDK:
       * `vim /etc/environment`

         ```bash
         JAVA_HOME=/usr/lib/jvm/default-java
         ```
         
        * `source /etc/environment`
        * `export JAVA_HOME=/usr/lib/jvm/default-java`

     * Amanzon Corretto OpenJDK:
       * `vim /etc/environment`

         ```bash
         JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto
         ```

       * `source /etc/environment`
       * `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`

5. Configure the right privileges for the SSL Certificate and Key used by HTTPS:
   * `chmod 400 /etc/ssl/private/idp.example.org.key`
   * `chmod 644 /etc/ssl/certs/idp.example.org.crt`

   **(ALTERNATIVE)** Create a self-signed certificate:
   * `openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/private/idp.example.org.key -out /etc/ssl/certs/idp.example.org.crt -nodes -days 1095`

### Install Jetty 9 Web Server

Jetty is a Web server and a Java Servlet container. It will be used to run the IdP application through its WAR file.

1. Become ROOT: 
   * `sudo su -`

2. Download and Extract Jetty:
   * `cd /usr/local/src`
   * `wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.24.v20191120/jetty-distribution-9.4.24.v20191120.tar.gz`
   * `tar xzvf jetty-distribution-9.4.24.v20191120.tar.gz`

3. Create the `jetty-src` folder as a symbolic link. It will be useful for future Jetty updates:
   * `ln -s jetty-distribution-9.4.24.v20191120 jetty-src`

4. Create the user `jetty` that can run the web server:
   * `useradd -r -m jetty`

5. Create your custom Jetty configuration that overrides the default one and will survive upgrades:
   * `mkdir /opt/jetty`
   * `cd /opt/jetty`
   * `vim /opt/jetty/start.ini`

     ```bash
     #===========================================================
     # Jetty Startup
     #
     # Starting Jetty from this {jetty.home} is not recommended.
     #
     # A proper {jetty.base} directory should be configured, instead
     # of making changes to this {jetty.home} directory.
     #
     # See documentation about {jetty.base} at
     # http://www.eclipse.org/jetty/documentation/current/startup.html
     #
     # A demo-base directory has been provided as an example of
     # this sort of setup.
     #
     # $ cd demo-base
     # $ java -jar ../start.jar
     #
     #===========================================================

     # To disable the warning message, comment the following line
     --module=home-base-warning

     # ---------------------------------------
     # Module: ext
     --module=ext

     # ---------------------------------------
     # Module: server
     --module=server

     ### ThreadPool configuration
     ## Minimum number of threads
     jetty.threadPool.minThreads=10

     ## Maximum number of threads
     jetty.threadPool.maxThreads=200

     ## Thread idle timeout (in milliseconds)
     jetty.threadPool.idleTimeout=60000

     ## Response content buffer size (in bytes)
     jetty.httpConfig.outputBufferSize=32768

     ## Max request headers size (in bytes)
     jetty.httpConfig.requestHeaderSize=8192

     ## Max response headers size (in bytes)
     jetty.httpConfig.responseHeaderSize=8192

     ## Whether to send the Server: header
     jetty.httpConfig.sendServerVersion=true

     ## Whether to send the Date: header
     jetty.httpConfig.sendDateHeader=false

     ## Dump the state of the Jetty server, components, and webapps after startup
     jetty.server.dumpAfterStart=false

     ## Dump the state of the Jetty server, components, and webapps before shutdown
     jetty.server.dumpBeforeStop=false

     # ---------------------------------------
     # Module: jsp
     --module=jsp

     # ---------------------------------------
     # Module: resources
     --module=resources

     # ---------------------------------------
     # Module: deploy
     --module=deploy

     # ---------------------------------------
     # Module: jstl
     --module=jstl

     # ---------------------------------------
     # Module: websocket
     --module=websocket

     # ---------------------------------------
     # Module: http
     --module=http

     ### HTTP Connector Configuration

     ## Connector host/address to bind to
     jetty.http.host=localhost

     ## Connector port to listen on
     jetty.http.port=8080

     ## Connector idle timeout in milliseconds
     jetty.http.idleTimeout=30000

     # ---------------------------------------
     # Module: annotations
     --module=annotations

     # Module: console-capture
     --module=console-capture

     jetty.console-capture.dir=/var/log/jetty

     # ---------------------------------------
     # Module: requestlog
     --module=requestlog

     # ---------------------------------------
     # Module: servlets
     --module=servlets

     # ---------------------------------------
     # Module: plus
     --module=plus

     # ---------------------------------------
     # Mwdule: http-forwarded
     --module=http-forwarded
     
     # Allows setting Java system properties (-Dname=value)
     # and JVM flags (-X, -XX) in this file
     # NOTE: spawns child Java process
     --exec

     # Set the IdP home dir
     -Didp.home=/opt/shibboleth-idp

     # Newer garbage collector that reduces memory needed for larger metadata files
     -XX:+UseG1GC
 
     # Maximum amount of memory that Jetty may use, at least 1.5G is recommended
     # for handling larger (> 25M) metadata files but you will need to test on
     # your particular metadata configuration
     -Xmx2000m

     # Prevent blocking for entropy.
     -Djava.security.egd=file:/dev/urandom

     # Set Java tmp location
     -Djava.io.tmpdir=tmp
     
     # Java contains a lot of classes which assume that there is a some sort of display and a keyboard attached. 
     # A code that runs on a server does not have them and this is called Headless mode.
     -Djava.awt.headless=true
     ```

6. Create the TMPDIR directory used by Jetty:
   * `mkdir /opt/jetty/tmp ; chown jetty:jetty /opt/jetty/tmp`
   * `chown -R jetty:jetty /opt/jetty/ /usr/local/src/jetty-src`

7. Create the service loadable from command line:
   * `cd /etc/init.d`
   * `ln -s /usr/local/src/jetty-src/bin/jetty.sh jetty`
   * `update-rc.d jetty defaults`

8. Create the Jetty Log's folder:
   * `mkdir /var/log/jetty`
   * `mkdir /opt/jetty/logs`
   * `chown jetty:jetty /var/log/jetty /opt/jetty/logs`

9. Configure **/etc/default/jetty**:
   * `vim /etc/default/jetty`
  
     ```bash
     JETTY_HOME=/usr/local/src/jetty-src
     JETTY_BASE=/opt/jetty
     JETTY_USER=jetty
     JETTY_START_LOG=/var/log/jetty/start.log
     TMPDIR=/opt/jetty/tmp
     ```

10. Check if all settings are OK:
    * `service jetty check`   (Jetty NOT running)
    * `service jetty start`
    * `service jetty check`   (Jetty running pid=XXXX)
  
    If you get an error likes "*Job for jetty.service failed because the control process exited with error code. See "systemctl status jetty.service" and "journalctl -xe" for details.*", try this: 
      * `rm /var/run/jetty.pid`
      * `systemctl start jetty.service`

### Install Shibboleth Identity Provider v3.4.x

The Identity Provider (IdP) is responsible for user authentication and providing user information to the Service Provider (SP). It is located at the home organization, which is the organization which maintains the user's account.
It is a Java Web Application that can be deployed with its WAR file.

1. Become ROOT:
   * `sudo su -`

2. Download the Shibboleth Identity Provider v3.4.x (replace '3.4.x' with the latest version found [here](https://shibboleth.net/downloads/identity-provider/)):
   * `cd /usr/local/src`
   * `wget http://shibboleth.net/downloads/identity-provider/3.4.x/shibboleth-identity-provider-3.4.x.tar.gz`
   * `tar -xzvf shibboleth-identity-provider-3.4.x.tar.gz`

3. Run the installer `install.sh`:
   * `cd /usr/local/src/shibboleth-identity-provider-3.4.x`
   * `./bin/install.sh`
  
     ```bash
     root@idp:/usr/local/src/shibboleth-identity-provider-3.4.x# ./bin/install.sh
     Source (Distribution) Directory: [/usr/local/src/shibboleth-identity-provider-3.4.x]
     Installation Directory: [/opt/shibboleth-idp]
     Hostname: [localhost.localdomain]
     idp.example.org
     SAML EntityID: [https://idp.example.org/idp/shibboleth]
     Attribute Scope: [localdomain]
     example.org
     Backchannel PKCS12 Password: ###PASSWORD-FOR-BACKCHANNEL###
     Re-enter password:           ###PASSWORD-FOR-BACKCHANNEL###
     Cookie Encryption Key Password: ###PASSWORD-FOR-COOKIE-ENCRYPTION###
     Re-enter password:              ###PASSWORD-FOR-COOKIE-ENCRYPTION###
     ```

     From this point on the variable **idp.home** refers to the directory: `/opt/shibboleth-idp`

4. Make the **jetty** user able to access the IdP main directories:
   * `cd /opt/shibboleth-idp`
   * `chown -R jetty logs/ metadata/ credentials/ conf/ system/ war/`

## Configuration Instructions

### Configure SSL on Apache2 (front-end of Jetty)

The Apache HTTP Server will be configured as a reverse proxy and it will be used for SSL offloading.

1. Become ROOT:
   * `sudo su -`

2. Create the DocumentRoot:
   * `mkdir /var/www/html/idp.example.org`
   * `sudo chown -R www-data: /var/www/html/idp.example.org`

3. Create the Virtualhost file `/etc/apache2/sites-available/idp.example.org.conf` and set the content as follows:

   ```apache
   <VirtualHost *:80>
      ServerName "idp.example.org"
      Redirect permanent "/" "https://idp.example.org/"
   </VirtualHost>

   <IfModule mod_ssl.c>
      SSLStaplingCache shmcb:/var/run/ocsp(128000)
      <VirtualHost _default_:443>
        ServerName idp.example.org:443
        ServerAdmin admin@example.org
        DocumentRoot /var/www/html/idp.example.org
        
        SSLEngine On
        
        SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
        SSLCipherSuite "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4"

        SSLHonorCipherOrder on

        # Disable SSL Compression
        SSLCompression Off
        
        # OCSP Stapling, only in httpd/apache >= 2.3.3
        SSLUseStapling          on
        SSLStaplingResponderTimeout 5
        SSLStaplingReturnResponderErrors off
        
        # Enable HTTP Strict Transport Security with a 2 year duration
        Header always set Strict-Transport-Security "max-age=63072000;includeSubDomains;preload"
        
        SSLCertificateFile /etc/ssl/certs/idp.example.org.crt
        SSLCertificateKeyFile /etc/ssl/private/idp.example.org.key
        SSLCACertificateFile /etc/ssl/certs/TERENA_SSL_CA_3.pem

        <IfModule mod_proxy.c>
           ProxyPreserveHost On
           RequestHeader set X-Forwarded-Proto "https"
           ProxyPass /idp http://localhost:8080/idp retry=5
           ProxyPassReverse /idp http://localhost:8080/idp retry=5

           <Location /idp>
              Require all granted
           </Location>
        </IfModule>
      </VirtualHost>
   </IfModule>

   # This virtualhost is only here to handle administrative commands for Shibboleth, executed from localhost
   <VirtualHost 127.0.0.1:80>
     ProxyPass /idp http://localhost:8080/idp retry=5
     ProxyPassReverse /idp http://localhost:8080/idp retry=5
     <Location /idp>
       Require all granted
     </Location>
   </VirtualHost>
   ```

4. Enable the required Apache2 modules and the virtual hosts:
   * `a2enmod proxy_http ssl headers alias include negotiation`
   * `a2ensite idp.example.org.conf`
   * `a2dissite 000-default.conf`
   * `systemctl restart apache2.service`

5. Check if the Apache Welcome page is available:
    * http://idp.example.org

6. Verify the quality and the strength of the SSL configuration:
   * [**https://www.ssllabs.com/ssltest/analyze.html**](https://www.ssllabs.com/ssltest/analyze.html)

### Configure Jetty

1. Become ROOT: 
   * `sudo su -`

2. Configure the IdP Context Descriptor:
   * `mkdir /opt/jetty/webapps`
   * `vim /opt/jetty/webapps/idp.xml`

     ```bash
     <Configure class="org.eclipse.jetty.webapp.WebAppContext">
       <Set name="war"><SystemProperty name="idp.home"/>/war/idp.war</Set>
       <Set name="contextPath">/idp</Set>
       <Set name="extractWAR">false</Set>
       <Set name="copyWebDir">false</Set>
       <Set name="copyWebInf">true</Set>
       <Set name="persistTempDirectory">false</Set>
     </Configure>
     ```

3. Restart Jetty:
   * `systemctl restart jetty.service`

4. Check that IdP metadata is available on:
   * https://idp.example.org/idp/shibboleth

5. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

### Configure Shibboleth Identity Provider StorageRecords (User Consent)

*Shibboleth Documentation reference*: https://wiki.shibboleth.net/confluence/display/IDP30/StorageConfiguration

> The IdP provides a number of general-purpose storage facilities that can be used by core subsystems like session management and consent. 

#### Default - Not Recommended

If you don't change anything, the IdP stores data in a long-lived browser cookie that can contain an extremely small number of records. This could bring problems in the long term period.

#### HTML Local Storage - Recommended

> It requires JavaScript be enabled because reading and writing to the client requires an explicit page be rendered.
> Note that this feature is safe to enable globally. The implementation is written to check for this capability in each client, and to back off to cookies.

1. Become ROOT: 
   * `sudo su -`

2. Enable HTML Local Storage and security for cookies:
   * `vim /opt/shibboleth-idp/conf/idp.properties`
   
     ```bash
     idp.storage.htmlLocalStorage = true
     idp.cookie.secure = true
     ```
3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

4. Check that the metadata is available on:
   * https://idp.example.org/idp/shibboleth

5. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

#### JPA Storage Service - using a database

This Storage service will memorize User Consent data on persistent database SQL.

1. Become ROOT: 
   * `sudo su -`

2. Rebuild the IdP with the needed libraries:
   * `apt install default-mysql-server libmariadb-java libcommons-dbcp-java libcommons-pool-java --no-install-recommends`
   * `cd /opt/shibboleth-idp`
   * `ln -s /usr/share/java/mariadb-java-client.jar edit-webapp/WEB-INF/lib`
   * `ln -s /usr/share/java/commons-dbcp.jar edit-webapp/WEB-INF/lib`
   * `ln -s /usr/share/java/commons-pool.jar edit-webapp/WEB-INF/lib`
   * `bin/build.sh`

3. Create the `StorageRegords` table on the `storageservice` database.
   * `vim shib-ss-db.sql`:

     ```bash
     SET NAMES 'utf8';

     SET CHARACTER SET utf8;

     CREATE DATABASE IF NOT EXISTS storageservice CHARACTER SET=utf8;

     GRANT ALL PRIVILEGES ON storageservice.* TO root@localhost IDENTIFIED BY '##ROOT-DB-PASSWORD-CHANGEME##';
     GRANT ALL PRIVILEGES ON storageservice.* TO ##USERNAME-CHANGEME##@localhost IDENTIFIED BY '##USER-PASSWORD-CHANGEME##';

     FLUSH PRIVILEGES;

     USE storageservice;

     CREATE TABLE IF NOT EXISTS StorageRecords
     (
     context VARCHAR(255) NOT NULL,
     id VARCHAR(255) NOT NULL,
     expires BIGINT(20) DEFAULT NULL,
     value LONGTEXT NOT NULL,
     version BIGINT(20) NOT NULL,
     PRIMARY KEY (context, id)
     );

     quit
     ```
     
    * (OPTIONAL) MySQL DB Access without password:
      * `vim /root/.my.cnf`

       ```bash
       [client]
       user=root
       password=##ROOT-DB-PASSWORD-CHANGEME##
       ```

   * `mysql -u root < shib-ss-db.sql`
   * `systemctl restart mysql.service`

4. Enable JPA Storage Service:
   * `vim /opt/shibboleth-idp/conf/global.xml` 
     and add the following directives to the tail, just before the **`</beans>`** tag (**IMPORTANT** remeber to modify the "**##USERNAME-CHANGEME##**" and "**##USER-PASSWORD-CHANGEME##**" with your DB user and password):

     ```bash
     <!-- Add bean to store User Consent data on StorageRecords database -->

     <bean id="storageservice.JPAStorageService" class="org.opensaml.storage.impl.JPAStorageService"
           p:cleanupInterval="%{idp.storage.cleanupInterval:PT10M}"
           c:factory-ref="storageservice.JPAStorageService.entityManagerFactory"/>

     <bean id="storageservice.JPAStorageService.entityManagerFactory"
           class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
           <property name="packagesToScan" value="org.opensaml.storage.impl"/>
           <property name="dataSource" ref="storageservice.JPAStorageService.DataSource"/>
           <property name="jpaVendorAdapter" ref="storageservice.JPAStorageService.JPAVendorAdapter"/>
           <property name="jpaDialect">
             <bean class="org.springframework.orm.jpa.vendor.HibernateJpaDialect" />
           </property>
     </bean>

     <bean id="storageservice.JPAStorageService.DataSource"
           class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close" lazy-init="true"
           p:driverClassName="org.mariadb.jdbc.Driver"
           p:url="jdbc:mysql://localhost:3306/storageservice?autoReconnect=true"
           p:username="##USERNAME-CHANGEME##"
           p:password="##USER-PASSWORD-CHANGEME##"
           p:maxActive="10"
           p:maxIdle="5"
           p:maxWait="15000"
           p:testOnBorrow="true"
           p:validationQuery="select 1"
           p:validationQueryTimeout="5" />

     <bean id="storageservice.JPAStorageService.JPAVendorAdapter" class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
         <property name="database" value="MYSQL" />
     </bean>
     ```

5. Set the consent storage service to the JPA storage service:
   * `vim /opt/shibboleth-idp/conf/idp.properties`

     ```xml
     idp.consent.StorageService = storageservice.JPAStorageService
     ```
	 
6. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

7. Check that the metadata is available on:
   * https://idp.example.org/idp/shibboleth

8. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

### Configure Shibboleth Identity Provider to release the persistent-id

**Shibboleth Documentation reference** https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration

SAML 2.0 (but not SAML 1.x) defines a kind of NameID called a "persistent" identifier that every SP receives for the IdP users.
This part will teach you how to release the "persistent" identifiers with a database (Stored Mode) or without it (Computed Mode).

By default, a transient NameID will always be released to the Service Provider if the persistent one is not requested.

#### Computed mode - Default & Recommended

1. Become ROOT: 
   * `sudo su -`

2. Enable the generation of the computed `persistent-id` with:
   * `vim /opt/shibboleth-idp/conf/saml-nameid.properties`
     (the *sourceAttribute* MUST be an attribute, or a list of comma-separated attributes, that uniquely identify the subject of the generated `persistent-id`. The sourceAttribute MUST be **Stable**, **Permanent** and **Not-reassignable** attribute.)

     ```xml
     idp.persistentId.sourceAttribute = uid
     ...
     idp.persistentId.salt = ### result of 'openssl rand -base64 36'###
     ...
     # BASE64 will match Shibboleth V2 values, we recommend BASE32 encoding for new installs.
     idp.persistentId.encoding = BASE32
     ```

   * `vim /opt/shibboleth-idp/conf/saml-nameid.xml`
     * Remove the comment for the line:

       ```xml
       <ref bean="shibboleth.SAML2PersistentGenerator" />
       ```

3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

4. Check that the metadata is available on:
   * https://idp.example.org/idp/shibboleth

5. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

#### Stored mode - using a database

1. Become ROOT: 
   * `sudo su -`

2. Rebuild the IdP with the required libraries:
   * `apt install default-mysql-server libmariadb-java libcommons-dbcp-java libcommons-pool-java --no-install-recommends`
   * `cd /opt/shibboleth-idp`
   * `ln -s /usr/share/java/mariadb-java-client.jar edit-webapp/WEB-INF/lib`
   * `ln -s /usr/share/java/commons-dbcp.jar edit-webapp/WEB-INF/lib`
   * `ln -s /usr/share/java/commons-pool.jar edit-webapp/WEB-INF/lib`
   * `bin/build.sh`

3. Create `shibpid` table on `shibboleth` database.
   * `vim shib-pid-db.sql`:

     ```bash
     SET NAMES 'utf8';

     SET CHARACTER SET utf8;

     CREATE DATABASE IF NOT EXISTS shibboleth CHARACTER SET=utf8;

     GRANT ALL PRIVILEGES ON shibboleth.* TO root@localhost IDENTIFIED BY '##ROOT-DB-PASSWORD-CHANGEME##';
     GRANT ALL PRIVILEGES ON shibboleth.* TO ##USERNAME-CHANGEME##@localhost IDENTIFIED BY '##USER-PASSWORD-CHANGEME##';

     FLUSH PRIVILEGES;

     USE shibboleth;

     CREATE TABLE IF NOT EXISTS shibpid
     (
     localEntity VARCHAR(1024) NOT NULL,
     peerEntity VARCHAR(1024) NOT NULL,
     persistentId VARCHAR(50) NOT NULL,
     principalName VARCHAR(255) NOT NULL,
     localId VARCHAR(255) NOT NULL,
     peerProvidedId VARCHAR(255) NULL,
     creationDate TIMESTAMP NOT NULL default CURRENT_TIMESTAMP on update CURRENT_TIMESTAMP,
     deactivationDate TIMESTAMP NULL default NULL,
     PRIMARY KEY (localEntity, peerEntity, persistentId)
     );

     quit
     ```

    * (OPTIONAL) MySQL DB Access without password:
      * `vim /root/.my.cnf`

       ```bash
       [client]
       user=root
       password=##ROOT-DB-PASSWORD-CHANGEME##
       ```
       
   * `mysql -u root < shib-pid-db.sql`
   * `systemctl restart mysql.service`

4. Enable Persistent Identifier's store:
   * `vim /opt/shibboleth-idp/conf/global.xml` 
     and add the following directives to the tail, just before the **`</beans>`** tag (**IMPORTANT** remeber to modify the "**##USERNAME-CHANGEME##**" and "**##USER-PASSWORD-CHANGEME##**" with your DB user and password):

     ```bash
     <!-- Add bean to store persistent-id on shibboleth database -->

     <bean id="shibboleth.JPAStorageService.DataSource"
           class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close" lazy-init="true"
           p:driverClassName="org.mariadb.jdbc.Driver"
           p:url="jdbc:mysql://localhost:3306/shibboleth?autoReconnect=true"
           p:username="##USERNAME-CHANGEME##"
           p:password="##USER-PASSWORD-CHANGEME##"
           p:maxActive="10"
           p:maxIdle="5"
           p:maxWait="15000"
           p:testOnBorrow="true"
           p:validationQuery="select 1"
           p:validationQueryTimeout="5" />
     ```

5. Enable the generation of the `persistent-id`:
   * `vim /opt/shibboleth-idp/conf/saml-nameid.properties`
   (the *sourceAttribute* MUST be an attribute, or a list of comma-separated attributes, that uniquely identify the subject of the generated `persistent-id`. The sourceAttribute MUST be **Stable**, **Permanent** and **Not-reassignable** attribute.)
   
     ```xml
     idp.persistentId.sourceAttribute = uid
     ...
     idp.persistentId.salt = ### result of 'openssl rand -base64 36'###
     ...
     idp.persistentId.generator = shibboleth.StoredPersistentIdGenerator
     ...
     idp.persistentId.dataSource = shibboleth.JPAStorageService.DataSource
     ...
     # BASE64 will match Shibboleth V2 values, we recommend BASE32 encoding for new installs.
     idp.persistentId.encoding = BASE32
     ```

   * Enable the **SAML2PersistentGenerator**:
     * `vim /opt/shibboleth-idp/conf/saml-nameid.xml`
       * Remove the comment from the line containing:

         ```xml
         <ref bean="shibboleth.SAML2PersistentGenerator" />
         ```

     * `vim /opt/shibboleth-idp/conf/c14n/subject-c14n.xml`
       * Remove the comment to the bean called "**c14n/SAML2Persistent**".
       
     * `vim /opt/shibboleth-idp/conf/c14n/simple-subject-c14n-config.xml` (OPTIONAL)
       * Transform each letter of username, before storing in into the database, to Lowercase or Uppercase by setting the proper constant.

6. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

7. Check that the metadata is available on:
   * https://idp.example.org/idp/shibboleth

8. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

### Configure Logout

1. Become ROOT: 
   * `sudo su -`

2. Enable Shibboleth Logout:
   * `vim /opt/shibboleth-idp/conf/idp.properties`
   
     ```bash
     idp.session.trackSPSessions = true
     idp.session.secondaryServiceIndex = true
     ```

3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

4. Check that the metadata is available on:
   * https://idp.example.org/idp/shibboleth

5. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

### Configure the Directory (openLDAP or AD) Connection

1. Check that you can reach the Directory from your IDP server:
   * For Active Directory: `ldapsearch -x -h <LDAP-SERVER-FQDN-OR-IP> -D 'CN=idpuser,CN=Users,DC=ad,DC=example,DC=org' -w '<IDPUSER-PASSWORD>' -b "CN=Users,DC=ad,DC=example,DC=org"`
   * For OpenLDAP: `ldapsearch -x -h <LDAP-SERVER-FQDN-OR-IP> -D 'cn=idpuser,ou=system,dc=example,dc=org' -w '<IDPUSER-PASSWORD>' -b "ou=people,dc=example,dc=org"`

2. Connect the openLDAP to the IdP to allow the authentication of the users:
   * `vim /opt/shibboleth-idp/conf/ldap.properties`

     (with **TLS** solutions we consider to have the LDAP certificate into `/opt/shibboleth-idp/credentials/ldap-server.crt`).

     * For OpenLDAP:
       * Solution 1: LDAP + STARTTLS:

         ```properties
         idp.authn.LDAP.authenticator = bindSearchAuthenticator
         idp.authn.LDAP.ldapURL = ldap://ldap.example.org:389
         idp.authn.LDAP.useStartTLS = true
         idp.authn.LDAP.useSSL = false
         idp.authn.LDAP.sslConfig = certificateTrust
         idp.authn.LDAP.trustCertificates = %{idp.home}/credentials/ldap-server.crt
         idp.authn.LDAP.returnAttributes = ### List comma-separated of attributes to retrieve from OpenLDAP ###
         idp.authn.LDAP.baseDN = ou=people,dc=example,dc=org
         idp.authn.LDAP.userFilter = (&(uid={user})(objectClass=inetOrgPerson))
         idp.authn.LDAP.bindDN = cn=idpuser,ou=system,dc=example,dc=org
         idp.authn.LDAP.bindDNCredential = ###IDPUSER_PASSWORD###
         idp.authn.LDAP.searchFilter = (uid=$resolutionContext.principal)

         # LDAP attribute configuration, see attribute-resolver.xml
         # Note, this likely won't apply to the use of legacy V2 resolver configurations
         idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
         idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
         idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
         idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
         idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
         idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
         idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
         idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
         idp.attribute.resolver.LDAP.searchFilter        = %{idp.authn.LDAP.searchFilter:undefined}
         idp.attribute.resolver.LDAP.returnAttributes    = %{idp.authn.LDAP.returnAttributes:undefined}
         ```

       * Solution 2: LDAP + TLS:

         ```properties
         idp.authn.LDAP.authenticator = bindSearchAuthenticator
         idp.authn.LDAP.ldapURL = ldaps://ldap.example.org:636
         idp.authn.LDAP.useStartTLS = false
         idp.authn.LDAP.useSSL = true
         idp.authn.LDAP.sslConfig = certificateTrust
         idp.authn.LDAP.trustCertificates = %{idp.home}/credentials/ldap-server.crt
         idp.authn.LDAP.returnAttributes = ### List comma-separated of attributes to retrieve from OpenLDAP ###
         idp.authn.LDAP.baseDN = ou=people,dc=example,dc=org
         idp.authn.LDAP.userFilter = (&(uid={user})(objectClass=inetOrgPerson))
         idp.authn.LDAP.bindDN = cn=idpuser,ou=system,dc=example,dc=org
         idp.authn.LDAP.bindDNCredential = ###IDPUSER_PASSWORD###
         idp.authn.LDAP.searchFilter = (uid=$resolutionContext.principal)

         # LDAP attribute configuration, see attribute-resolver.xml
         # Note, this likely won't apply to the use of legacy V2 resolver configurations
         idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
         idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
         idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
         idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
         idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
         idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
         idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
         idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
         idp.attribute.resolver.LDAP.searchFilter        = %{idp.authn.LDAP.searchFilter:undefined}
         idp.attribute.resolver.LDAP.returnAttributes    = %{idp.authn.LDAP.returnAttributes:undefined}
         ```

       * Solution 3: plain LDAP

         ```properties
         idp.authn.LDAP.authenticator = bindSearchAuthenticator
         idp.authn.LDAP.ldapURL = ldap://ldap.example.org:389
         idp.authn.LDAP.useStartTLS = false
         idp.authn.LDAP.useSSL = false
         idp.authn.LDAP.returnAttributes = ### List comma-separated of attributes to retrieve from OpenLDAP ###
         idp.authn.LDAP.baseDN = ou=people,dc=example,dc=org
         idp.authn.LDAP.userFilter = (&(uid={user})(objectClass=inetOrgPerson))
         idp.authn.LDAP.bindDN = cn=idpuser,ou=system,dc=example,dc=org
         idp.authn.LDAP.bindDNCredential = ###IDPUSER_PASSWORD###
         idp.authn.LDAP.searchFilter = (uid=$resolutionContext.principal)

         # LDAP attribute configuration, see attribute-resolver.xml
         # Note, this likely won't apply to the use of legacy V2 resolver configurations
         idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
         idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
         idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
         idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
         idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
         idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
         idp.attribute.resolver.LDAP.searchFilter        = %{idp.authn.LDAP.searchFilter:undefined}
         idp.attribute.resolver.LDAP.returnAttributes    = %{idp.authn.LDAP.returnAttributes:undefined}
         ```

     * For Active Directory:
       * Solution 1: AD + STARTTLS:

         ```properties
         idp.authn.LDAP.authenticator = bindSearchAuthenticator
         idp.authn.LDAP.ldapURL = ldap://ldap.example.org:389
         idp.authn.LDAP.useStartTLS = true
         idp.authn.LDAP.useSSL = false
         idp.authn.LDAP.sslConfig = certificateTrust
         idp.authn.LDAP.trustCertificates = %{idp.home}/credentials/ldap-server.crt
         idp.authn.LDAP.returnAttributes = ### List comma-separated of attributes to retrieve from AD ###
         idp.authn.LDAP.baseDN = CN=Users,DC=ad,DC=example,DC=org
         idp.authn.LDAP.userFilter = (sAMAccountName={user})
         idp.authn.LDAP.bindDN = CN=idpuser,CN=Users,DC=ad,DC=example,DC=org
         idp.authn.LDAP.bindDNCredential = ###IDPUSER_PASSWORD###

         idp.authn.LDAP.searchFilter = (sAMAccountName=$resolutionContext.principal)

         # LDAP attribute configuration, see attribute-resolver.xml
         # Note, this likely won't apply to the use of legacy V2 resolver configurations
         idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
         idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
         idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
         idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
         idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
         idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
         idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
         idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
         idp.attribute.resolver.LDAP.searchFilter        = %{idp.authn.LDAP.searchFilter:undefined}
         idp.attribute.resolver.LDAP.returnAttributes    = %{idp.authn.LDAP.returnAttributes:undefined}
         ```

       * Solution 2: AD + TLS:

         ```properties
         idp.authn.LDAP.authenticator = bindSearchAuthenticator
         idp.authn.LDAP.ldapURL = ldaps://ldap.example.org:636
         idp.authn.LDAP.useStartTLS = false
         idp.authn.LDAP.useSSL = true
         idp.authn.LDAP.sslConfig = certificateTrust
         idp.authn.LDAP.trustCertificates = %{idp.home}/credentials/ldap-server.crt
         idp.authn.LDAP.returnAttributes = ### List comma-separated of attributes to retrieve from AD ###
         idp.authn.LDAP.baseDN = CN=Users,DC=ad,DC=example,DC=org
         idp.authn.LDAP.userFilter = (sAMAccountName={user})
         idp.authn.LDAP.bindDN = CN=idpuser,CN=Users,DC=ad,DC=example,DC=org
         idp.authn.LDAP.bindDNCredential = ###IDPUSER_PASSWORD###
         idp.authn.LDAP.searchFilter = (sAMAccountName=$resolutionContext.principal)

         # LDAP attribute configuration, see attribute-resolver.xml
         # Note, this likely won't apply to the use of legacy V2 resolver configurations
         idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
         idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
         idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
         idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
         idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
         idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
         idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
         idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
         idp.attribute.resolver.LDAP.searchFilter        = %{idp.authn.LDAP.searchFilter:undefined}
         idp.attribute.resolver.LDAP.returnAttributes    = %{idp.authn.LDAP.returnAttributes:undefined}
         ```

       * Solution 3: plain AD

         ```properties
         idp.authn.LDAP.authenticator = bindSearchAuthenticator
         idp.authn.LDAP.ldapURL = ldap://ldap.example.org:389
         idp.authn.LDAP.useStartTLS = false
         idp.authn.LDAP.useSSL = false
         idp.authn.LDAP.returnAttributes = ### List comma-separated of attributes to retrieve from AD ###
         idp.authn.LDAP.baseDN = CN=Users,DC=ad,DC=example,DC=org
         idp.authn.LDAP.userFilter = (sAMAccountName={user})
         idp.authn.LDAP.bindDN = CN=idpuser,CN=Users,DC=ad,DC=example,DC=org
         idp.authn.LDAP.bindDNCredential = ###IDPUSER_PASSWORD###
         idp.authn.LDAP.searchFilter = (sAMAccountName=$resolutionContext.principal)

         # LDAP attribute configuration, see attribute-resolver.xml
         # Note, this likely won't apply to the use of legacy V2 resolver configurations
         idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
         idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
         idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
         idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
         idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
         idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
         idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
         idp.attribute.resolver.LDAP.searchFilter        = %{idp.authn.LDAP.searchFilter:undefined}
         idp.attribute.resolver.LDAP.returnAttributes    = %{idp.authn.LDAP.returnAttributes:undefined}
         ```

       If you decide to use the Solution 3, remove or comment the following directives from your Attribute Resolver file:

       ```xml
       Line 1:  useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}"
       Line 2:  trustFile="%{idp.attribute.resolver.LDAP.trustCertificates}"
       ```

       **UTILITY FOR OPENLDAP ADMINISTRATOR:**
         * `slapcat | grep dn`
           * the baseDN ==> `ou=people,dc=example,dc=org` (branch containing the registered users)
           * the bindDN ==> `cn=idpuser,ou=system,dc=example,dc=org` (distinguished name for the user that can made queries on the LDAP)

#### Configure the attribute resolver (sample)

1. Define which attributes your IdP can manage into your Attribute Resolver file. Here you can find a sample **attribute-resolver-sample.xml** as example:
    * Download the sample attribute resolver provided:
      * For OpenLDAP use: `wget https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-resolver-sample.xml -O /opt/shibboleth-idp/conf/attribute-resolver-sample.xml`
      * For AD use: `wget https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-resolver-AD-sample.xml -O /opt/shibboleth-idp/conf/attribute-resolver-sample.xml`
    
    * Configure the right owner/group:
      `chown jetty:jetty /opt/shibboleth-idp/conf/attribute-resolver-sample.xml`

    * Modify `services.xml` file:
      `vim /opt/shibboleth-idp/conf/services.xml`

      ```xml
      <value>%{idp.home}/conf/attribute-resolver.xml</value>
      ```

      must become:

      ```xml
      <!-- <value>%{idp.home}/conf/attribute-resolver.xml</value> -->
      <value>%{idp.home}/conf/attribute-resolver-sample.xml</value>
      ```
   
      (ONLY FOR IDEM Federation MEMBERS) The following sample is provided by IDEM GARR AAI:
      * OpenLDAP: https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-resolver-idem-ldap-sample.xml
      * Active Directory: https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-resolver-idem-ad-sample.xml

        **Pay attention on `<DataConnector id="myStoredId"` when you use the Stored Mode (database) to manage your persistentIDs. You have to put the right bean ID into `<BeanManagedConnection>` or IdP will not work. You have to put there the ID of the `BasicDataSource` bean**

2. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

3. Check to be able to retrieve transient NameID for an user:
   * `export JAVA_HOME=/usr/lib/jvm/default-java`
   * `cd /opt/shibboleth-idp/bin`
   * `./aacli.sh -n <USERNAME> -r https://sp.example.org/shibboleth --saml2`

4. Check IdP Status:
   * If OpenJDK is chosen: `export JAVA_HOME=/usr/lib/jvm/default-java`
   * If Amazon Corretto JDK is chosen: `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto`   
   * `cd /opt/shibboleth-idp/bin`
   * `./status.sh`

### Configure IdP Logging

Enrich IDP logs with the authentication error occurred on LDAP:
   * ```bash
     sed -i '/^    <logger name="org.ldaptive".*/a \\n    <!-- Logs on LDAP user authentication - ADDED -->' /opt/shibboleth-idp/conf/logback.xml
     ```

   * ```bash
     sed -i '/^    <!-- Logs on LDAP user authentication - ADDED -->/a \ \ \ \ \<logger name="org.ldaptive.auth.Authenticator" level="INFO" />' /opt/shibboleth-idp/conf/logback.xml
     ```

### Translate IdP messages into preferred language

Translate the IdP messages in your language:
   * Get the files translated in your language from [Shibboleth page](https://wiki.shibboleth.net/confluence/display/IDP30/MessagesTranslation)
   * Put 'messages_XX.properties' downloaded file into `/opt/shibboleth-idp/messages` directory
   * Restart IdP to apply the changes: `touch /opt/jetty/webapps/idp.xml`

### Disable SAML1 Deprecated Protocol

1. Modify the IdP metadata to enable only the SAML2 protocol:
   * `vim /opt/shibboleth-idp/metadata/idp-metadata.xml`

      ```bash
      <EntityDescriptor> Section:
        – Remove `validUntil` XML attribute.
	
      <IDPSSODescriptor> Section:
        - From the list of "protocolSupportEnumeration" remove:
          - urn:oasis:names:tc:SAML:1.1:protocol
          - urn:mace:shibboleth:1.0

        - Remove the endpoint:
          <ArtifactResolutionService Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAP-binding" Location="https://idp.example.org:8443/idp/profile/SAML1/SOAP/ArtifactResolution" index="1"/>
          (and modify the index value of the next one to “1”)

        - Remove comment from SingleLogoutService endpoints

        - Between the last <SingleLogoutService> and the first <SingleSignOnService> endpoints add these 2 lines:
          <NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:transient</NameIDFormat>
          <NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>

          (because the IdP installed with this guide will release transient NameID, by default, and persistent NameID if requested.)

        - Remove the endpoint: 
          <SingleSignOnService Binding="urn:mace:shibboleth:1.0:profiles:AuthnRequest" Location="https://idp.example.org/idp/profile/Shibboleth/SSO"/>

        - Remove all ":8443" from the existing URL (such port is not used anymore)

      <AttributeAuthorityDescriptor> Section:
        - From the list "protocolSupportEnumeration" remove the value:
          - urn:oasis:names:tc:SAML:1.1:protocol
          and leave:
          - urn:oasis:names:tc:SAML:2.0:protocol

        - Remove the endpoint: 
          <AttributeService Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAP-binding" Location="https://idp.example.org:8443/idp/profile/SAML1/SOAP/AttributeQuery"/>

        - Remove the comment from:
          <AttributeService Binding="urn:oasis:names:tc:SAML:2.0:bindings:SOAP" Location="https://idp.example.org/idp/profile/SAML2/SOAP/AttributeQuery"/>

        - Remove all ":8443" from the existing URL (such port is not used anymore)
      ```

2. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

3. Verify your IdP metadata on:
   *  `https://idp.example.org/idp/shibboleth`

### (ONLY FOR IDEM Federation members) Register the IdP on the Federation

1. Register you IdP metadata on IDEM Entity Registry (your entity have to be approved by an IDEM Federation Operator before become part of IDEM Test Federation):
   * `https://registry.idem.garr.it/`

2. Configure the IdP to retrieve the Federation Metadata:
   * `cd /opt/shibboleth-idp/conf`
   * `vim metadata-providers.xml`

     ```xml
     <!-- Piece of code to add before the last </MetadataProvider> -->

     <!-- IDEM Test Federation -->
     <MetadataProvider
        id="URLMD-IDEM-Federation"
        xsi:type="FileBackedHTTPMetadataProvider"
        backingFile="%{idp.home}/metadata/idem-test-metadata-sha256.xml"
        metadataURL="http://md.idem.garr.it/metadata/idem-test-metadata-sha256.xml">

        <!--
            Verify the signature on the root element of the metadata aggregate
            using a trusted metadata signing certificate.
        -->
        <MetadataFilter xsi:type="SignatureValidation" requireSignedRoot="true" certificateFile="${idp.home}/metadata/federation-cert.pem"/>   

        <!--
            Require a validUntil XML attribute on the root element and
            make sure its value is no more than 10 days into the future.
        -->
        <MetadataFilter xsi:type="RequiredValidUntil" maxValidityInterval="P10D"/>
   
        <!-- Consume all SP metadata in the aggregate -->
        <MetadataFilter xsi:type="EntityRoleWhiteList">
          <RetainedRole>md:SPSSODescriptor</RetainedRole>
        </MetadataFilter>
     </MetadataProvider>
     ```

   * Retrieve the Federation Certificate used to verify signed metadata:
     *  `wget https://md.idem.garr.it/certs/idem-signer-20220121.pem -O /opt/shibboleth-idp/metadata/federation-cert.pem`

   * Check the validity:
     *  `cd /opt/shibboleth-idp/metadata`
     *  `openssl x509 -in federation-cert.pem -fingerprint -sha1 -noout`
       
        (sha1: D1:68:6C:32:A4:E3:D4:FE:47:17:58:E7:15:FC:77:A8:44:D8:40:4D)
     *  `openssl x509 -in federation-cert.pem -fingerprint -md5 -noout`

        (md5: 48:3B:EE:27:0C:88:5D:A3:E7:0B:7C:74:9D:24:24:E0)
  
3. Reload service with id `shibboleth.MetadataResolverService` to retrieve the Federation Metadata:
    *  `cd /opt/shibboleth-idp/bin`
    *  `./reload-service.sh -id shibboleth.MetadataResolverService`

4. One hour after the IDEM Federation Operators approval your entity on IDEM Entity Registry, check if you can login with your IdP on the following services:
    * https://sp-demo.aai-test.garr.it/secure   (Service Provider provided for testing the IDEM Test Federation)
    * https://sp24-test.garr.it/secure (Service Provider provided for testing the IDEM Test Federation and IDEM Production Federation)

    or check which attributes are released to one the above SP with AACLI:

    * `cd /opt/shibboleth-idp/bin`
    * `./aacli.sh -n <USERNAME> -r https://sp24-test.garr.it/shibboleth --saml2`

### Configure attribute filter policies for the REFEDS Research and Scholarship and the GEANT Data Protection Code of Conduct Entity Categories

1. Download the attribute filter file:
   * `wget -O /opt/shibboleth-idp/conf/attribute-filter-v3-RS-CoCo.xml https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-filter-v3-RS-CoCo.xml`

2. Modify your `services.xml`:
   * `vim /opt/shibboleth-idp/conf/services.xml`

     ```xml
     <util:list id ="shibboleth.AttributeFilterResources">
         <!-- <value>%{idp.home}/conf/attribute-filter.xml</value> -->
         <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>
      </util:list>
      ```

3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

### (ONLY FOR IDP TRAINING AT CYNET) Register the IdP on the Training Test Federation

1. Send your IdP Metadata URL to [marco.malavolti@garr.it](mailto:marco.malavolti@garr.it)

2. Configure the IdP to retrieve the Federation Metadata:
   * `cd /opt/shibboleth-idp/conf`
   * `vim metadata-providers.xml`

     ```xml
     <!-- Piece of code to add before the last </MetadataProvider> -->

     <!-- Training Test Test Federation -->
     <MetadataProvider
        id="URLMD-Training-Federation"
        xsi:type="FileBackedHTTPMetadataProvider"
        backingFile="%{idp.home}/metadata/training-test-federation-sha256.xml"
        metadataURL="https://registry-test.idem.garr.it/rr3/signedmetadata/federation/cyprusIDPtraining/metadata.xml">

        <!--
            Verify the signature on the root element of the metadata aggregate
            using a trusted metadata signing certificate.
        -->
        <MetadataFilter xsi:type="SignatureValidation" requireSignedRoot="true" certificateFile="${idp.home}/metadata/federation-cert.pem"/>   

        <!--
            Require a validUntil XML attribute on the root element and
            make sure its value is no more than 10 days into the future.
        -->
        <MetadataFilter xsi:type="RequiredValidUntil" maxValidityInterval="P10D"/>
   
        <!-- Consume all SP metadata in the aggregate -->
        <MetadataFilter xsi:type="EntityRoleWhiteList">
          <RetainedRole>md:SPSSODescriptor</RetainedRole>
        </MetadataFilter>
     </MetadataProvider>
     ```

   * Retrieve the Federation Certificate used to verify signed metadata:
     *  `wget https://registry-test.idem.garr.it/rr3/signedmetadata/federation/cyprusIDPtraining/metadata-signer.crt -O /opt/shibboleth-idp/metadata/federation-cert.pem`

   * Check the validity:
     *  `cd /opt/shibboleth-idp/metadata`
     *  `openssl x509 -in federation-cert.pem -fingerprint -sha1 -noout`
       
        (sha1: 76:18:EA:B6:D4:9D:0C:C0:A0:16:FD:C0:2D:7C:B1:CD:44:26:B6:94)
     *  `openssl x509 -in federation-cert.pem -fingerprint -md5 -noout`

        (md5: 01:59:50:95:83:26:F2:8D:BA:50:9D:10:30:1D:19:3A)
  
3. Wait that your IdP Metadata is approved and reload service `shibboleth.MetadataResolverService` to refresh Training Test Federation metadata:
    *  `cd /opt/shibboleth-idp/bin`
    *  `./reload-service.sh -id shibboleth.MetadataResolverService`

4. Check which attributes are released to test SP with AACLI:
    * `cd /opt/shibboleth-idp/bin`
    * `./aacli.sh -n <USERNAME> -r https://geanttraining.cynet.ac.cy/sp-garr --saml2`

### (ONLY FOR IDP TRAINING AT CYNET) Configure Attribute Filters to release all attributes to all federation resources

1. Download sample Attribute Filter file:
   * `wget -O /opt/shibboleth-idp/conf/attribute-filter-v3-all.xml https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-filter-v3-all.xml`

2. Modify your `services.xml`:
   * `vim /opt/shibboleth-idp/conf/services.xml`

     ```xml
     ...
     <util:list id ="shibboleth.AttributeFilterResources">
         <!-- <value>%{idp.home}/conf/attribute-filter.xml</value> -->
         <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>
         <value>%{idp.home}/conf/attribute-filter-v3-all.xml</value>
     </util:list>
     ```

3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

### (ONLY FOR IDP TRAINING AT CYNET) Configure Attribute Filters to release recommended attributes for eduGAIN

1. Download sample Attribute Filter file:
   * `wget -O /opt/shibboleth-idp/conf/attribute-filter-v3-eduGAIN.xml https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/utils/attribute-filter-v3-eduGAIN.xml`

2. Modify your `services.xml`:
   * `vim /opt/shibboleth-idp/conf/services.xml`

     ```xml
     <util:list id ="shibboleth.AttributeFilterResources">
         <!-- <value>%{idp.home}/conf/attribute-filter.xml</value> -->
         <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>		 
         <value>%{idp.home}/conf/attribute-filter-v3-all.xml</value>
         <value>%{idp.home}/conf/attribute-filter-v3-eduGAIN.xml</value>
      </util:list>
      ```

3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

### Appendix A: (ONLY FOR IDEM Federation members) Configure Attribute Filters to release the mandatory attributes to the IDEM Default Resources

1. Become ROOT:
   * `sudo su -`

2. Make sure to you have "`tmp/httpClientCache`" used by "`shibboleth.FileCachingHttpClient`":
   * `mkdir -p /opt/shibboleth-idp/tmp/httpClientCache ; chown jetty /opt/shibboleth-idp/tmp/httpClientCache`

3. Modify your `services.xml`:
   * `vim /opt/shibboleth-idp/conf/services.xml`
     * Without a Proxy Server:
       ```xml
       <bean id="IDEM-Default-Filter" class="net.shibboleth.ext.spring.resource.FileBackedHTTPResource"
             c:client-ref="shibboleth.FileCachingHttpClient"
             c:url="https://registry.idem.garr.it/idem-conf/shibboleth/IDP3/attribute-filter-v3-idem.xml"
             c:backingFile="%{idp.home}/conf/attribute-filter-v3-idem.xml"/>
          
       <util:list id ="shibboleth.AttributeFilterResources">
           <value>%{idp.home}/conf/attribute-filter.xml</value>
           <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>
           <ref bean="IDEM-Default-Filter"/>
       </util:list>
       ```

     * With a Proxy Server:
       ```xml
       <bean id="MyHTTPClient" parent="shibboleth.FileCachingHttpClient"
             p:connectionTimeout="PT30S"
             p:connectionRequestTimeout="PT30S"
             p:socketTimeout="PT30S"
             p:cacheDirectory="%{idp.home}/configcache"
             p:connectionProxyHost="<PROXY.FQDN>"
             p:connectionProxyUsername="<USERNAME>"
             p:connectionProxyPassword="<PASSWORD>"
             p:connectionProxyPort="<NUM_PORTA>" />

       <bean id="IDEM-Default-Filter" class="net.shibboleth.ext.spring.resource.FileBackedHTTPResource"
             c:client-ref="MyHTTPClient"
             c:url="https://registry.idem.garr.it/idem-conf/shibboleth/IDP3/attribute-filter-v3-idem.xml"
             c:backingFile="/opt/shibboleth-idp/conf/attribute-filter-v3-idem.xml"/>
	     
       <util:list id ="shibboleth.AttributeFilterResources">
           <value>%{idp.home}/conf/attribute-filter.xml</value>
           <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>
           <ref bean="IDEM-Default-Filter"/>
       </util:list>
       ```

4. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

### Appendix B: (ONLY FOR IDEM Federation members) Configure Attribute Filters to release the mandatory attributes to the IDEM Production Resources

1. Make sure that you have the "`tmp/httpClientCache`" used by "`shibboleth.FileCachingHttpClient`":
   * `mkdir -p /opt/shibboleth-idp/tmp/httpClientCache ; chown jetty /opt/shibboleth-idp/tmp/httpClientCache`

2. Modify your `services.xml`:
   * `vim /opt/shibboleth-idp/conf/services.xml`
     * Without a Proxy Server:
       ```xml
       <bean id="IDEM-Production-Filter" class="net.shibboleth.ext.spring.resource.FileBackedHTTPResource"
             c:client-ref="shibboleth.FileCachingHttpClient"
             c:url="https://registry.idem.garr.it/idem-conf/shibboleth/IDP3/attribute-filter-v3-required.xml"
             c:backingFile="%{idp.home}/conf/attribute-filter-v3-required.xml"/>
       
       <util:list id ="shibboleth.AttributeFilterResources">
           <value>%{idp.home}/conf/attribute-filter.xml</value>
           <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>
           <ref bean="IDEM-Default-Filter"/>
           <ref bean="IDEM-Production-Filter"/>
       </util:list>
       ```
      
     * With a Proxy Server:
       ```xml
       <bean id="MyHTTPClient" parent="shibboleth.FileCachingHttpClient"
             p:connectionTimeout="PT30S"
             p:connectionRequestTimeout="PT30S"
             p:socketTimeout="PT30S"
             p:cacheDirectory="%{idp.home}/configcache"
             p:connectionProxyHost="<PROXY.FQDN>"
             p:connectionProxyUsername="<USERNAME>"
             p:connectionProxyPassword="<PASSWORD>"
             p:connectionProxyPort="<NUM_PORTA>" />

       <bean id="IDEM-Production-Filter" class="net.shibboleth.ext.spring.resource.FileBackedHTTPResource"
             c:client-ref="MyHTTPClient"
             c:url="https://registry.idem.garr.it/idem-conf/shibboleth/IDP3/attribute-filter-v3-required.xml"
             c:backingFile="%{idp.home}/conf/attribute-filter-v3-required.xml"/>
       
       <util:list id ="shibboleth.AttributeFilterResources">
           <value>%{idp.home}/conf/attribute-filter.xml</value>
           <value>%{idp.home}/conf/attribute-filter-v3-RS-CoCo.xml</value>
           <ref bean="IDEM-Default-Filter"/>
           <ref bean="IDEM-Production-Filter"/>
       </util:list>
       ```

3. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`

### Appendix C: Import metadata from previous IDP v2.x ###

1. Store into /tmp directory the following files:
   * `idp-metadata.xml`
   * `idp.crt`
   * `idp.key`

2. Follow the steps on your IdP v3.x:
   * `sudo su -`
   * `mv /tmp/idp-metadata.xml /opt/shibboleth-idp/metadata`
   * `mv /tmp/idp.crt /tmp/idp.key /opt/shibboleth-idp/credentials`
   * `cd /opt/shibboleth-idp/credentials/`
   * `rm idp-encryption.crt idp-backchannel.crt idp-encryption.key idp-signing.crt idp-signing.key`
   * `ln -s idp.crt idp-encryption.crt`
   * `ln -s idp.key idp-encryption.key`
   * `ln -s idp.key idp-signing.key`
   * `ln -s idp.crt idp-signing.crt`
   * `ln -s idp.crt idp-backchannel.crt`
   * `openssl pkcs12 -export -in idp-encryption.crt -inkey idp-encryption.key -out idp-backchannel.p12 -password pass:#YOUR.BACKCHANNEL.CERT.PASSWORD#`

3. Check that *idp.entityID* property value on *idp.properties* file is equal to the previous entityID value into *idp-metadata.xml*.

4. Modify IdP metadata to enable only the SAML2 protocol:
   * `vim /opt/shibboleth-idp/metadata/idp-metadata.xml`
 
      ```bash
      <EntityDescriptor> Section:
        – Remove `validUntil` XML attribute.

      <IDPSSODescriptor> Section:
        – From the list of "protocolSupportEnumeration" remove:
          - urn:oasis:names:tc:SAML:1.1:protocol
          - urn:mace:shibboleth:1.0

        – Remove the endpoint:
          <ArtifactResolutionService Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAP-binding" Location="https://idp.example.org:8443/idp/profile/SAML1/SOAP/ArtifactResolution" index="1"/>
          (and modify the index value of the next one to “1”)

        - Between the last <SingleLogoutService> and the first <SingleSignOnService> endpoints add these 2 lines:
          <NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:transient</NameIDFormat>
          <NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>

          (because the IdP installed with this guide will release transient, by default, and persistent NameID if requested.)

        - Remove the endpoint: 
          <SingleSignOnService Binding="urn:mace:shibboleth:1.0:profiles:AuthnRequest" Location="https://idp.example.org/idp/profile/Shibboleth/SSO"/>
       
        - Remove all ":8443" from the existing URL (such port is not used anymore)
	
        - Remove comment from each <SingleLogoutService> endpoint

      <AttributeAuthorityDescriptor> Section:
        - From the list "protocolSupportEnumeration" remove the value:
          - urn:oasis:names:tc:SAML:1.1:protocol
          and leave
          - urn:oasis:names:tc:SAML:2.0:protocol

        - Remove the endpoint: 
          <AttributeService Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAP-binding" Location="https://idp.example.org:8443/idp/profile/SAML1/SOAP/AttributeQuery"/>

        - Remove the comment from:
          <AttributeService Binding="urn:oasis:names:tc:SAML:2.0:bindings:SOAP" Location="https://idp.example.org/idp/profile/SAML2/SOAP/AttributeQuery"/>

        - Remove all ":8443" from the existing URL (such port is not used anymore)
      ```

5. Restart IdP to apply the changes:
   * `touch /opt/jetty/webapps/idp.xml`
  
6. Don't forget to update your IdP Metadata on [IDEM Entity Registry](https://registry.idem.garr.it/rr3) to apply changes on the federation IDEM! For any help write to idem-help@garr.it

### Appendix D: Import persistent-id from a previous database

1. Become ROOT:
   * `sudo su -`

2. Create a DUMP of `shibpid` table from the previous DB `shibboleth` on the OLD IdP:
   * `cd /tmp`
   * `mysqldump --complete-insert --no-create-db --no-create-info -u root -p shibboleth shibpid > /tmp/shibboleth_shibpid.sql`

3. Move the `/tmp/shibboleth_shibpid.sql` of old IdP into `/tmp/shibboleth_shibpid.sql` on the new IdP.
 
4. Import the content of `/tmp/shibboleth_shibpid.sql` into the DB of the new IDP:
   * `cd /tmp ; mysql -u root -p shibboleth < /tmp/shibboleth_shibpid.sql`

5. Delete `/tmp/shibboleth_shibpid.sql`:
   * `rm /tmp/shibboleth_shibpid.sql`
   
### Appendix E: Useful logs to find problems

1. Jetty Logs:
   * `cd /opt/jetty/logs`
   * `ls -l *.stderrout.log`

2. Shibboleth IdP Logs:
   * `cd /opt/shibboleth-idp/logs`
   * **Audit Log:** `vim idp-audit.log`
   * **Consent Log:** `vim idp-consent-audit.log`
   * **Warn Log:** `vim idp-warn.log`
   * **Process Log:** `vim idp-process.log`

### Authors

#### Original Author

 * Marco Malavolti (marco.malavolti@garr.it)
