# Instalace a konfigurace Shibboleth a Jetty pro UTIA AV ČR

## Server
Minimální instalace server Rocky 9, 2x CPU 4G RAM
Po instalci standardní kroky pro atyp instalce, update, zákaz selinuxu, instalace potřebných věcí a javy
```
yum -y update
yum -y install java-17-openjdk.x86_64
yum -y install wget
yum -y install openldap-clients
yum -y install firewall-config
yum -y install xauth
alternatives --config java
Type java
eval $(echo "export JAVA_HOME=/usr" | tee -a /root/.bashrc)
```
Napsat disabled do:
```
vi /etc/sysconfig/selinux
```
a reboot.

## Mariadb
Instalace:
```
yum -y install mariadb-server mariadb-java-client mariadb
```
Konfigurace:
```
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
```
Helso na mariadb z předchozí instalace a nebo jiné dle chuti kde kterého soudruha.
Dále je nutné založit databázi shibboleth, uživatele shibboleth, přenést data z předchozí instalace a založit tabulku `StorageRecords`.
Takže na starém serveru se udělá:
```
mysqldump -u root -p shibboleth > ~/persistentIDs.sql
```
Na novém:
```
mysql -u root -p
SET NAMES 'utf8';
SET CHARACTER SET utf8;
CHARSET utf8;
CREATE DATABASE IF NOT EXISTS shibboleth CHARACTER SET=utf8;
USE shibboleth;
CREATE TABLE IF NOT EXISTS `StorageRecords` (
  context varchar(255) COLLATE utf8mb4_bin NOT NULL,
  id varchar(255) COLLATE utf8mb4_bin NOT NULL,
  expires bigint DEFAULT NULL,
  value text NOT NULL,
  version bigint NOT NULL,
  PRIMARY KEY (context, id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
DESCRIBE StorageRecords;
GRANT ALL PRIVILEGES ON shibboleth.*
      TO 'shibboleth'@'localhost'
      IDENTIFIED BY 'SILNE_HESLO';
FLUSH PRIVILEGES;
```
a pak nahrátí starých dat
```
mysql -u root -p shibboleth < ~/persistentIDs.sql
```
**Hack na nefunkční mariadb-java-client, je třeba z nějakého debianu stáhnout mariadb-java-client.jar a připravit si.**
Hotovo

## Jetty
### Instalace jetty z zdroje:
```
cd /opt
mkdir src
cd src
wget -P /opt      https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/11.0.16/jetty-home-11.0.16.tar.gz      https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/11.0.16/jetty-home-11.0.16.tar.gz.asc      https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/11.0.16/jetty-home-11.0.16.tar.gz.sha1
cd /opt
ls
echo -e '\t jetty-home-11.0.16.tar.gz' >> jetty-home-11.0.16.tar.gz.sha1
sha1sum -c jetty-home-11.0.16.tar.gz.sha1
gpg --verify jetty-home-11.0.16.tar.gz.asc
tar -xzf jetty-home-11.0.16.tar.gz
mv jetty-home-11.0.16 jetty11
mkdir -p /opt/jetty11-idp/etc
ls
mv jetty-home-11.0.16.tar.gz  jetty-home-11.0.16.tar.gz.asc  jetty-home-11.0.16.tar.gz.sha1  src
```
### Konfigurace pro potřeby IdP, soubor `/opt/jetty11/modules/` `idp.mod`:
```
[description]
Shibboleth IdP

[depend]
annotations
deploy
ext
http
http2
https
jsp
jstl
plus
requestlog
resources
rewrite
server
servlets
ssl

[files]
tmp/
```
Připraven u cesnetu:
```
wget -P /opt/jetty11/modules     https://www.eduid.cz/jetty/idp.mod
```
### Keystore, převzat z starého IdP, a nebo vygenerován nový:
```
mv keystore /opt/jetty11-idp/etc/keystore.p12
```
nebo:
```
cat cert.pem chain.pem > jetty.txt
openssl pkcs12 -export -inkey key.pem -in jetty.txt -out jetty.pkcs12
mv jetty.pkcs12 /opt/jetty11-idp/etc/keystore.p12
```
### Založení uživatele pro běh:
```
# Vytvoření uživatele jetty
useradd -s /bin/false -d /opt/jetty11 jetty
 
# Změna práv ke keystoru
chmod 640 /opt/jetty11-idp/etc/keystore.p12
chgrp jetty /opt/jetty11-idp/etc/keystore.p12
```

