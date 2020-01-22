# Instalace a konfigurace Jetty a mysql pro AV ČR verze CENTOS 8 pro ASUCH Win AD
## Instalace CENTOS
### Instalace minimal 4G RAM  4 CPU
Instalace potřebných součástí
```
yum install openldap-clients
yum install mariadb mariadb-server
yum -y install wget
wget http://35.244.242.82/yum/java/el7/x86_64/jdk-8u231-linux-x64.rpm
yum localinstall jdk-8u231-linux-x64.rpm
yum install unzip jce_policy-8.zip
yum install unzip
yum install firewall-config
yum install xauth
yum install net-tools
yum update
```
Do `.bash_profile` přidáme
```
export JAVA_HOME=/usr/java/jdk1.8.0_231-amd64
```


### Java Cryptography Extension (Unlimited Strength Jurisdiction Policy Files)
Po nainstalování Oracle JDK je ještě potřeba doinstalovat tzv. JCE US (Java Cryptography Extension Unlimited Strength), které zajistí možnost využít silnější šifrování.

Stáhneme JCE a umístíme jej do adresáře /opt/src.
```
http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html`
```
#### Rozbalení archivu s JCE US
```
cd /opt
unzip -x src/jce_policy-8.zip
```
Pro „instalaci“ JCE US stačí zkopírovat dva následující JAR (Java ARchive) soubory do Oracle JDK:

#### Instalace JCE US
```
cp UnlimitedJCEPolicyJDK8/US_export_policy.jar jdk1.8.0_77/jre/lib/security/
cp UnlimitedJCEPolicyJDK8/local_policy.jar jdk1.8.0_77/jre/lib/security/
```
Adresář s rozbaleným rozšířením JCE US můžeme nyní smazat.

#### Smazání rozbaleného archivu s JCE US
```
rm -rf UnlimitedJCEPolicyJDK8/
```

## Instalace Jetty 
Založíme uživatele idp.
```
adduser idp                                                                                                         
```
Dále se jetty instaluje následně:

```
wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.24.v20191120/jetty-distribution-9.4.24.v20191120.tar.gz
tar -xzvf jetty-distribution-9.4.24.v20191120.tar.gz
mkdir /opt
tar -xzf ~/jetty-distribution-9.4.24.v20191120.tar.gz
chown -R idp:idp jetty-distribution-9.4.24.v20191120/
ln -snf /opt/jetty-distribution-9.4.24.v20191120/bin/jetty.sh /etc/init.d/jetty
echo "JETTY_HOME=/opt/jetty-distribution-9.4.24.v20191120" > /etc/default/jetty
echo "JETTY_BASE=/opt/jetty" >> /etc/default/jetty
echo "JETTY_USER=idp" >> /etc/default/jetty
echo "JETTY_RUN=/opt/run" >> /etc/default/jetty
mkdir /opt/run
chown idp /opt/run
mdir /opt/jetty
chkconfig --add jetty
cd /opt/jetty
su idp
java -jar /opt/jetty-distribution-9.4.24.v20191120/start.jar    --create-startd --add-to-start=annotations,deploy,ext,http,http2,https,jsp,jstl,plus,requestlog,resources,rewrite,server,servlets,ssl
exit
cd ~
```
Vytvoření keystore
```
openssl rsa -in idp.asuch.serverkey.pem -out idp.asuch.serverkey1.pem -des
cat idp.asuch.cert.pem chain_TERENA_SSL_CA_3.pem >> jetty-cert.txt
openssl pkcs12 -export -inkey idp.asuch.serverkey1.pem -in jetty-cert.txt -out jetty-cert.pkcs12
/usr/java/jdk1.8.0_231-amd64/bin/keytool  -importkeystore -srckeystore jetty-cert.pkcs12 -srcstoretype PKCS12 -destkeystore keystore
```
Dále hesla:
```
java -cp /opt/jetty-distribution-9.4.24.v20191120/lib/jetty-util-9.4.24.v20191120.jar      org.eclipse.jetty.util.security.Password nejaketajneheslo
```
Takto získaný řetězec se zapíše do `/opt/jetty/start.d/ssl.ini` do řádků.
```
## Keystore password
jetty.sslContext.keyStorePassword=OBF:xxxxxxxxxxxxxxxxxxxx

