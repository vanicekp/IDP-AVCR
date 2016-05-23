# Instalace a konfigurace Shibboleth a Jetty pro AV ČR
## Oracle JDK - upgrade
Ačkoliv je v linuxových distribucích mnohdy možnost nainstalovat Javu pomocí balíčkovacího systému dané distribuce, např. OpenJDK, silně doporučujeme to, co Shibboleth konzorcium. Použijeme tedy Javu od Oracle. Čas od času se objeví nějaký problém způsobený použitím např. OpenJDK. Budeme-li žádat o podporu, může se stát, že budeme vyzváni, abychom problém reprodukovali s využitím Javy od společnosti Oracle.

Po stažení Oracle JDK umístíme archiv do adresáře `/opt/src` a pomocí následujících příkazů JDK nainstalujeme:
```
http://www.oracle.com/technetwork/java/javase/downloads/index.html
```
#### Rozbalení a instalace Oracle JDK
```
cd /opt
tar -xzvf src/jdk-8u91-linux-x64.tar.gz 
alternatives --install /usr/bin/java java /opt/jdk1.8.0_91/bin/java 120                                                                                                                                                            
alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_91/bin/javac 120                                                                                                                                                         
sed -i.bak 's%export JAVA_HOME=/opt/jdk1.8.0_77%export JAVA_HOME=/opt/jdk1.8.0_91%' ~/.bashrc                                                                                                                                      
source ~/.bashrc                                                                                                                                                                                                                   
```
Odstranění starých java instalaci
```
alternatives --remove java /opt/jdk1.8.0_77/bin/java
alternatives --remove javac /opt/jdk1.8.0_77/bin/javac
alternatives --remove java /usr/lib/jvm/jre-1.7.0-openjdk.x86_64/bin/java
alternatives --remove java /usr/lib/jvm/jre-1.6.0-openjdk.x86_64/bin/java
alternatives --remove java /usr/lib/jvm/jre-1.5.0-gcj/bin/java
```

O korektním nastavení právě nainstalovaného Oracle JDK se můžeme přesvědčit následovně.

#### Příkaz pro zobrazení aktuálně využívané verze Javy
```
alternatives --display java
```

Můžeme se také přesvědčit zavoláním příkazu java.

#### Příkaz pro zobrazení aktuálně využívané verze Javy
```java -version```

#### Výstup příkazu pro zobrazení aktuálně využívané verze Javy
```
java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)
```
Zda máme správně nastavenou proměnnou prostřední $JAVA_HOME zjistíme, pokud znovu načteme konfigurační soubor interpretu BASH a vyžádáme si hodnotu proměnné $JAVA_HOME.

#### Příkaz pro zobrazení hodnoty proměnné $JAVA_HOME
```
source ~/.bashrc && echo $JAVA_HOME
```
#### Výstup příkazu pro zobrazení hodnoty proměnné $JAVA_HOME
```
/opt/jdk1.8.0_91
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
### Odstranění staré Javy (8u77)
```
rm -rf /opt/jdk1.8.0_77/
```

## Jetty - upgrade
Založíme uživatele idp.
```
adduser idp                                                                                                         
```

Instalace Jetty je velice jednoduchá, stačí stáhnout zdrojové kódy Jetty do adresáře `/opt/src` a spustit několik následujících příkazů:

```
http://download.eclipse.org/jetty/9.3.9.v20160517/dist/
```
#### příkazy zadané do terminálu:
```
cd /opt
tar -xzf src/jetty-distribution-9.3.9.v20160517.tar.gz
chown -R idp:idp jetty-distribution-9.3.9.v20160517/                                                    
ln -snf /opt/jetty-distribution-9.3.9.v20160517/bin/jetty.sh /etc/init.d/jetty                          
echo "JETTY_HOME=/opt/jetty-distribution-9.3.9.v20160517" > /etc/default/jetty                           
echo "JETTY_BASE=/opt/jetty" >> /etc/default/jetty                                                      
echo "JETTY_USER=idp" >> /etc/default/jetty                                                                             
```

Pokud dojde k potřebě zvýšit velikost paměti pro běh Jetty, což se nedá při větším počtu virtuálů odhadnout, udělá se to
v `/etc/default/jetty`.
```
JAVA_OPTIONS="-Xmx8192m -Djava.awt.headless=true"
```

Nyní je potřeba Jetty ještě správně nakonfigurovat. Základní konfigurace probíhá spuštěním Jetty s definováním modulů, které budou pro provoz Shibbolethu potřeba:

#### příkazy zadané do terminálu:
```
mv /opt/jetty /opt/jetty-9.3.2.v20150730                                                                
mdir /opt/jetty                                                                                     
chown -R idp:idp /opt/jetty     
su idp
cd /opt/jetty
java -jar /opt/jetty-distribution-9.3.8.v20160314/start.jar \
    --add-to-startd=http,https,logging,deploy,jsp,jstl,plus,servlets,annotations,ext,resources,logging,requestlog