### Soubor `/opt/jetty11-idp/start.d/` `idp.ini`
```
# --------------------------------------- 
# Module: idp
# Shibboleth IdP
# --------------------------------------- 
--module=idp
 
# Allows setting Java system properties (-Dname=value)
# and JVM flags (-X, -XX) in this file
--exec
 
# Newer garbage collector that reduces memory needed for larger metadata files
-XX:+UseG1GC
 
# Maximum amount of memory that Jetty may use
-Xmx1500m
 
# Keystore password
jetty.sslContext.keyStorePassword=FIXME
# Truststore password
jetty.sslContext.trustStorePassword=FIXME
# KeyManager password
jetty.sslContext.keyManagerPassword=FIXME
 
# HTTP
jetty.http.host=127.0.0.1
jetty.http.port=80
 
# HTTPS
jetty.ssl.host=0.0.0.0
jetty.ssl.port=443
 
# Disable SSL renegotiation
jetty.sslContext.renegotiationAllowed=false
 
# Hide Jetty version
jetty.httpConfig.sendServerVersion=false
 
etc/tweak-ssl.xml
```
Možno stáhnout a doplnit hesla:
```
wget -P /opt/jetty11-idp/start.d \
    https://www.eduid.cz/jetty/idp.ini
 
# Otevřeme konfigurační soubor idp.ini
vi /opt/jetty11-idp/start.d/idp.ini
```
### Soubor `/opt/jetty11-idp/` `tweak-ssl.xml`
```
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">
 
<Configure id="sslContextFactory" class="org.eclipse.jetty.util.ssl.SslContextFactory">
 
    <!-- Exclude old and unsafe ciphers -->
    <Call name="addExcludeCipherSuites">
        <Arg>
            <Array type="String">
                <Item>.*DES.*</Item>
                <Item>.*DSS.*</Item>
                <Item>.*MD5.*</Item>
                <Item>.*NULL.*</Item>
                <Item>.*RC4.*</Item>
                <Item>.*_RSA_.*MD5$</Item>
                <Item>.*_RSA_.*SHA$</Item>
                <Item>.*_RSA_.*SHA1$</Item>
                <Item>TLS_DHE_RSA_WITH_AES_128.*</Item>
                <Item>TLS_DHE_RSA_WITH_AES_256.*</Item>
                <Item>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256</Item>
                <Item>TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384</Item>
            </Array>
        </Arg>
    </Call>
 
    <!-- Exclude old and unsafe protocols -->
    <Call name="addExcludeProtocols">
        <Arg>
            <Array type="java.lang.String">
                <Item>SSL</Item>
                <Item>SSLv2</Item>
                <Item>SSLv2Hello</Item>
                <Item>SSLv3</Item>
            </Array>
        </Arg>
    </Call>
 
    <!-- Forward Secrecy -->
    <Set name="IncludeCipherSuites">
        <Array type="String">
            <Item>TLS_ECDHE.*</Item>
        </Array>
    </Set>
 
</Configure>
```
Možno stáhnout
```
# Stažení souboru tweak-ssl.xml
wget -P /opt/jetty11-idp/etc \
    https://www.eduid.cz/jetty/tweak-ssl.xml
```
### Dále soubor `/opt/jetty11-idp/` `idp.xml` 
pozor bude se pak měnit, ale pro začátek dobrý.
```
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Mort Bay Consulting//DTD Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">
 
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <Set name="war"><SystemProperty name="idp.war.path" default="/opt/shibboleth-idp/war/idp.war" /></Set>
    <Set name="contextPath"><SystemProperty name="idp.context.path" default="/idp" /></Set>
    <Set name="extractWAR">false</Set>
    <Set name="copyWebDir">false</Set>
    <Set name="copyWebInf">true</Set>
</Configure>
```
Možno stáhnout
```
wget -P /opt/jetty11-idp/webapps \
    https://www.eduid.cz/jetty/idp.xml
```
### A nakonec vytvoření fake titulní stránky s přesměrováním na něco.
```
# Vytvoření složky pro statický web v Jetty
mkdir -p /opt/jetty11-idp/webapps/root
 
# Vytvoření přesměrování na domovskou stránku
echo '<% response.sendRedirect("https://www.example.org"); %>' > \
    /opt/jetty11-idp/webapps/root/index.jsp
```
### Knihovna pro mariadb
Z místa kam jsme si dali knihovnu ukradenou z debianu ji nakopirujeme do stromu jetty:
```
mkdir -p /opt/jetty11-idp/lib/ext
cp ~/mariadb-java-client.jar \
      /opt/jetty11-idp/lib/ext/mariadb-java-client.jar
```
### Bezpečnost jetty-rewrite.xml
soubor ` /opt/jetty11-idp/etc/` `jetty-rewrite.xml`
```
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">
 
<Configure id="Server" class="org.eclipse.jetty.server.Server">
 
    <Call name="insertHandler">
        <Arg>
            <New class="org.eclipse.jetty.rewrite.handler.RewriteHandler">
 
                <Set name="rewriteRequestURI"><Property name="jetty.rewrite.rewriteRequestURI" deprecated="rewrite.rewriteRequestURI" default="true"/></Set>
                <Set name="rewritePathInfo"><Property name="jetty.rewrite.rewritePathInfo" deprecated="rewrite.rewritePathInfo" default="false"/></Set>
                <Set name="originalPathAttribute"><Property name="jetty.rewrite.originalPathAttribute" deprecated="rewrite.originalPathAttribute" default="requestedPath"/></Set>
 
                <Set name="dispatcherTypes">
                    <Array type="jakarta.servlet.DispatcherType">
                        <Item><Call class="jakarta.servlet.DispatcherType" name="valueOf"><Arg>REQUEST</Arg></Call></Item>
                        <Item><Call class="jakarta.servlet.DispatcherType" name="valueOf"><Arg>ASYNC</Arg></Call></Item>
                    </Array>
                </Set>
 
                <!-- Strict-Transport-Security -->
                <Call name="addRule">
                    <Arg>
                        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
                            <Set name="pattern">*</Set>
                            <Set name="name">Strict-Transport-Security</Set>
                            <Set name="value">max-age=15768000</Set>
                        </New>
                    </Arg>
                </Call>
 
                <!-- X-Content-Type-Options -->
                <Call name="addRule">
                    <Arg>
                        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
                            <Set name="pattern">*</Set>
                            <Set name="name">X-Content-Type-Options</Set>
                            <Set name="value">nosniff</Set>
                        </New>
                    </Arg>
                </Call>
 
                <!-- X-Xss-Protection -->
                <Call name="addRule">
                    <Arg>
                        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
                            <Set name="pattern">*</Set>
                            <Set name="name">X-Xss-Protection</Set>
                            <Set name="value">1; mode=block</Set>
                        </New>
                    </Arg>
                </Call>
 
                <!-- X-Frame-Options -->
                <Call name="addRule">
                    <Arg>
                        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
                            <Set name="pattern">*</Set>
                            <Set name="name">X-Frame-Options</Set>
                            <Set name="value">DENY</Set>
                        </New>
                    </Arg>
                </Call>
 
                <!-- Content-Security-Policy -->
                <Call name="addRule">
                    <Arg>
                        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
                            <Set name="pattern">*</Set>
                            <Set name="name">Content-Security-Policy</Set>
                            <Set name="value">default-src 'self'; style-src 'self'; script-src 'self' 'unsafe-inline'; img-src 'self'; font-src 'self'; frame-ancestors 'none'</Set>
                        </New>
                    </Arg>
                </Call>
 
                <!-- Referrer-Policy -->
                <Call name="addRule">
                    <Arg>
                        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
                            <Set name="pattern">*</Set>
                            <Set name="name">Referrer-Policy</Set>
                            <Set name="value">no-referrer-when-downgrade</Set>
                        </New>
                    </Arg>
                </Call>
 
            </New>
        </Arg>
    </Call>
 
</Configure>
```
Možno stáhnout:
```
wget -O /opt/jetty11-idp/etc/jetty-rewrite.xml \
    https://www.eduid.cz/jetty/jetty11-rewrite.xml
```
### Vytbvoření souboru pro systemd
příkaz `systemctl edit --full --force jetty11.service`
Nakopírovat:
```
[Unit]
Description=Jetty 11 Web Application Server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]

AmbientCapabilities=CAP_NET_BIND_SERVICE

# Configuration
Environment="JETTY_HOME=/opt/jetty11"
Environment="JETTY_STATE=/opt/jetty11-idp/jetty11/jetty.state"
Environment="JETTY_RUN=/opt/jetty11-idp/jetty11"
Environment="JAVA_OPTS=-Djava.awt.headless=true"
EnvironmentFile=-/opt/jetty11/default/jetty11

# Security
User=jetty
Group=jetty
PrivateTmp=yes
NoNewPrivileges=true
WorkingDirectory=/opt/jetty11-idp/
LogsDirectory=jetty11
LogsDirectoryMode=750
ProtectSystem=strict
ReadWritePaths=/opt/jetty11-idp/logs/
ReadWritePaths=/opt/jetty11-idp/jetty11/
TimeoutSec=infinity

# Lifecycle
Type=forking
ExecStart=/opt/jetty11/bin/jetty.sh start
PIDFile=/opt/jetty11-idp/jetty11/jetty.pid
SuccessExitStatus=143
Restart=on-abort

# Logging
SyslogIdentifier=jetty11

[Install]
WantedBy=multi-user.target
```
A praconí adresáře:
```
# Vytvoření složek pro jetty11
mkdir /opt/jetty11-idp/{jetty11,logs}
 
# Upravení práv složek
chown jetty /opt/jetty11-idp/{jetty11,logs}
 
# Vytvoření souboru stavu jetty
touch /opt/jetty11-idp/jetty11/jetty.state
chown jetty:jetty /opt/jetty11-idp/jetty11/jetty.state
```
### Modul rewrite
```
export JETTY_HOME=/opt/jetty11
export JETTY_BASE=/opt/jetty11-idp
cd $JETTY_HOME
java -jar start.jar --add-modules=rewrite
```

