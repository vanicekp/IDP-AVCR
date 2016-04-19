# Oracle JDK

Ačkoliv je v linuxových distribucích mnohdy možnost nainstalovat Javu pomocí b
alíčkovacího systému dané distribuce, např. OpenJDK, silně doporučujeme to, co S
hibboleth konzorcium. Použijeme tedy Javu od Oracle. Čas od času se objeví nějaký pr
oblém způsobený použitím např. OpenJDK. Budeme-li žádat o podporu, může se stát, že b
udeme vyzváni, abychom problém reprodukovali s využitím Javy od společnosti Oracle.

Po stažení Oracle JDK umístíme archiv do adresáře /opt/src a pomocí následujících příkazů J
DK nainstalujeme:
```
http://www.oracle.com/technetwork/java/javase/downloads/index.html
```

### Rozbalení a instalace Oracle JDK
```
cd /opt
tar -xzf src/jdk-8u77-linux-x64.tar.gz
alternatives --install /usr/bin/java java /opt/jdk1.8.0_77/bin/java 120
alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_77/bin/javac 120
echo export JAVA_HOME=/opt/jdk1.8.0_77 >> ~/.bashrc
```
O korektním nastavení právě nainstalovaného Oracle JDK se můžeme přesvědčit následovně.

### Příkaz pro zobrazení aktuálně využívané verze Javy
```
alternatives --display java
```

### Výstup příkazu pro zobrazení aktuálně využívané verze Javy
```
java - auto mode
  link currently points to /opt/jdk1.8.0_77/bin/java
/opt/jdk1.8.0_77/bin/java - priority 120
Current 'best' version is '/opt/jdk1.8.0_77/bin/java'.
```
Můžeme se také přesvědčit zavoláním příkazu java.

### Příkaz pro zobrazení aktuálně využívané verze Javy
```java -version```

### Výstup příkazu pro zobrazení aktuálně využívané verze Javy
```
java version "1.8.0_77"
Java(TM) SE Runtime Environment (build 1.8.0_77-b03)
Java HotSpot(TM) 64-Bit Server VM (build 25.77-b03, mixed mode)
```

Zda máme správně nastavenou proměnnou prostřední $JAVA_HOME zjistíme, pokud znovu načteme konfigurační soubor interpretu BASH a vyžádáme si hodnotu proměnné $JAVA_HOME.

### Příkaz pro zobrazení hodnoty proměnné $JAVA_HOME
```
source ~/.bashrc && echo $JAVA_HOME
```

### Výstup příkazu pro zobrazení hodnoty proměnné $JAVA_HOME
```
/opt/jdk1.8.0_77
```
## Java Cryptography Extension (Unlimited Strength Jurisdiction Policy Files)

Po nainstalování Oracle JDK je ještě potřeba doinstalovat tzv. JCE US (Java Cryptography Extension Unlimited Strength), které zajistí možnost využít silnější šifrování.

Stáhneme JCE a umístíme jej do adresáře /opt/src.
```
http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html`
```
### Rozbalení archivu s JCE US
```
cd /opt
unzip -x src/jce_policy-8.zip
```

Pro „instalaci“ JCE US stačí zkopírovat dva následující JAR (Java ARchive) soubory do Oracle JDK:

### Instalace JCE US
```
cp UnlimitedJCEPolicyJDK8/US_export_policy.jar jdk1.8.0_77/jre/lib/security/
cp UnlimitedJCEPolicyJDK8/local_policy.jar jdk1.8.0_77/jre/lib/security/
```
Adresář s rozbaleným rozšířením JCE US můžeme nyní smazat.

### Smazání rozbaleného archivu s JCE US
```
rm -rf UnlimitedJCEPolicyJDK8/
```

# Jetty

Instalace Jetty je velice jednoduchá, stačí stáhnout zdrojové kódy Jetty do adřesáře /opt/src a spustit několik následujících příkazů:

```
http://download.eclipse.org/jetty/9.3.8.v20160314dist/
```

### příkazy zadané do terminálu:
```
cd /opt
mkdir -p jetty/tmp
tar -xzvf src/jetty-distribution-9.3.8.v20160314.tar.gz
ln -snf /opt/jetty-distribution-9.3.8.v20160314/bin/jetty.sh /etc/init.d/jetty
echo "JETTY_HOME=/opt/jetty-distribution-99.3.8.v20160314 >> /etc/default/jetty
echo "JETTY_BASE=/opt/jetty" >> /etc/default/jetty
```

Nyní je potřeba Jetty ještě správně nakonfigurovat. Základní konfigurace probíhá spuštěním Jetty s definováním modulů, které budou pro provoz Shibbolethu potřeba:

### příkazy zadané do terminálu:
``` 
cd /opt/jetty
java -jar /opt/jetty-distribution-9.3.2.v20150730/start.jar \
    --add-to-startd=https,logging,deploy,jsp,jstl,plus,servlets,annotations,ext,resources,logging,requestlog