```
 Odkomentujeme proměnnou jetty.http.host a nastavíme ji na hodnotu localhost (výchozí 0.0.0.0).
```
// Povolení HTTP pouze pro localhost
jetty.http.host=localhost
```

 adresáři `/opt/jetty/webapps/ zkopirujeme obsah puvodniho /opt/jetty

#### příkazy zadané do terminálu:
``` 
cp -rv /opt/jetty-9.3.2.v20150730/webapps/* jetty/webapps/                                                                                                                                                                     ```
Připravíme si konfigurační soubor idp.xml, pomocí něhož definujeme, který WAR (Web application ARchive) bude obsahovat webovou aplikaci našeho IdP a na jaké adrese (v tomto případě `https://HOSTNAME_SERVERU/idp`) bude přes web IdP naslouchat.

#### příkaz zadaný do terminálu:
``` 
vi /opt/jetty/webapps/idp.foo.cas.cz.xml
```
Obsah konfiguračního souboru /opt/jetty/webapps/idp.foo.cas.cz.xml je následující.
```
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <Set name="war">/opt/idp/idp.foo.cas.cz/war/idp.war</Set>
    <Set name="contextPath">/idp</Set>
    <Set name="virtualHosts">
     <Array type="java.lang.String">
      <Item>idp.foo.cas.cz</Item>
     </Array>
    </Set>
    <Set name="extractWAR">false</Set>
    <Set name="copyWebDir">false</Set>
    <Set name="copyWebInf">true</Set>
    <Set name="tempDirectory">/opt/jetty/tmp/idp.foo.cas.cz</Set>
</Configure>
```
#### Musíme založit adresář `/opt/jetty/tmp`
```
mkdir /opt/jetty/tmp
```

Tímto máme Jetty téměř připraveno. Zatím jej však nebudeme pouštět, jelikož stejně nemáme nainstalovaný Shibboleth IdP a tedy idp.war zatím neexistuje. Navíc jsme ještě nenakonfigurovali SSL certifikát, aby bylo možné k Shibbolethu přistupovat přes HTTPS.

### SSL certifikát

# Zkopírování současného 'keystore'
```
opt/jetty-9.3.2.v20150730/etc/keystore /opt/jetty/etc/
```

Ze stávající konfigurace použijeme samozřejmě i obfuskovaná hesla.

``
# Získání stávajících hesel pro SSL konfiguraci
grep ^jetty.sslContext. /opt/jetty-9.3.2.v20150730/start.d/ssl.ini
```

Vaše hesla budou samozřejmě jiná. Ta níže uvedená jsou zde jen pro ilustraci.

```
jetty.sslContext.keyStorePassword=OBF:1pifri71kjq1fm1ugw11b1kk01uf1w8f1u61jr1w9b1a31kmm11b1uu1lbw1mw1ri71pjr
jetty.sslContext.keyManagerPassword=OBF:1yy1jn919k1rt1j01j61v2htvl1j41saj1sarjrm1th1v1xj8x1jrq1oph1v921jk91owk
jetty.sslContext.trustStorePassword=OBF:1pifri71kjq1fm1ugw11b1kk01uf1w8f1u61jr1w9b1a31kmm11b1uu1lbw1mw1ri71pjr
````