### Restart
```
systemctl enable jetty11
systemctl restart jetty11
```

### Test že je vše OK
```
ss -tlpn | fgrep java

LISTEN  0  50  [::ffff:127.0.0.1]:80   *:*  users:(("java",pid=15630,fd=61))                                               
LISTEN  0  50                   *:443  *:*  users:(("java",pid=15630,fd=55))                                               
```
Vyzkoušíme, jestli funguje přesměrování:
```
wget -S -O/dev/null http://localhost
wget -S -O/dev/null https://`hostname -f`
```
### Nastavení firewallu
```
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```
A nebo pomocí `firewall-config`

Hotovo

## Shibboleth IdP
### Stažení Idp ze serveru:
```
wget -P /opt  https://shibboleth.net/downloads/identity-provider/5.1.2/shibboleth-identity-provider-5.1.2.tar.gz https://shibboleth.net/downloads/identity-provider/5.1.2/shibboleth-identity-provider-5.1.2.tar.gz.asc https://shibboleth.net/downloads/identity-provider/5.1.2/shibboleth-identity-provider-5.1.2.tar.gz.sha256
ls
sha256sum shibboleth-identity-provider-5.1.2.tar.gz
ls
more shibboleth-identity-provider-5.1.2.tar.gz.sha256 
ls
mv shibboleth-identity-provider-5.1.2.tar.gz* src/
cd src/
tar -xzvf shibboleth-identity-provider-5.1.2.tar.gz
```
### Pokud se bude dělat multiinstalace tak je nutné do souboru `webapp/WEB-INF/web.xml` doplnit magický kód:
```
<context-param>
    <param-name>idp.home</param-name>
    <param-value>/opt/idp/idp.utia.cas.cz</param-value>
</context-param>
```
Buďto v editoru nebo příkazem:
```
cd shibboleth-identity-provider-5.1.2
sed "/<display-name>Shibboleth Identity Provider.*/a <context-param>\n\t\t<param-name>idp.home<\/param-name>\n\t\t<param-value>/opt/idp/idp.utia.cas.cz<\/param-value>\n\t</context-param>" webapp/WEB-INF/web.xml > webapp/WEB-INF/web.xml.new
mv webapp/WEB-INF/web.xml webapp/WEB-INF/web.xml.bak
cp webapp/WEB-INF/web.xml.new webapp/WEB-INF/web.xml
```
### Spuštění instalace
```
mkdir /opt/idp
./bin/install.sh 
```
Dostaneme zhruma toto: (ipm nahradit necim jinym)
```
Installation Directory: [/opt/shibboleth-idp] ? 
/opt/idp/idp.ipm.cas.cz
INFO  - New Install.  Version: 5.1.2
Host Name: [test.site.cas.cz] ? 
idp.ipm.cas.cz
INFO  - Creating idp-signing, CN = idp.ipm.cas.cz URI = https://idp.ipm.cas.cz/idp/shibboleth, keySize=3072
INFO  - Creating idp-encryption, CN = idp.ipm.cas.cz URI = https://idp.ipm.cas.cz/idp/shibboleth, keySize=3072
INFO  - Creating backchannel keystore, CN = idp.ipm.cas.cz URI = https://idp.ipm.cas.cz/idp/shibboleth, keySize=3072
INFO  - Creating Sealer KeyStore
INFO  - No existing versioning property, initializing...
SAML EntityID: [https://idp.ipm.cas.cz/idp/shibboleth] ? 

Attribute Scope: [ipm.cas.cz] ? 

INFO  - Initializing OpenSAML using the Java Services API
INFO  - Algorithm failed runtime support check, will not be usable: http://www.w3.org/2001/04/xmlenc#ripemd160
INFO  - Algorithm failed runtime support check, will not be usable: http://www.w3.org/2001/04/xmldsig-more#hmac-ripemd160
INFO  - Algorithm failed runtime support check, will not be usable: http://www.w3.org/2001/04/xmldsig-more#rsa-ripemd160
INFO  - Including auto-located properties in /opt/idp/idp.ipm.cas.cz/conf/admin/admin.properties
INFO  - Including auto-located properties in /opt/idp/idp.ipm.cas.cz/conf/authn/authn.properties
INFO  - Including auto-located properties in /opt/idp/idp.ipm.cas.cz/conf/c14n/subject-c14n.properties
INFO  - Including auto-located properties in /opt/idp/idp.ipm.cas.cz/conf/ldap.properties
INFO  - Including auto-located properties in /opt/idp/idp.ipm.cas.cz/conf/saml-nameid.properties
INFO  - Including auto-located properties in /opt/idp/idp.ipm.cas.cz/conf/services.properties
INFO  - Creating Metadata to /opt/idp/idp.ipm.cas.cz/metadata/idp-metadata.xml
INFO  - Rebuilding /opt/idp/idp.ipm.cas.cz/war/idp.war, Version 5.1.2
INFO  - Initial populate from /opt/idp/idp.ipm.cas.cz/dist/webapp to /opt/idp/idp.ipm.cas.cz/webpapp.tmp
INFO  - Overlay from /opt/idp/idp.ipm.cas.cz/edit-webapp to /opt/idp/idp.ipm.cas.cz/webpapp.tmp
INFO  - Creating war file /opt/idp/idp.ipm.cas.cz/war/idp.war
```
### Konfigurace 
přechod do adresáře /opt/idp/idp.utia.cas.cz
### idp.properties
 v `conf/idp.properties` se nastaví proměné pro consent, lokálního HTML úložiště, a zkaz lepších šifer přestože to CESNET nedoporučuje.
 ```
vi conf/idp.properties

idp.consent.StorageService = shibboleth.JPAStorageService
idp.storage.htmlLocalStorage = false
#idp.encryption.config=shibboleth.EncryptionConfiguration.GCM
```