```
V souboru start.d/ssl.ini je nutné změnit port, na kterém poběží HTTPS:

### příkaz zadaný do terminálu:
``` 
vi start.d/ssl.ini
```
Výchozí nastavení jetty.ssl.port=8443 změníme následovně:
```
jetty.ssl.port=443
```
V adresáři /opt/jetty/webapps/root vytvoříme jednoduchou stránku, která se zobrazí při zadání URL adresy naší instalace Jetty. Toto je sice nepoviné, ale pokud se někdo dostane na stránku samotného IdP, je zajisté dobré, aby stránka nevypadala matoucím dojmem. Obsah souboru index.html si upravte dle svého vlastního uvážení – můžete např. nastavit přesměrování na domovskou stránku své organizace.

### příkazy zadané do terminálu:
``` 
mkdir -p /opt/jetty/webapps/root
vi /opt/jetty/webapps/root/index.html
```
Připravíme si konfigurační soubor idp.xml, pomocí něhož definujeme, který WAR (Web application ARchive) bude obsahovat webovou aplikaci našeho IdP a na jaké adrese (v tomto případě https://HOSTNAME_SERVERU/idp) bude přes web IdP naslouchat.

# příkaz zadaný do terminálu:
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
      <Item>idp.test.cas.cz</Item>
     </Array>
    </Set>
    <Set name="extractWAR">false</Set>
    <Set name="copyWebDir">false</Set>
    <Set name="copyWebInf">true</Set>
    <Set name="tempDirectory">/opt/jetty/tmp</Set>
</Configure>
```

Tímto máme Jetty téměř připraveno. Zatím jej však nebudeme pouštět, jelikož stejně nemáme nainstalovaný Shibboleth IdP a tedy idp.war zatím neexistuje. Navíc jsme ještě nenakonfigurovali SSL certifikát, aby bylo možné k Shibbolethu přistupovat přes HTTPS.

## SSL certifikát

Nyní je ještě potřeba nakonfigurovat použití SSL certifikátu v Jetty, aby bylo možné provozovat Shibboleth IdP přes HTTPS. K tomuto účelu slouží u Javy tzv. „keystore“. Pro korektní zprovoznění HTTPS je potřeba, aby se do klíčenky („keystore“) uložil SSL certifikát včetně kompletního řetězce až ke kořenovému certifikátu certifikační autority (CA).

Následující návod je pro SSL certifikát získaný pomocí služby TCS CESNET. Soubor cert.pem obsahuje cílový certifikát pro server whoami-dev.cesnet.cz a chain_TERENA_SSL_CA_2.pem obsahuje řetězec certifikátů až k samotnému kořenovému CA. Certifikát pro server nejprve sloučíme s kompletním řetězcem:

# příkaz zadaný do terminálu:
 
cat cert.pem chain_TERENA_SSL_CA_2.pem >> jetty-cert.txt

Nyní převedeme certifikát s kompletním řetězcem až k CA do formátu PKCS #12. Budeme požádáni o heslo ke klíči (soubor key.pem) a následně budeme požádáni o nové heslo k souboru jetty-cert.pkcs12.

# příkaz zadaný do terminálu:
 
openssl pkcs12 -export -inkey key.pem -in jetty-cert.txt -out jetty-cert.pkcs12

# výstup příkazu:
 
Enter pass phrase for serverkey.pem:
Enter Export Password:
Verifying - Enter Export Password:

Nyní certifikát včetně kořene ve formátu PKCS #12 (soubor jetty-cert.pkcs12) importujeme do „klíčenky“ Java keystore:

# příkaz zadaný do terminálu:
 
/opt/jdk1.8.0_60/bin/keytool -importkeystore -srckeystore jetty-cert.pkcs12 -srcstoretype PKCS12 -destkeystore keystore

Nejprve budeme požádáni o heslo k nově vytvářenému „keystore“ (Enter destination keystore password). Pak budeme požádáni o zopakování tohoto hesla (Re-enter new password). Následně budeme požádáni o heslo k certifikátu (soubor jetty-cert.pkcs12), který importujeme do „keystore“ (Enter source keystore password).

