# Instalace IdP

Stáhnout Shibboleth IdP a umístit do adresáře /opt/src.
```
http://shibboleth.net/downloads/identity-provider/
```
### příkazy zadané do terminálu:
``` 
cd /opt/src
tar -xzf src/shibboleth-identity-provider-3.1.2.tar.gz
```
### Příprava instalace
```
mkdir /opt/dist
```
```
cd /opt/dist
cp -r /opt/src/shibboleth-identity-provider-3.1.2 idp.foo.cas.cz-source
```
### Změna idp.home
V souboru idp.foo.cas.cz-source/webapp/WEB-INF/web.xml musíme doplnit definici proměné idp.home, jinak bude instalace předpokládat umístění v 
defaultním adresáři /opt/shibboleth-idp.
```
vi idp.foo.cas.cz-source/webapp/WEB-INF/web.xml
```
Za parametr <display-name> volžíme 
```
<context-param>
    <param-name>idp.home</param-name>
    <param-value>/opt/idp/idp.foo.cas.cz</param-value>
</context-param>
```

### Instalce
```
cd idp.foo.cas.cz-source/
./bin/install.sh
```

Po spuštění instalačního skriptu:

    potvrdíme zdrojový instalační adresář,
    vyplníme hostname serveru (pokud není předvyplněn správně),
    potvrdíme entityID (případně upravíme na to stávající, které aktuálně používáme ve federaci [prosím, neměňte entityID, zbytečně si tím přiděláte problémy!]),
    zadáme scope organizace,
    vložíme dvě hesla, o která budeme požádáni (jako generátor lze použít unixovou utilitu pwgen).

Zde je vyobrazen průběh instalačního skriptu install.sh.
```
$ ./bin/install.sh 
Source (Distribution) Directory: [/opt/dist/idp.foo.cas.cz-source]
 
Installation Directory: [/opt/shibboleth-idp]
/opt/idp/idp.foo.cas.cz
 
Hostname: [gedeon.cas.cz]
idp.foo.cas.cz

SAML EntityID: [https://idp.foo.cas.cz/idp/shibboleth]
 
Attribute Scope: [cas.cz]
foo.cas.cz
TLS Private Key Password: 
Re-enter password: 
Cookie Encryption Key Password: 
Re-enter password: 
Warning: /opt/idp/idp.foo.cas.cz/bin does not exist.
Warning: /opt/idp/idp.foo.cas.cz/dist does not exist.
Warning: /opt/idp/idp.foo.cas.cz/doc does not exist. 
Warning: /opt/idp/idp.foo.cas.cz/system does not exist.
Warning: /opt/idp/idp.foo.cas.cz/webapp does not exist.
Generating Signing Key, CN = idp.foo.cas.cz URI = https://idp.foo.cas.cz/idp/shibboleth ...
...done                                                                                      
Creating Encryption Key, CN = idp.foo.cas.cz URI = https://idp.foo.cas.cz/idp/shibboleth ...
...done
Creating TLS keystore, CN = idp.foo.cas.cz URI = https://idp.foo.cas.cz/idp/shibboleth ...
...done
Creating cookie encryption key filest   s...
...done
Rebuilding /opt/idp/idp.foo.cas.cz/war/idp.war ...
...done

BUILD SUCCESSFUL
Total time: 1 minute 58 seconds
```
### Povolení Status URL

V souboru /opt/idp/idp.foo.cas.cz/conf/access-control.xml doplníme IP adresy správcovských stanic pro kontrolu 
statusu serveru.
```
cd /opt/idp/idp.test.cas.cz/
vi conf/access-control.xml
```
Editujeme:
```
<entry key="AccessByIPAddress">
        <bean parent="shibboleth.IPRangeAccessControl"
        p:allowedRanges="#{ {'127.0.0.1/32', '::1/128', '147.231.12.0/22'} }" />
</entry>
```
## Konfigurace Jetty virtuálu
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
    <Set name="tempDirectory">/opt/jetty/tmp/idp.foo.cas.cz</Set>
</Configure>
```

Restartujeme Jetty, čímž dosáhneme nahrání servletu s Shibboleth IdP:

### příkaz zadaný do terminálu:
``` 
/etc/init.d/jetty restart
```
Nyní můžeme vyzkoušet, zda-li Shibboleth IdP běží 

V prohlížeči dáme URL
```
https://idp.foo.cas.cz/idp/status
```
Pokud vše proběhlo bez problémů, uvidíte následující výstup:
```
### Operating Environment Information
operating_system: Linux
operating_system_version: 2.6.32-573.22.1.el6.x86_64
operating_system_architecture: amd64
jdk_version: 1.8.0_77
available_cores: 4
used_memory: 510 MB
maximum_memory: 1751 MB

### Identity Provider Information
idp_version: 3.1.2
start_time: 2016-04-19T09:06:51+02:00
current_time: 2016-04-19T09:06:52+02:00
uptime: 803 ms

service: shibboleth.LoggingService
last successful reload attempt: 2016-04-19T07:06:41Z
last reload attempt: 2016-04-19T07:06:41Z

service: shibboleth.ReloadableAccessControlService
last successful reload attempt: 2016-04-19T07:06:43Z
last reload attempt: 2016-04-19T07:06:43Z

service: shibboleth.MetadataResolverService
last successful reload attempt: 2016-04-19T07:06:43Z
last reload attempt: 2016-04-19T07:06:43Z

	metadata source: ShibbolethMetadata

service: shibboleth.RelyingPartyResolverService
last successful reload attempt: 2016-04-19T07:06:43Z
last reload attempt: 2016-04-19T07:06:43Z

service: shibboleth.NameIdentifierGenerationService
last successful reload attempt: 2016-04-19T07:06:43Z
last reload attempt: 2016-04-19T07:06:43Z

service: shibboleth.AttributeResolverService
last successful reload attempt: 2016-04-19T07:06:42Z
last reload attempt: 2016-04-19T07:06:42Z

service: shibboleth.AttributeFilterService
last successful reload attempt: 2016-04-19T07:06:42Z
last reload attempt: 2016-04-19T07:06:42Z
```

Můžete také zkusit ze svého počítače přístoupit na URL adresu s IdP: https://idp.foo.cas.cz/idp. Nicméně neuvidíte nic zajímavého. 