### ldap.properties
```
vi conf/ldap.properties

# LDAP authentication configuration, see authn/ldap-authn-config.xml
# Note, this doesn't apply to the use of JAAS

## Authenticator strategy, either anonSearchAuthenticator, bindSearchAuthenticator, directAuthenticator, adAuthenticator
idp.authn.LDAP.authenticator                   = bindSearchAuthenticator

## Connection properties ##
idp.authn.LDAP.ldapURL                          = ldap://lily.utia.cas.cz
#idp.authn.LDAP.useStartTLS                     = true
#idp.authn.LDAP.useSSL                          = false
# Time in milliseconds that connects will block
#idp.authn.LDAP.connectTimeout                  = PT3S
# Time in milliseconds to wait for responses
#idp.authn.LDAP.responseTimeout                 = PT3S

## SSL configuration, either jvmTrust, certificateTrust, or keyStoreTrust
#idp.authn.LDAP.sslConfig                       = certificateTrust
## If using certificateTrust above, set to the trusted certificate's path
idp.authn.LDAP.trustCertificates                = %{idp.home}/credentials/ldap-server.crt
## If using keyStoreTrust above, set to the truststore path
idp.authn.LDAP.trustStore                       = %{idp.home}/credentials/ldap-server.truststore

## Return attributes during authentication
idp.authn.LDAP.returnAttributes                 = passwordExpirationTime,loginGraceRemaining

## DN resolution properties ##

# Search DN resolution, used by anonSearchAuthenticator, bindSearchAuthenticator
# for AD: CN=Users,DC=example,DC=org
idp.authn.LDAP.baseDN                           = ou=users,dc=utia,dc=cas,dc=cz
#idp.authn.LDAP.subtreeSearch                   = true
idp.authn.LDAP.userFilter                       = (&(uid={user})(o=UTIA))
# bind search configuration
# for AD: idp.authn.LDAP.bindDN=adminuser@domain.com
idp.authn.LDAP.bindDN                           = cn=shibboleth,dc=utia,dc=cas,dc=cz

# Format DN resolution, used by directAuthenticator, adAuthenticator
# for AD use idp.authn.LDAP.dnFormat=%s@domain.com
idp.authn.LDAP.dnFormat                         = uid=%s,ou=people,dc=example,dc=org

# LDAP attribute configuration, see attribute-resolver.xml
# Note, this likely won't apply to the use of legacy V2 resolver configurations
idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
#idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
idp.attribute.resolver.LDAP.searchFilter        = (uid=$resolutionContext.principal)
idp.attribute.resolver.LDAP.connectionStrategy  = %{idp.authn.LDAP.connectionStrategy:ACTIVE_PASSIVE}


# LDAP pool configuration, used for both authn and DN resolution
#idp.pool.LDAP.minSize                          = 3
#idp.pool.LDAP.maxSize                          = 10
#idp.pool.LDAP.validateOnCheckout               = false
#idp.pool.LDAP.validatePeriodically             = true
#idp.pool.LDAP.validatePeriod                   = PT5M
#idp.pool.LDAP.prunePeriod                      = PT5M
#idp.pool.LDAP.idleTime                         = PT10M
#idp.pool.LDAP.blockWaitTime                    = PT3S
#idp.pool.LDAP.failFastInitialize               = false
```
**Je nutné nezapomenout nakopírovat soubor ldap-server.crt na odpovídající místo!**