Výše uvedená hesla zadáme do nového konfiguračního souboru start.d/ssl.ini.
```
# Zadáme výše uvedená hesla do start.d/ssl.ini
vi start.d/ssl.ini
```

#### SSL konfigurace

Výchozí konfigurace Jetty umožňuje i použití dnes již nepříliš důvěryhodných šifer. Proto jejich použití v konfiguraci Jetty zakážeme. K tomu si vytvoříme soubor tweak-ssl.xml v adresáři /opt/jetty/etc.

```
# Úprava SSL konfigurace pro Jetty (jako uživatel idp)
#su idp
vi /opt/jetty/etc/tweak-ssl.xml
```

Obsah souboru tweak-ssl.xml je následující:
```
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">
 
<Configure id="sslContextFactory" class="org.eclipse.jetty.util.ssl.SslContextFactory">
 
  <!-- Zakázání starých a nedůvěryhodných šifer -->
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
      </Array>
    </Arg>
  </Call>
 
  <!-- Zakázání nedůvěryhodných protokolů -->
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
 
  <!-- Povolení Forward Secrecy -->
  <Set name="IncludeCipherSuites">
    <Array type="String">
      <Item>TLS_DHE_RSA.*</Item>
      <Item>TLS_ECDHE.*</Item>
    </Array>
  </Set>
 
</Configure>
```

Nyní už zbývá jen Jetty instruovat, aby tento konfigurační soubor načetlo, to se provede jeho zápisem do start.d/https.ini:

```
# Přidání tweak-ssl.xml do konfigurace Jetty
echo etc/tweak-ssl.xml >> start.d/https.ini
```

S tímto nastavením je zabezpečení komunikace na výrazně vyšší úrovni.
Spuštění Jetty

Nyní zbývá nastavit vlastnická práva na Jetty uživateli idp ze skupiny idp (pro případ, že jsme v předchozích krocích někde zapomněli provést určitou úpravu jako uživatel idp a provedli jsme ji jako root) a spustit Jetty.

```
# Nastavení vlastnických práv pro Jetty a spuštění
chown -R idp:idp /opt/jetty/ /opt/jetty-distribution*/
/etc/init.d/jetty start
```
#### MYSQL
Pro Provoz mysql je třeba ještě nakopírovat knihovny:

```
cp mysql-connector-java-5.1.38-bin.jar /opt/jetty/lib/ext/
cp commons-dbcp-1.4.jar commons-pool-1.6.jar /opt/jetty/lib/ext/
```
#### Firewall
Protože jetty teď běží na neprivilegovanem porty jetřeba ve firewalllu vytvořit pravidlo, které přesměruje příchozí porvoz a povolit přístup na port 8443.
```
Original src Any, Original Dst gedeon, Original Srv https, Translatet Src Original,
Translatet Dst Original, Translatet Stv https 8443.
```

## Hack pro Oracle LDAP
Protože nějak blbne LDAP od Oracle a Java 8 si s ním odmítá povídat, bylo nutné to vyřešit hackem přes stunnel.
Konfigurační soubor /etc/stunnel/stunnel.conf.
```
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1
sslVersion = SSLv3
client=yes
[ldap]
accept  = 127.0.0.1:50000
connect = oid1.eis.cas.cz:3132
```
Je nutno  použít verzi stunnelu minimálně 5. Ta z distribuce centos 6 nefunguje.
Startování stunnelu je v `/etc/rc.local`.

