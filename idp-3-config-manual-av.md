# Instalace IdP

Stáhnout Shibboleth IdP a umístit do adresáře /opt/src.
```
http://shibboleth.net/downloads/identity-provider/
```
### příkazy zadané do terminálu:
``` 
cd /opt
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



cd shibboleth-identity-provider-3.1.2/
./bin/install.sh
```
Po spuštění instalačního skriptu:

    potvrdíme zdrojový instalační adresář,
    vyplníme hostname serveru (pokud není předvyplněn správně),
    potvrdíme entityID (případně upravíme na to stávající, které aktuálně používáme ve federaci [prosím, neměňte entityID, zbytečně si tím přiděláte problémy!]),
    zadáme scope organizace,
    vložíme dvě hesla, o která budeme požádáni (jako generátor lze použít unixovou utilitu pwgen).

Zde je vyobrazen průběh instalačního skriptu install.sh.

$ ./bin/install.sh 
Source (Distribution) Directory: [/opt/shibboleth-identity-provider-3.1.2]
 
Installation Directory: [/opt/shibboleth-idp]
 
Hostname: [localhost.localdomain]
whoami-dev.cesnet.cz
SAML EntityID: [https://whoami-dev.cesnet.cz/idp/shibboleth]
 
Attribute Scope: [localdomain]
cesnet.cz
TLS Private Key Password: 
Re-enter password: 
Cookie Encryption Key Password: 
Re-enter password: 
Warning: /opt/shibboleth-idp/bin does not exist.
Warning: /opt/shibboleth-idp/dist does not exist.
Warning: /opt/shibboleth-idp/doc does not exist.
Warning: /opt/shibboleth-idp/system does not exist.
Warning: /opt/shibboleth-idp/webapp does not exist.
Generating Signing Key, CN = whoami-dev.cesnet.cz URI = https://whoami-dev.cesnet.cz/idp/shibboleth ...
...done
Creating Encryption Key, CN = whoami-dev.cesnet.cz URI = https://whoami-dev.cesnet.cz/idp/shibboleth ...
...done
Creating TLS keystore, CN = whoami-dev.cesnet.cz URI = https://whoami-dev.cesnet.cz/idp/shibboleth ...
...done
Creating cookie encryption key files...
...done
Rebuilding /opt/shibboleth-idp/war/idp.war ...
...done
 
BUILD SUCCESSFUL
Total time: 1 minute 0 seconds

Restartujeme Jetty, čímž dosáhneme nahrání servletu s Shibboleth IdP:

# příkaz zadaný do terminálu:
 
/etc/init.d/jetty restart

Nyní můžeme vyzkoušet, zda-li Shibboleth IdP běží (je potřeba, aby Jetty běželo na HTTP/port 80, jinak je nutno použít parametr -u):

# příkazy zadané do terminálu:
 
cd /opt/shibboleth-idp/
./bin/status.sh

Pokud vše proběhlo bez problémů, uvidíte následující výstup:

### Operating Environment Information
operating_system: Linux
operating_system_version: 3.16.0-4-amd64
operating_system_architecture: amd64
jdk_version: 1.8.0_51
available_cores: 1
used_memory: 192 MB
maximum_memory: 958 MB

### Identity Provider Information
idp_version: 3.1.2
start_time: 2015-08-05T12:15:13+02:00
current_time: 2015-08-05T12:15:14+02:00
uptime: 633 ms

service: shibboleth.LoggingService
last successful reload attempt: 2015-08-05T07:12:19Z
last reload attempt: 2015-08-05T07:12:19Z

service: shibboleth.ReloadableAccessControlService
last successful reload attempt: 2015-08-05T07:12:23Z
last reload attempt: 2015-08-05T07:12:23Z

service: shibboleth.MetadataResolverService
last successful reload attempt: 2015-08-05T07:12:23Z
last reload attempt: 2015-08-05T07:12:23Z

        metadata source: ShibbolethMetadata

service: shibboleth.RelyingPartyResolverService
last successful reload attempt: 2015-08-05T07:12:22Z
last reload attempt: 2015-08-05T07:12:22Z

service: shibboleth.NameIdentifierGenerationService
last successful reload attempt: 2015-08-05T07:12:22Z
last reload attempt: 2015-08-05T07:12:22Z

service: shibboleth.AttributeResolverService
last successful reload attempt: 2015-08-05T07:12:22Z
last reload attempt: 2015-08-05T07:12:22Z

service: shibboleth.AttributeFilterService
last successful reload attempt: 2015-08-05T07:12:22Z
last reload attempt: 2015-08-05T07:12:22Z

Můžete také zkusit ze svého počítače přístoupit na URL adresu s IdP: https://HOSTNAME/idp. Nicméně neuvidíte nic zajímavého: 