## KeyManager password
jetty.sslContext.keyManagerPassword=OBF:xxxxxxxxxxxxxxxxxxxxx

## Truststore password
jetty.sslContext.trustStorePassword=OBF:xxxxxxxxxxxxxxxxxxxxxx
```
Dále upravíme šifry pro SSL
```
cp ../jetty-distribution-9.4.24.v20191120/etc/jetty-ssl-context.xml etc/
vi etc/jetty-ssl-context.xml
```
Přidáme
```
<Set name="ExcludeCipherSuites">
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
</Array>
</Set>
<Set name="IncludeCipherSuites">
  <Array type="String">
            <Item>TLS_DHE_RSA.*</Item>
            <Item>TLS_ECDHE.*</Item>
  </Array>
</Set>

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
```
Upravíme `start.d/http.ini` tak aby pro http běžel jen na localhostu.
```
jetty.http.host=localhost
```
## Konfigurace firewallu
Pomocí rozhraní `firewall-config` a `firewall-cmd` je třeba definovat trusted zonu (IP rozsah), z té povolit ssh a https. V zoně public povolit  https a zakazat ssh.
Dále je třeba nastavi v obou zónách port forwarding pro https a http. 
```
firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8080
firewall-cmd --zone=public --add-forward-port=port=443:proto=tcp:toport=8443
firewall-cmd --zone=trusted --add-forward-port=port=443:proto=tcp:toport=8443
firewall-cmd --zone=trusted --add-forward-port=port=80:proto=tcp:toport=8080
firewall-cmd --runtime-to-permanent
```
## Knihovny pro práci s databázemi
Do adresáře `/opt/jetty/lib/ext/` se nakopírují knihovny:
```
commons-dbcp2-2.1.1.jar
commons-logging-api-1.1.jar
commons-pool2-2.4.2.jar
mysql-connector-java-5.1.39-bin.jar
```

## Mělo by fungovat `/etc/init.d/jetty start`

# Mysql
## Konfigurace serveru
Start serveru
```
systemctl start mariadb
systemctl enable mariadb
```
Konfigurace root hesla
```
mysql_secure_installation
```
## Konfigurace databáze
```
mysql -u root -p
```
Založení databáze
```
SET NAMES 'utf8';
SET CHARACTER SET utf8;
CHARSET utf8;
CREATE DATABASE IF NOT EXISTS shibboleth CHARACTER SET=utf8;
USE shibboleth;
CREATE TABLE IF NOT EXISTS `shibpid` (
  `localEntity` VARCHAR(255) NOT NULL,
  `peerEntity` VARCHAR(255) NOT NULL,
  `principalName` VARCHAR(255) NOT NULL DEFAULT '',
  `localId` VARCHAR(255) NOT NULL,
  `persistentId` VARCHAR(50) NOT NULL,
  `peerProvidedId` VARCHAR(255) DEFAULT NULL,
  `creationDate` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deactivationDate` TIMESTAMP NULL DEFAULT NULL,
  PRIMARY KEY (localEntity, peerEntity, persistentId)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
Vytvoření uživatele shibboleth, heslo se dá vygenerovat třeba `openssl rand -hex 20`
```
GRANT ALL PRIVILEGES ON shibboleth.*
      TO 'shibboleth'@'localhost'
      IDENTIFIED BY 'xxxxxxxxxxxxxxxxxxxxxxxxxxx';
FLUSH PRIVILEGE
```
Test se dá udělat třeba `mysql -u shibboleth -p shibboleth`.
Poslední věc je přenos dat z minulé instalace. Uloženo pomocí mysqldump.
```
mysql -p -u root shibboleth < ~/shibbolethdb.sql
```