### secrets.properties
```
 vi credentials/secrets.properties 

idp.authn.LDAP.bindDNCredential = ___HESLO_PRO_BIND_K_LDAP_SERVERU___
idp.persistentId.salt = ___SŮL_PRO_GENEROVÁNÍ_IDENTIFIKÁTORŮ___
```

### metadata-providers.xml
Do souboru `conf/metadata-providers.xml` se přidá následující vkládá se před poslední tag </MetadataProvider>:

```
<!-- eduID.cz -->
<MetadataProvider
    id="eduidcz"
    xsi:type="FileBackedHTTPMetadataProvider"
    backingFile="%{idp.home}/metadata/eduidcz.xml"
    metadataURL="https://metadata.eduid.cz/entities/eduid+sp"
    maxRefreshDelay="PT30M">
 
    <MetadataFilter
        xsi:type="SignatureValidation"
        requireSignedRoot="true"
        certificateFile="%{idp.home}/credentials/metadata.eduid.cz.crt.pem" />
 
    <MetadataFilter
        xsi:type="RequiredValidUntil"
        maxValidityInterval="P30D" />
 
</MetadataProvider>
 
<!-- eduGAIN -->
<MetadataProvider
    id="edugain"
    xsi:type="FileBackedHTTPMetadataProvider"
    backingFile="%{idp.home}/metadata/edugain.xml"
    metadataURL="https://metadata.eduid.cz/entities/edugain+sp"
    maxRefreshDelay="PT30M">
 
    <MetadataFilter
        xsi:type="SignatureValidation"
        requireSignedRoot="true"
        certificateFile="%{idp.home}/credentials/metadata.eduid.cz.crt.pem" />
 
    <MetadataFilter
        xsi:type="RequiredValidUntil"
        maxValidityInterval="P30D" />
 
</MetadataProvider>
```

Dále staneme klíče k podpisu metadat:
```
wget -P credentials \
    https://www.eduid.cz/docs/eduid/metadata/metadata.eduid.cz.crt.pem
```

### attribute-resolver.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<AttributeResolver
    xmlns="urn:mace:shibboleth:2.0:resolver"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="urn:mace:shibboleth:2.0:resolver http://shibboleth.net/schema/idp/shibboleth-attribute-resolver.xsd">

    <!-- ========================================== -->
    <!--      Attribute Definitions                 -->
    <!-- ========================================== -->

<AttributeDefinition id="eduPersonPrincipalName" xsi:type="ScriptedAttribute">
    <InputDataConnector ref="myLDAP"  attributeNames="pager uid"/>
    <Script><![CDATA[
logger = Java.type("org.slf4j.LoggerFactory").getLogger("net.shibboleth.idp.attribute.resolver.eppnbuilder");
scopedValueType =  Java.type("net.shibboleth.idp.attribute.ScopedStringAttributeValue");
var localpart = "";
if (typeof pager == "undefined" || pager.getValues().size() < 1) {
    logger.debug("No pager in LDAP found, creating uid");
    localpart = uid.getValues().get(0);
} else {
    logger.debug("pager in LDAP found");
    localpart = pager.getValues().get(0);
}
eduPersonPrincipalName.addValue(new scopedValueType(localpart, "%{idp.scope}"));
logger.debug("ePPN final value: " + eduPersonPrincipalName.getValues().get(0));
    ]]>
</Script>
</AttributeDefinition>

<!-- mail -->
    <AttributeDefinition xsi:type="Simple" id="mail">
        <InputDataConnector ref="myLDAP" attributeNames="mail"/>
    </AttributeDefinition>

<!-- eduPersonAffiliation -->
    <AttributeDefinition xsi:type="Simple" id="eduPersonAffiliation">
        <InputDataConnector ref="staticAttributes" attributeNames="eduPersonAffiliation"/>
    </AttributeDefinition>

<!-- eduPersonScopedAffiliation -->
    <AttributeDefinition xsi:type="Scoped" id="eduPersonScopedAffiliation" scope="%{idp.scope}">
        <InputDataConnector ref="staticAttributes" attributeNames="eduPersonAffiliation"/>
    </AttributeDefinition>