## JAAS
Pro autentifikaci je vzhledem ke komplikovnému schématu nutno použít JAAS. Zdá se, že JETTY má pro každou virtuální instanci zvláštní instanci JAAS, takže není třeba harakiri se změnou názvu přihlašovací procedury. Konfigurace se provede v `conf/authn/password-authn-config.xml`, kde zakomentujeme ladap autentifikaci a povolíme JAAS.
```
    <import resource="jaas-authn-config.xml" />
    <!-- <import resource="krb5-authn-config.xml" /> -->
    <!-- <import resource="ldap-authn-config.xml" /> -->
```
Dále provedeme konfiguraci JAAS v souboru `conf/authn/jaas.config`. `{ID-foo-number}` je číslo ústavu.
```
ShibUserPassAuth {
   org.ldaptive.jaas.LdapLoginModule required
      ldapUrl="ldap://localhost:50000"
      baseDn="cn=Users,dc=eis,dc=cas,dc=cz"
      userFilter="(&(cn={user})(employeenumber={ID-foo-number}*)(orclisenabled=ENABLED))";
};
```

## LDAP connector
Konfigurace LDAP connectoru v conf/ldap.properties opět nefunguje, nejspíše "protože Oracle LDAP", vyřešeno použítím výše zmíněného stunnelu. V `conf/attribute-resolver.xml` nadefinujeme novy DataConnector pro LDAP v jednoduché konfiguraci, ten původní používající `ldap.properties` zakomentujeme.

```
    <!--
        LDAP Connector
        https://wiki.shibboleth.net/confluence/display/SHIB2/ResolverLDAPDataConnector
    -->
    <resolver:DataConnector id="myLDAP" xsi:type="dc:LDAPDirectory"
        ldapURL="ldap://localhost:50000"
        baseDN="cn=Users,dc=eis,dc=cas,dc=cz"
        >
        <dc:FilterTemplate>
            <![CDATA[
                (cn=$requestContext.principalName)
            ]]>
        </dc:FilterTemplate>
    </resolver:DataConnector>
```
## metadata-providers

V konfiguračním souboru `conf/metadata-providers.xml` se nastavuje zdroj metadat. 

#### příkaz zadaný do terminálu:
``` 
vi /opt/shibboleth-idp/conf/metadata-providers.xml
```
```
<!-- czTestFed -->
<MetadataProvider
    id="cztestfed-metadata"
    xsi:type="FileBackedHTTPMetadataProvider"
    backingFile="%{idp.home}/metadata/cztestfed.xml"
    metadataURL="https://metadata.eduid.cz/entities/cztestfed"
    maxRefreshDelay="PT15M">
    <MetadataFilter
        xsi:type="SignatureValidation"
        requireSignedRoot="true"
        certificateFile="%{idp.home}/credentials/metadata.eduid.cz.crt.pem" />
</MetadataProvider>
```
```
<!-- eduID.cz -->
<MetadataProvider
    id="eduid-metadata"
    xsi:type="FileBackedHTTPMetadataProvider"
    backingFile="%{idp.home}/metadata/eduid.xml"
    metadataURL="https://metadata.eduid.cz/entities/eduid+sp"
    maxRefreshDelay="PT15M">
    <MetadataFilter
        xsi:type="SignatureValidation"
        requireSignedRoot="true"
        certificateFile="%{idp.home}/credentials/metadata.eduid.cz.crt.pem" />
</MetadataProvider>
```
```
<!-- eduGAIN -->
<MetadataProvider
    id="edugain-metadata"
    xsi:type="FileBackedHTTPMetadataProvider"
    backingFile="%{idp.home}/metadata/edugain.xml"
    metadataURL="https://metadata.eduid.cz/entities/edugain+sp"
    maxRefreshDelay="PT1H">
    <MetadataFilter
        xsi:type="SignatureValidation"
        requireSignedRoot="true"
        certificateFile="%{idp.home}/credentials/metadata.eduid.cz.crt.pem" />
</MetadataProvider>
```
Veřejný klíč je k dispozici na adrese `https://www.eduid.cz/docs/eduid/metadata/metadata.eduid.cz.crt.pem`. Stáhněte ho a uložte do adresáře `credentials`.

## attribute-resolver
V souboru `attribute-resolver.xml` definujeme atributy.

je nutno důsledně definovat `sourceAttributeID=`. Jinak to huhlá do logu.


## attribute-filter