# výstup příkazu:
 
Enter destination keystore password:  
Re-enter new password: 
Enter source keystore password:  
Entry for alias 1 successfully imported.
Import command completed:  1 entries successfully imported, 0 entries failed or cancelled

Následně je už jen potřeba keystore uložený v souboru keystore přesunout do Jetty:

# příkaz zadaný do terminálu:
 
mv keystore /opt/jetty/etc

Předposledním krokem je vygenerovat si pomocí jetty-util, jež je součástí instalace Jetty, obfuskovanou podobu hesla pro přístup ke „keystore“ a pro přístup k certifikátu.

Heslo ke keystore:

# příkaz zadaný do terminálu:
 
java -cp /opt/jetty-distribution-9.3.2.v20150730/lib/jetty-util-9.3.2.v20150730.jar \
    org.eclipse.jetty.util.security.Password <heslo_ke_keystore>

# výstup příkazu:
 
2015-06-16 15:56:58.986:INFO::main: Logging initialized @322ms
keystore
OBF:1u9x1vn61z0p1yta1ytc1z051vnw1u9l
MD5:5fba3d2b004d68d3c5ca4e174024fc81

Heslo k certifikátu (heslo, které jste použili při generování klíče k certifikátu):

# příkaz zadaný do terminálu:
 
java -cp /opt/jetty-distribution-9.3.2.v20150730/lib/jetty-util-9.3.2.v20150730.jar \
    org.eclipse.jetty.util.security.Password <heslo_k_certifikátu>

# výstup příkazu:
 
2015-06-16 15:57:02.322:INFO::main: Logging initialized @308ms
certificate
OBF:1sot1w1c1uvk1vo01unz1thb1unz1vn21uum1w261sox
MD5:e0d30cef5c6139275b58b525001b413c

Heslo (případně hesla) je potřeba zadat do souboru start.d/ssl.ini (jetty.keystore.password bude stejné jako jetty.truststore.password):

# příkaz zadaný do terminálu:
 
vi /opt/jetty/start.d/ssl.ini

# konfigurační změny v souboru 'ssl.ini'
 
jetty.sslContext.keyStorePassword=OBF:1u9x1vn61z0p1yta1ytc1z051vnw1u9l
jetty.sslContext.keyManagerPassword=OBF:1sot1w1c1uvk1vo01unz1thb1unz1vn21uum1w261sox
jetty.sslContext.trustStorePassword=OBF:1u9x1vn61z0p1yta1ytc1z051vnw1u9l

Proměnné jetty.sslContext.keyStorePassword a jetty.sslContext.trustStorePassword nastavte na obfuskované heslo ke „keystore“. Proměnnou jetty.sslContext.keyManagerPassword nastavte na obfuskované heslo ke klíči certifikátu (soubor jetty-cert.pkcs12!). Pokud to popletete, Jetty odmítne nastartovat, jelikož nepřečte keystore a klíč.
SSL konfigurace

Výchozí konfigurace Jetty umožňuje použití i dnes již nepříliš důvěryhodných šifer. Proto jejich použití v konfiguraci zakážeme.

# příkazy zadané do terminálu:
 
cd /opt/jetty
cp ../jetty-distribution-9.3.2.v20150730/etc/jetty-ssl-context.xml etc/
vi etc/jetty-ssl-context.xml

Konfigurační soubor jetty-ssl-context.xml, který jsme zkopírovali z distribučního adresáře Jetty, obsahuje seznam některých zakázaných šifer:

<Set name="ExcludeCipherSuites">
  <Array type="String">
    <Item>SSL_RSA_WITH_DES_CBC_SHA</Item>
    <Item>SSL_DHE_RSA_WITH_DES_CBC_SHA</Item>
    <Item>SSL_DHE_DSS_WITH_DES_CBC_SHA</Item>
    <Item>SSL_RSA_EXPORT_WITH_RC4_40_MD5</Item>
    <Item>SSL_RSA_EXPORT_WITH_DES40_CBC_SHA</Item>
    <Item>SSL_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA</Item>
    <Item>SSL_DHE_DSS_EXPORT_WITH_DES40_CBC_SHA</Item>
  </Array>
</Set>

My však seznam zakázaných šifer upravíme následujícím způsobem:

