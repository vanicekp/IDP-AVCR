# Instalace a konfigurace Shibboleth a Jetty pro UTIA AV ČR

## Server
Minimální instalace Rocky 9, 2x CPU 4G RAM
Po instalci standardní kroky pro atyp instalce, uodate, zákaz selinuxu, instalace potřebných věcí a javy
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
Instalace jetty z zdroje:
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
Konfigurace pro potřeby IdP, soubor `/opt/jetty11/modules/` `idp.mod`:
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
Keystore, převzat z starého IdP, a nebo vygenerován nový:
```
mv keystore /opt/jetty11-idp/etc/keystore.p12
```
nebo:
```
cat cert.pem chain.pem > jetty.txt
openssl pkcs12 -export -inkey key.pem -in jetty.txt -out jetty.pkcs12
mv jetty.pkcs12 /opt/jetty11-idp/etc/keystore.p12
```
Založení uživatele pro běh:
```
# Vytvoření uživatele jetty
useradd -s /bin/false -d /opt/jetty11 jetty
 
# Změna práv ke keystoru
chmod 640 /opt/jetty11-idp/etc/keystore.p12
chgrp jetty /opt/jetty11-idp/etc/keystore.p12
```

Soubor `/opt/jetty11-idp/start.d/` `idp.ini`
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
Soubor `/opt/jetty11-idp/` `tweak-ssl.xml`
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
Dále soubor `/opt/jetty11-idp/` `idp.xml` pozor bude se pak měnit, ale pro začátek dobrý.
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
A nakonec vytvoření fake titulní stránky s přesměrováním na něco.
```
# Vytvoření složky pro statický web v Jetty
mkdir -p /opt/jetty11-idp/webapps/root
 
# Vytvoření přesměrování na domovskou stránku
echo '<% response.sendRedirect("https://www.example.org"); %>' > \
    /opt/jetty11-idp/webapps/root/index.jsp
```