Oproti předchozím verzím IDP je jednodužší syntaxe souboru, kromě kratší hlavičky :
```
<AttributeFilterPolicyGroup id="ShibbolethFilterPolicy"
        xmlns="urn:mace:shibboleth:2.0:afp"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="urn:mace:shibboleth:2.0:afp http://shibboleth.net/schema/idp/shibboleth-afp.xsd">
```
a zjednodušené hlavičky grup:
```
<AttributeFilterPolicy id="eduidcz">

    <PolicyRequirementRule xsi:type="InEntityGroup"
        groupID="https://eduid.cz/metadata" />
```
je nutno soubor zbavit balastu například:
```
cat /opt/templates/shibboleth/attribute-filter.xml | sed "s/basic://g" | sed "s/afp://g" 
```

### Skripty pro shibboleth 3
Protože Java 8 změnila engine pro vnořené javascripty a zároveň shibbolet 3 změnil částčně API pro psaní javascriptů, bylo nutné upravit scripty pro odháčkování a `eduPersonEntitlement`.

Script `/opt/idp/common/script/commonNameASCII.js`
```
load("nashorn:mozilla_compat.js")
logger = Java.type("org.slf4j.LoggerFactory").getLogger("net.shibboleth.idp.attribute.resolver.eppnbuilder");
scopedValueType =  Java.type("net.shibboleth.idp.attribute.ScopedStringAttributeValue");
var localpart = "";

importPackage(Packages.java.lang);
importPackage(Packages.java.text);


if (!commonName.getValues().isEmpty()) {
    originalValue = commonName.getValues().get(0);
    asciiValue = Normalizer.normalize(originalValue, Normalizer.Form.NFD).replaceAll("\\p{InCombiningDiacriticalMarks}+", "");

    commonNameASCII.getValues().add(asciiValue);
}
```

Script  `/opt/idp/common/script/eduPersonEntitlementFoo.js`
```
load("nashorn:mozilla_compat.js")
logger = Java.type("org.slf4j.LoggerFactory").getLogger("net.shibboleth.idp.attribute.resolver.eppnbuilder");
scopedValueType =  Java.type("net.shibboleth.idp.attribute.ScopedStringAttributeValue");
var localpart = "";

importPackage(Packages.org.slf4j);

if (typeof uniqueIdentifier != "undefined" && uniqueIdentifier != null && uniqueIdentifier.getValues().contains("EduID")) {
                originalValue = unstructuredName.getValues().get(0);
                        if (originalValue == "{OID1}") {
                                eduPersonEntitlement.getValues().add("urn:mace:terena.org:tcs:escience-admin");
                                eduPersonEntitlement.getValues().add("urn:mace:terena.org:tcs:personal-admin");
                        }
          eduPersonEntitlement.getValues().add("urn:mace:terena.org:tcs:personal-user");
          eduPersonEntitlement.getValues().add("urn:mace:terena.org:tcs:escience-user");
}
eduPersonEntitlement.getValues().add("urn:mace:dir:entitlement:common-lib-terms");
```
## Persistentní identifikátor / eduPersonTargetedID
### MySQL
Nainstalujeme mysql-server a uděláme základní konfiguraci
```
 yum install mysql-server
 /sbin/chkconfig --levels 235 mysqld on 
 service mysqld start
```
Nastavíme hesla k mysql aby to trochu chodilo, odpovědi na otázky scriptu dejte podle nejlepšího vědomí a svědomí.
```
mysql_secure_installation
```
#### Migrace stávajících identifikátorů
na původním IDP stáhneme databázi identifikátorů
```
mysqldump shibboleth > ~/persistentID.sql
```
Hesla použijeme z existujícího souboru `attribute-resolver.xml`