<Set name="ExcludeCipherSuites">
  <Array type="String">
    <Item>.*NULL.*</Item>
    <Item>.*RC4.*</Item>
    <Item>.*MD5.*</Item>
    <Item>.*DES.*</Item>
    <Item>.*DSS.*</Item>
    <Item>TLS_DHE_RSA_WITH_AES_128.*</Item>
    <Item>TLS_DHE_RSA_WITH_AES_256.*</Item>
  </Array>
</Set>

A nyní ještě zahrneme do konfigurace šifry pro podporu „Forward Secrecy“. Do konfiguračního souboru jetty-ssl-context.xml tedy těsně za předchozí nastavení vložíme tyto řádky:

<Set name="IncludeCipherSuites">
  <Array type="String">
    <Item>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA</Item>
    <Item>TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA</Item>
    <Item>TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA</Item>
  </Array>
</Set>

Konfigurační soubor jetty-ssl-context.xml s těmito úpravami lze stáhnout.

S tímto nastavením by mělo být zabezpečení komunikace na výrazně vyšší úrovni.
Shibboleth CLI skripty & HTTP

Chcete-li využívat Shibboleth CLI skripty (není potřeba, avšak vřele doporučuji, jelikož je možné požádat o „reload“ konfigurace z shellu serveru bez nutnosti restartovat celý javovský Jetty kontejner), je vhodné, aby byl Shibboleth dostupný i přes HTTP na localhosti (https://localhost/idp). Bezpečného řešení lze docílit spuštěním Jetty přes HTTP pouze na adrese lokálního rozhraní 127.0.0.1.

Druhou možností je spouštět skript reload-service.sh s parametrem -u, pomocí něhož definujeme alternativní adresu (např. https://whoami-dev.cesnet.cz) a tím pádem tedy Jetty na HTTP portu poslouchat nemusí. Pro zjednodušení je možné udělat tzv. alias v interpretu příkazů, abyste se vyhnuli zadávání -u <URL>.

Zprovoznění Jetty na HTTP je však triviální záležitostí pomocí několika následujících příkazů a konfiguračních úprav.

# příkazy zadané do terminálu:
 
cd /opt/jetty
java -jar /opt/jetty-distribution-9.3.2.v20150730/start.jar --add-to-startd=http

V souboru start.d/http.ini provedeme dvě konfigurační změny.

# příkaz zadaný do terminálu:
 
vi start.d/http.ini

Odkomentujeme proměnné jetty.http.host a jetty.http.port a nastavíme je na hodnoty 127.0.0.1 (výchozí 0.0.0.0), resp. 80 (výchozí 8080).

jetty.http.host=127.0.0.1
jetty.http.port=80

Toto je velice důležité. Prosím, zkontrolujte, zda vám po restartu běží Jetty nešifrovaně (port 80) pouze na „localhostu“ (IP adresa 127.0.0.1) např. pomocí utility nestat:

# příkazy zadané do terminálu:
 
/etc/init.d/jetty start
netstat -an | grep ":80"

Měl by se Vám zobrazit následující výstup.

# výstup příkazu:
 
tcp6       0      0 127.0.0.1:80            :::*                    LISTEN     

Budete-li se dívat do logů Jetty, nelekejte se následující chyby, která se „táhne“ přes mnoho řádků. Je to v pořádku. Shibboleth IdP ještě není nainstalován, soubor idp.war tedy ještě neexistuje:

2015-08-05 09:02:22.871:WARN:oejw.WebInfConfiguration:main: Web application not found /opt/shibboleth-idp/war/idp.war
2015-08-05 09:02:22.872:WARN:oejw.WebAppContext:main: Failed startup of context o.e.j.w.WebAppContext@df27fae{/idp,null,null}{/opt/shibboleth-idp/war/idp.war}
java.io.FileNotFoundException: /opt/shibboleth-idp/war/idp.war
        at org.eclipse.jetty.webapp.WebInfConfiguration.unpack(WebInfConfiguration.java:495)
        at org.eclipse.jetty.webapp.WebInfConfiguration.preConfigure(WebInfConfiguration.java:72)
        at org.eclipse.jetty.webapp.WebAppContext.preConfigure(WebAppContext.java:474)
        at org.eclipse.jetty.webapp.WebAppContext.doStart(WebAppContext.java:510)
...

Jetty je však možné předchozím příkazem nastartovat a ověřit, že v pořádku funguje. Vyzkoušejte přístup přes HTTPS ze svého počítače a případně i přístup přes HTTP z terminálu serveru:

# příkaz zadaný do terminálu:
 
wget -q -O - http://127.0.0.1

Měli byste vidět obsah souboru /opt/jetty/webapps/root/index.html. 