<!-- givenName -->
    <AttributeDefinition xsi:type="Simple" id="givenName">
        <InputDataConnector ref="myLDAP" attributeNames="givenName"/>
    </AttributeDefinition>

<!-- sn -->
    <AttributeDefinition xsi:type="Simple" id="sn">
        <InputDataConnector ref="myLDAP" attributeNames="sn"/>
    </AttributeDefinition>

<!-- displayName -->
   <AttributeDefinition xsi:type="Template" id="displayName">
      <InputDataConnector ref="myLDAP" attributeNames="givenName sn"/>
        <Template>${givenName} ${sn}</Template>
    </AttributeDefinition>

<!-- commonName -->
    <AttributeDefinition xsi:type="Template" id="cn">
        <InputDataConnector ref="myLDAP" attributeNames="givenName sn"/>
        <Template>${givenName} ${sn}</Template>
    </AttributeDefinition>

<!-- commonName#ASCII pro TCS-P -->
    <AttributeDefinition xsi:type="ScriptedAttribute" id="commonNameASCII">
        <InputAttributeDefinition ref="cn" />
        <Script>
        <![CDATA[
            load("nashorn:mozilla_compat.js");

            importPackage(Packages.edu.internet2.middleware.shibboleth.common.attribute.provider);
            importPackage(Packages.java.lang);
            importPackage(Packages.java.text);

            if(cn.getValues().size() > 0) {
                originalValue = cn.getValues().get(0);
                asciiValue = Normalizer.normalize(originalValue, Normalizer.Form.NFD).replaceAll("\\p{InCombiningDiacriticalMarks}+", "");
                commonNameASCII.getValues().add(asciiValue);
            }
        ]]>
        </Script>
    </AttributeDefinition>

<!-- employeeNumber -->
    <AttributeDefinition xsi:type="Simple" id="employeeNumber">
        <InputDataConnector ref="myLDAP" attributeNames="employeeNumber"/>
    </AttributeDefinition>

<!-- unstructuredName pro TCS-P -->
    <AttributeDefinition xsi:type="Mapped" id="unstructuredName">
        <InputDataConnector ref="myLDAP" attributeNames="employeeNumber"/>
        <ValueMap>
            <ReturnValue>$1</ReturnValue>
            <SourceValue>15(.+)</SourceValue>
        </ValueMap>
    </AttributeDefinition>

<!-- eduPersonUniqueId -->
<AttributeDefinition id="eduPersonUniqueId" xsi:type="Scoped" scope="%{idp.scope}">
    <InputAttributeDefinition ref="unstructuredName" />
</AttributeDefinition>

<!-- uniqueIdentifier -->
    <AttributeDefinition xsi:type="Simple"  id="uniqueIdentifier">
        <InputDataConnector ref="staticAttributes" attributeNames="uniqueIdentifier"/>
    </AttributeDefinition>

<!-- eduPersonEntitlement pro TCS-P -->
   <AttributeDefinition xsi:type="ScriptedAttribute" id="eduPersonEntitlement" >
       <InputAttributeDefinition ref="unstructuredName" />
       <InputAttributeDefinition ref="uniqueIdentifier" />
        <ScriptFile>/opt/idp/common/script/eduPersonEntitlementUtia.js</ScriptFile>
    </AttributeDefinition>

<!-- mail  pro TCS-P -->
    <AttributeDefinition xsi:type="Simple" id="authMail">
        <InputDataConnector ref="myLDAP" attributeNames="mail"/>
    </AttributeDefinition>

<!-- organizationName -->
     <AttributeDefinition id="o" xsi:type="Simple">
        <InputDataConnector ref="staticAttributes" attributeNames="organizationName"/>
     </AttributeDefinition>

<!-- schacHomeOrg -->
    <AttributeDefinition xsi:type="Simple" id="schacHomeOrganization">
        <InputDataConnector ref="staticAttributes" attributeNames="schacHomeOrganization"/>
    </AttributeDefinition>

<!-- eduPersonTargetedID -->
    <AttributeDefinition xsi:type="SAML2NameID" id="eduPersonTargetedID" nameIdFormat="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
        <InputDataConnector ref="myStoredId" attributeNames="storedId" />
        <AttributeEncoder xsi:type="SAML2XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10"/>
    </AttributeDefinition>

<!-- samlPairwiseID-->
    <AttributeDefinition xsi:type="Scoped" id="samlPairwiseID" scope="%{idp.scope}">
        <InputDataConnector ref="computed" attributeNames="computedId"/>
    </AttributeDefinition>

<!-- samlSubjectHash -->
    <AttributeDefinition xsi:type="ScriptedAttribute" id="samlSubjectHash" dependencyOnly="true">
        <InputAttributeDefinition ref="unstructuredName"/>
        <Script>
            <![CDATA[
                if (typeof unstructuredName != "undefined" && unstructuredName != "null" && unstructuredName.getValues().size()) {
                    var digestUtils = Java.type("org.apache.commons.codec.digest.DigestUtils");
                    var saltedHash = digestUtils.sha256Hex(unstructuredName.getValues().get(0) + "%{idp.persistentId.salt}");
                    samlSubjectHash.addValue(saltedHash);
                }
            ]]>
        </Script>
    </AttributeDefinition>

<!-- samlSubjectID-->
    <AttributeDefinition xsi:type="Scoped" id="samlSubjectID" scope="%{idp.scope}">
        <InputAttributeDefinition ref="samlSubjectHash"/>
    </AttributeDefinition>
    

