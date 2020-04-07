# Instalace a konfigurace Jetty a mysql pro AV ČR verze CENTOS 8
#### Instalace minimal 4G RAM  4 CPU
Zákaz selinuxu v souboru `/etc/sysconfig/selinux`

Instalace potřebných součástí
```
yum install openldap-clients
yum install mariadb mariadb-server
yum -y install wget
yum install unzip
yum install firewall-config
yum install xauth
yum install net-tools
yum install java-11-openjdk java-11-openjdk-headless
yum update
yum install stunnel
```
Do `.bash_profile` přidáme
```
export JAVA_HOME=/usr/lib/jvm/jre
```

## Instalace Jetty 
Založíme uživatele idp.
```
adduser idp                                                                                                         
```
Dále se jetty instaluje následně:

```
wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.24.v20191120/jetty-distribution-9.4.24.v20191120.tar.gz
mkdir /opt
cd /opt/
tar -xzf ~/jetty-distribution-9.4.27.v20200227.tar.gz
chown -R idp:idp jetty-distribution-9.4.27.v20200227
ln -snf /opt/jetty-distribution-9.4.27.v20200227/bin/jetty.sh /etc/init.d/jetty
echo "JETTY_HOME=/opt/jetty-distribution-9.4.27.v20200227" > /etc/default/jetty
echo "JETTY_BASE=/opt/jetty" >> /etc/default/jetty
echo "JETTY_USER=idp" >> /etc/default/jetty
echo "JETTY_RUN=/opt/run" >> /etc/default/jetty
mkdir /opt/run
chown idp /opt/run
mkdir /opt/jetty
chown idp /opt/jetty
chkconfig --add jetty
cd /opt/jetty
su idp
java -jar /opt/jetty-distribution-9.4.27.v20200227/start.jar    --create-startd --add-to-start=annotations,deploy,ext,http,http2,https,jsp,jstl,plus,requestlog,resources,rewrite,server,servlets,ssl,logging-jetty,console-capture
exit
cd ~
```
Vytvoření keystore
```
openssl rsa -in gedeon-all-idp-key.pem  -out idp.samson.serverkey1.pem -des
cat 1554145495 chain_TERENA_SSL_CA_3.pem  >> jetty-cert.txt
openssl pkcs12 -export -inkey idp.samson.serverkey1.pem -in jetty-cert.txt -out jetty-cert.pkcs12
/usr/lib/java  -importkeystore -srckeystore jetty-cert.pkcs12 -srcstoretype PKCS12 -destkeystore keystore
```
Dále hesla:
```
java -cp /opt/jetty-distribution-9.4.27.v20200227/lib/jetty-util-9.4.27.v20200227.jar      org.eclipse.jetty.util.security.Password nejaketajneheslo
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
cp ../jetty-distribution-9.4.27.v20200227/etc/jetty-ssl-context.xml etc/
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
Do adresáře `/opt/jetty/webapps` nakopírujeme adresář `root` ze starého serveru. Jsou tam loga a další spubory vhodné pro běh IdP.

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

# Instalace stunnel
Protože jako obvykle je oser s připojením k Oracle LDAPu je třeba Stunel. A navíc Oracle LDAP neumí TLS takže to nechodí s stunnelem z distribuce.
Takže nakopírujeme z centos 6 (ještě že ho máme) 
```
cp libnsl.so.1 /lib64/
cp libssl.so.10  /usr/lib64/
cp libcrypto.so.10 /usr/lib64/libcrypto.so.10
cp libwrap.so.0 /lib64/libwrap.so.0
cp /usr/bin/stunnel /usr/bin/stunnel.bak
cp stunnel /usr/bin/stunnel
```
Do `/etc/stunnel/stunnel.conf` dáme obsah:
```
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1
sslVersion = SSLv3
client=yes
[ldap]
accept  = 127.0.0.1:50000
connect = oid1.eis.cas.cz:3132
```
a
```
systemctl enable stunnel
systemctl start stunnel
```
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