Na novém IDP založíme databázi a importujeme data.
```
mysql -u root -p
```
Zadáme příkazy:
```
SET NAMES 'utf8';
SET CHARACTER SET utf8;
CHARSET utf8;
CREATE DATABASE IF NOT EXISTS shibboleth CHARACTER SET=utf8;
GRANT ALL PRIVILEGES ON shibboleth.* TO 'shibboleth'@'localhost' IDENTIFIED BY 'silne_heslo';
FLUSH PRIVILEGES;
```
Importujeme data
```
mysql -u root -p shibboleth < ~/persistentID.sql
```

#### Migrace z IDP 3.1.x
```
mysqldump shibboleth > shibpid-3.1.2.sql
```
změníme strukturu tabulky z:
```
CREATE TABLE `shibpid` (
  `localEntity` text NOT NULL,
  `peerEntity` text NOT NULL,
  `principalName` varchar(255) NOT NULL DEFAULT '',
  `localId` varchar(255) NOT NULL,
  `persistentId` varchar(36) NOT NULL,
  `peerProvidedId` varchar(255) DEFAULT NULL,
  `creationDate` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deactivationDate` timestamp NULL DEFAULT NULL,
  KEY `persistentId` (`persistentId`),
  KEY `persistentId_2` (`persistentId`,`deactivationDate`),
  KEY `localEntity` (`localEntity`(16),`peerEntity`(16),`localId`),
  KEY `localEntity_2` (`localEntity`(16),`peerEntity`(16),`localId`,`deactivationDate`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```
na:
```
CREATE TABLE `shibpid` (
  `localEntity` varchar(255) NOT NULL,
  `peerEntity` varchar(255) NOT NULL,
  `principalName` varchar(255) NOT NULL DEFAULT '',
  `localId` varchar(255) NOT NULL,
  `persistentId` varchar(50) NOT NULL,
  `peerProvidedId` varchar(255) DEFAULT NULL,
  `creationDate` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deactivationDate` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (localEntity, peerEntity, persistentId)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
a importujeme
```
mysql shibboleth < shibpid-3.2.0.sql
```

### /etc/my.cnf
Do konfiguračního souboru přidáme delší timeout.
```
[mysqld]
wait_timeout=31536000
```

### Jetty
Jetty potřebuje pro správnou funkčnost tyto čtyři JAR soubory, které je nutné umístit do složky s externími knihovnami `/opt/jetty/lib/ext`: 

commons-dbcp2-2.1.1.jar `https://search.maven.org/remotecontent?filepath=org/apache/commons/commons-dbcp2/2.1.1/commons-dbcp2-2.1.1.jar`

commons-pool2-2.4.2.jar `https://search.maven.org/remotecontent?filepath=org/apache/commons/commons-pool2/2.4.2/commons-pool2-2.4.2.jar`

commons-logging-api-1.1.jar `https://search.maven.org/remotecontent?filepath=commons-logging/commons-logging-api/1.1/commons-logging-api-1.1.jar`

mysql-connector-java-5.1.36-bin.jar `http://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.36.tar.gz`


#### Apache Commons DBCP & Pool
```
cd /opt/src/
cp commons-dbcp2-2.1.1.jar commons-pool2-2.4.2.jar commons-logging-api-1.1.jar /opt/jetty/lib/ext/
```
#####  V případě upgrade z IDP 3.1.x 
smazat staré jar soubory.
```
cd /opt/jetty/lib/ext/
rm -i commons-dbcp-1.4.jar commons-pool-1.6.jar
```

#### MySQL Connector/J
```
cd /opt/src/
tar -xzf mysql-connector-java-5.1.38.tar.gz
cp mysql-connector-java-5.1.38/mysql-connector-java-5.1.38-bin.jar /opt/jetty/lib/ext/
```
### Shibboleth IdP
Do konfiguračního souboru attribute-resolver.xml musíme doplnit podporu pro persistentní identifikátor.
```
vi conf/attribute-resolver.xml
```
Nejprve je potřeba zadat definici atributu:
```
<!-- eduPersonTargetedID -->
<resolver:AttributeDefinition id="eduPersonTargetedID" xsi:type="ad:SAML2NameID" sourceAttributeID="persistentID" nameIdFormat="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
    <resolver:Dependency ref="myStoredId" />
    <resolver:AttributeEncoder xsi:type="enc:SAML1XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" />
    <resolver:AttributeEncoder xsi:type="enc:SAML2XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" friendlyName="eduPersonTargetedID" />
</resolver:AttributeDefinition>
```
A poté je potřeba nadefinovat konektor. Salt jsem vzal ze staré instalace.
```
<!-- StoredId Data Connector -->
<resolver:DataConnector id="myStoredId"
    xsi:type="dc:StoredId"
    sourceAttributeID="eduPersonPrincipalName"
    generatedAttributeID="persistentID"
    salt="SALT"
    queryTimeout="0">
    <resolver:Dependency ref="eduPersonPrincipalName" />
    <dc:BeanManagedConnection>shibboleth.MySQLDataSource</dc:BeanManagedConnection>
</resolver:DataConnector>
```
Nyní je potřeba v souboru `conf/global.xml` definovat bean-y. 
```
vi conf/global.xml
```
Následující konfigurace zajistí správnou konektivitu na MySQL databázi pro ukládání persistentních identifikátorů.
```
<bean id="shibboleth.MySQLDataSource"
    class="org.apache.commons.dbcp2.BasicDataSource"
    p:driverClassName="com.mysql.jdbc.Driver"
    p:url="jdbc:mysql://localhost:3306/shibboleth?autoReconnect=true&amp;initialTimeout=3&amp;sessionVariables=wait_timeout=31536000"
    p:username="shibboleth"
    p:password="silne_heslo" />
```
 V konfiguračním souboru attribute-filter.xml nastavíme vydávání atributu eduPersonTargetedID. 
``` 
vi /opt/shibboleth-idp/conf/attribute-filter.xml
```
```
<afp:AttributeRule attributeID="eduPersonTargetedID">
  <afp:PermitValueRule xsi:type="basic:ANY" />
</afp:AttributeRule>
```
Konfigurační soubor `saml-nameid.properties` je potřeba také upravit.
```
vi conf/saml-nameid.properties
```
Zde definujeme odkazy na výše definované bean-y, dále atribut, který se bude pro výpočet persistentního identifikátoru používat (uid) a salt použitou pro výpočet (tu jste si již vygenerovali výše).
```
idp.persistentId.generator = shibboleth.StoredPersistentIdGenerator
idp.persistentId.dataSource = shibboleth.MySQLDataSource
idp.persistentId.sourceAttribute = eduPersonPrincipalName
idp.persistentId.salt = SALT
```
Podpora persistentních identifikátorů je ještě potřeba zapnout v konfiguračním souboru saml-nameid.xml.
```
vi conf/saml-nameid.xml
```
Stačí odkomentovat následující řádek, který je ve výchozí konfiguraci po instalaci IdP zakomentovaný.
```
<ref bean="shibboleth.SAML2PersistentGenerator" />
```
Ještě zbývá úprava v souboru `subject-c14n.xml`.
```
vi conf/c14n/subject-c14n.xml
```
Odkomentujte následující řádek:
```
<ref bean="c14n/SAML2Persistent" />
```
V metadatech IdP je potřeba zadat, že IdP podporuje persistentní identifikátor.
```
vi /opt/shibboleth-idp/metadata/idp-metadata.xml
```
Přidejte tedy do elementu `<IDPSSODescriptor>` následující řádek ke zbývajícím dvěma elementům `<NameIDFormat>`.
```
<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>
```
Poslední nutný konfigurační krok spočívá v přidání knihovny JSTL do Shibbolethu. Zkopírujte tedy `jstl-1.2.jar` (`http://search.maven.org/remotecontent?filepath=jstl/jstl/1.2/jstl-1.2.jar`) do adresáře `edit-webapp/WEB-INF/lib`. A následně přegenerujte JAR Shibbolethu a restartujte Jetty:
```
cd /opt/shibboleth-idp
./bin/build.sh
chown -R idp:idp /opt/jetty
chown -R idp:idp /opt/idp/idp*
/etc/init.d/jetty restart
```