<!-- telephoneNumber -->
    <AttributeDefinition xsi:type="Mapped" id="telephoneNumber">
        <InputDataConnector ref="mySQL" attributeNames="telephoneNumber"/>

        <DefaultValue passThru="true"/>

    <ValueMap>
        <ReturnValue>26605$1</ReturnValue>
        <SourceValue>(.+)</SourceValue>
    </ValueMap>
    </AttributeDefinition>
<!-- 
    <AttributeDefinition xsi:type="Simple" id="telephoneNumber">
        <InputDataConnector ref="myLDAP" attributeNames="telephoneNumber"/>
    </AttributeDefinition>
-->
<!-- uid -->
    <AttributeDefinition xsi:type="Simple" id="uid">
        <InputDataConnector ref="myLDAP" attributeNames="uid"/>
    </AttributeDefinition>

<!-- eduPersonAssurance -->
<AttributeDefinition xsi:type="Simple" id="eduPersonAssurance">
    <InputDataConnector ref="staticAttributes" attributeNames="eduPersonAssurance" />
</AttributeDefinition>



    <!-- ========================================== -->
    <!--      Data Connectors                       -->
    <!-- ========================================== -->

    <!-- Static Connector -->
<DataConnector id="staticAttributes" xsi:type="Static">
    <Attribute id="organizationName">
        <Value>UTIA AV CR v.v.i.</Value>
    </Attribute>
       <Attribute id="eduPersonAffiliation">
            <Value>staff</Value>
            <Value>employee</Value>
            <Value>member</Value>
        </Attribute>
    <Attribute id="schacHomeOrganization">
        <Value>%{idp.scope}</Value>
    </Attribute>
        <Attribute id="uniqueIdentifier">
            <Value>EduID</Value>
        </Attribute>
     <Attribute id="eduPersonAssurance">
        <Value>https://refeds.org/assurance/ID/eppn-unique-no-reassign</Value>
        <Value>https://refeds.org/assurance/IAP/high</Value>
    </Attribute>
</DataConnector>


    <!-- LDAP Connector -->
    <!-- exportAttributes="cesnetEmplID" -->
    <DataConnector id="myLDAP" xsi:type="LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}" 
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
        useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}"
        connectTimeout="%{idp.attribute.resolver.LDAP.connectTimeout}"
        trustFile="%{idp.attribute.resolver.LDAP.trustCertificates}"
        responseTimeout="%{idp.attribute.resolver.LDAP.responseTimeout}"
        connectionStrategy="%{idp.attribute.resolver.LDAP.connectionStrategy}"
        noResultIsError="true"
        multipleResultsIsError="true"
        excludeResolutionPhases="c14n/attribute">
        <FilterTemplate>
            <![CDATA[
                %{idp.attribute.resolver.LDAP.searchFilter}
            ]]>
        </FilterTemplate>
        <ConnectionPool
            minPoolSize="%{idp.pool.LDAP.minSize:3}"
            maxPoolSize="%{idp.pool.LDAP.maxSize:10}"
            blockWaitTime="%{idp.pool.LDAP.blockWaitTime:PT3S}"
            validatePeriodically="%{idp.pool.LDAP.validatePeriodically:true}"
            validateTimerPeriod="%{idp.pool.LDAP.validatePeriod:PT5M}"
            validateDN="%{idp.pool.LDAP.validateDN:}"
            validateFilter="%{idp.pool.LDAP.validateFilter:(objectClass=*)}"
            expirationTime="%{idp.pool.LDAP.idleTime:PT10M}"/>
    </DataConnector>

    <!-- StoredID Data Connector -->
    <DataConnector id="myStoredId"
        xsi:type="StoredId"
        generatedAttributeID="storedId"
        salt="%{idp.persistentId.salt}"
        queryTimeout="0">
        <InputAttributeDefinition ref="%{idp.persistentId.sourceAttribute}"/>
        <BeanManagedConnection>shibboleth.MySQLDataSource</BeanManagedConnection>
    </DataConnector>

    <DataConnector id="computed"
        xsi:type="ComputedId"
        excludeResolutionPhases="c14n/attribute"
        generatedAttributeID="computedId"
        salt="%{idp.persistentId.salt}"
        algorithm="%{idp.persistentId.algorithm:SHA}"
        encoding="%{idp.persistentId.encoding:BASE32}">
        <InputAttributeDefinition ref="unstructuredName"/>
    </DataConnector>

<!-- MySQL Data Connector -->
<DataConnector id="mySQL"
    xsi:type="RelationalDatabase">

<!--   <ApplicationManagedConnection -->
   <SimpleManagedConnection
        jdbcDriver="org.mariadb.jdbc.Driver"
        jdbcURL="jdbc:mysql://localhost:3306/shibboleth"
        jdbcUserName="shibboleth"
        jdbcPassword="XXXXXXX"/>

        <QueryTemplate>
            <![CDATA[
                SELECT * FROM export_telefony WHERE name='$resolutionContext.principal'
            ]]>
        </QueryTemplate>

        <Column columnName="field_linka_value" attributeID="telephoneNumber" />

</DataConnector>



</AttributeResolver>


```
### cesnetAttributes.xml
```
wget -O conf/attributes/cesnetAttributes.xml \
    https://www.eduid.cz/shibboleth-idp/cesnetAttributes.xml
```

### default-rules.xml
Doplníme o řádek `<import resource="cesnetAttributes.xml" />`

a nebo
```
wget -O conf/attributes/default-rules.xml \
    https://www.eduid.cz/shibboleth-idp/default-rules.xml
```

### eduPerson.xml
```
wget -O conf/attributes/eduPerson.xml \
    https://www.eduid.cz/shibboleth-idp/eduPerson.xml
```

### inetOrgPerson.xml
```
wget -O conf/attributes/inetOrgPerson.xml \
    https://www.eduid.cz/shibboleth-idp/inetOrgPerson.xml
```

### attribute-filter.xml
```
wget -O conf/attribute-filter.xml \
    https://www.eduid.cz/shibboleth-idp/attribute-filter.xml
```

### idp-metadata.xml

**Zatím ponechat bez zázahu**


### global.xml
Definice beanu pro přístup do mariadb.

```
vi conf/global.xml

<bean id="shibboleth.MySQLDataSource"
    class="org.apache.commons.dbcp2.BasicDataSource"
    p:driverClassName="org.mariadb.jdbc.Driver"
    p:url="jdbc:mysql://localhost:3306/shibboleth"
    p:username="shibboleth"
    p:password="___SILNE_HESLO___" />
 
    <bean id="JDBCStorageService" parent="shibboleth.JDBCStorageService"
          p:dataSource-ref="shibboleth.MySQLDataSource"
          p:transactionIsolation="4"
          p:retryableErrors="40001"
     />
 
    <bean id="shibboleth.JPAStorageService"
        parent="shibboleth.JDBCStorageService"
        p:cleanupInterval="%{idp.storage.cleanupInterval:PT10M}"
        p:dataSource-ref="shibboleth.MySQLDataSource"/>
```

### saml-nameid.properties
Upravíme a doplníme

```
vi conf/saml-nameid.properties

#idp.persistentId.sourceAttribute = uid
# Nové IdP (BASE32)
#idp.persistentId.encoding = BASE32
# Migrované IdP (BASE64)
idp.persistentId.encoding = BASE64
idp.persistentId.generator = shibboleth.StoredPersistentIdGenerator
idp.persistentId.dataSource = shibboleth.MySQLDataSource
idp.persistentId.sourceAttribute = eduPersonPrincipalName

```

### saml-nameid.xml
odkomentovat v souboru řádek

```
vi conf/saml-nameid.xml

<ref bean="shibboleth.SAML2PersistentGenerator" />
```

### subject-c14n.xml
Odkomentujeme tedy řádek: 
```
vi conf/c14n/subject-c14n.xml

<ref bean="c14n/SAML2Persistent" />
```

### messages_cs.properties
Český překlad hlášek
```
wget https://shibboleth.atlassian.net/wiki/download/attachments/1265631751/messages_cs.properties \
    -O messages/messages_cs.properties
```
## Spuštění
opravíme práva v adresáři 
```
chown jetty {logs,metadata}
chgrp -R jetty {conf,credentials}
chmod -R g+r conf
chmod 750 credentials
chmod 640 credentials/*
```
 V systemd musíme Jetty povolit přístup pro zápis do adresářů s logy a metadaty. 
```
systemctl edit jetty11
```
přidáme
```
[Service]
ReadWritePaths=/opt/idp/idp.utia.cas.cz/logs/
ReadWritePaths=/opt/idp/idp.utia.cas.cz/metadata/
```
Přidání doplňků a potvrdíme dvakrát pomocí y a enter. 
```
./bin/plugin.sh -I net.shibboleth.plugin.storage.jdbc
./bin/plugin.sh -I net.shibboleth.idp.plugin.nashorn
```
### Úprava /opt/jetty11-idp/webapps/idp.xml
Změnit soubor na:

```
vi /opt/jetty11-idp/webapps/idp.xml

<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Mort Bay Consulting//DTD Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">

<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <Set name="war"><SystemProperty name="idp.war.path" default="/opt/idp/idp.utia.cas.cz/war/idp.war" /></Set>
    <Set name="contextPath"><SystemProperty name="idp.context.path" default="/idp" /></Set>
    <Set name="virtualHosts">
     <Array type="java.lang.String">
      <Item>idp2.utia.cas.cz</Item>
      <Item>idp.utia.cas.cz</Item>
     </Array>
     </Set>
    <Set name="extractWAR">false</Set>
    <Set name="copyWebDir">false</Set>
    <Set name="copyWebInf">true</Set>
</Configure>
```


 Zbývá jen znovu načíst konfiguraci pro službu Jetty a tu následně restartovat:

Restart Jetty
```
systemctl daemon-reload
systemctl restart jetty11
```


## Test
V souboru `conf/access-control.xml` odkomentovat blok a změnit jmeno uživatele
```
<entry key="AccessByAdminUser">
  <bean parent="shibboleth.PredicateAccessControl">
    <constructor-arg>
      <bean parent="shibboleth.Conditions.SubjectName" c:collection="#{'jdoe'}" />
    </constructor-arg>
  </bean>
</entry>
```

Potom na URL `https://idp.ipm.cas.cz/idp/profile/admin/hello` je cvičné přihlašovátko

## Dokončení instalace
Je třeba nakopírovat logo do `edit-webapp/images/` a vyplnit položky v `messages/messages_cs.properties` a v `messages/messages.properties`.
Konkrétně:
```
dp.title = UTIA Web Login Service
idp.logo = /images/utia+av.png
idp.logo.alt-text = UTIA Logo
idp.footer = IDP UTIA AV ČR, provozuje IT UTIA AV ČR
root.footer = IDP UTIA AV ČR, provozuje IT UTIA AV ČR
idp.url.password.reset=https://lily.utia.cas.cz/
idp.url.helpdesk=http://www.utia.cas.cz/cs/eduid
```

Dále v `views/login.vm` případně doplnit informační texty či to jinak upravit.
V případě ÚTIA doplněn jen informační text.

## Zálohováni, wazuh a nagios


