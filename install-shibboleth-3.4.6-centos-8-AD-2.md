# Instalace IdP

Stáhnout Shibboleth IdP a umístit do adresáře `/opt/src`.
```
http://shibboleth.net/downloads/identity-provider/
```
#### příkazy zadané do terminálu:
``` 
cd /opt/src
tar -xzf src/shibboleth-identity-provider-3.4.3.tar.gz
```
#### Příprava instalace
```
mkdir /opt/dist
```
```
cd /opt/dist
cp -r /opt/src/shibboleth-identity-provider-3.4.3 idp.foo.cas.cz-source
```
#### Změna idp.home
V souboru `idp.foo.cas.cz-source/webapp/WEB-INF/web.xml` musíme doplnit definici proměnné `idp.home`, jinak bude instalace předpokládat umístění v defaultním adresáři `/opt/shibboleth-idp`.
```
vi idp.foo.cas.cz-source/webapp/WEB-INF/web.xml
```
Za parametr `<display-name>` vložíme 
```
<context-param>
    <param-name>idp.home</param-name>
    <param-value>/opt/idp/idp.foo.cas.cz</param-value>
</context-param>
```

#### Instalce
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
#### Povolení Status URL

V souboru `/opt/idp/idp.foo.cas.cz/conf/access-control.xml` doplníme IP adresy správcovských stanic pro kontrolu 
statusu serveru.
```
cd /opt/idp/idp.foo.cas.cz/
vi conf/access-control.xml
```
Editujeme:
```
<entry key="AccessByIPAddress">
        <bean parent="shibboleth.IPRangeAccessControl"
        p:allowedRanges="#{ {'127.0.0.1/32', '::1/128', '147.231.12.0/22'} }" />
</entry>
```
#### Perzonifikace loga pro identifikaci virtuálu
Do adresáře `/opt/idp/idp.foo.cas.cz/edit-webapp/images` dáme místo prázdného loga logo pro rozlišení virtuálu při chybových hláškách. V opačném případě nepoznáme, který virtuál vygeneroval chybu, není to na stránce napsané.
```
cp  ~/loga/foo.png edit-webapp/images/dummylogo.png
chown -R idp:idp .
./bin/build.sh
Installation Directory: [/opt/idp/idp.test.cas.cz]
```

## Konfigurace Jetty virtuálu
Připravíme si konfigurační soubor idp.xml, pomocí něhož definujeme, který WAR (Web application ARchive) bude obsahovat webovou aplikaci našeho IdP a na jaké adrese (v tomto případě `https://HOSTNAME_SERVERU/idp`) bude přes web IdP naslouchat.

#### příkaz zadaný do terminálu:
``` 
vi /opt/jetty/webapps/idp.foo.cas.cz.xml
```
Obsah konfiguračního souboru `/opt/jetty/webapps/idp.foo.cas.cz.xml` je následující.
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

Restartujeme Jetty, čímž dosáhneme nahrání servletu s Shibboleth IdP:

#### příkaz zadaný do terminálu:
``` 
/etc/init.d/jetty restart
```
Nyní můžeme vyzkoušet, zdali Shibboleth IdP běží 

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

Můžete také zkusit ze svého počítače přístoupit na URL adresu s IdP: `https://idp.foo.cas.cz/idp`. Nicméně neuvidíte nic zajímavého. 

## JAAS autentifikace
Pro autentifikaci je vzhledem ke komplikovanému schématu nutno použít JAAS. Zdá se, že JETTY má pro každou virtuální instanci zvláštní instanci JAAS, takže není třeba harakiri se změnou názvu přihlašovací procedury. Konfigurace se provede v `conf/authn/password-authn-config.xml`, kde zakomentujeme ladap autentifikaci a povolíme JAAS.
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
V případě komplikovaných ústavů s více čísly je obsah `conf/authn/jaas.config` následující:
```
ShibUserPassAuth {
      org.ldaptive.jaas.LdapLoginModule sufficient
      ldapUrl="ldap://localhost:50000"
      baseDn="cn=Users,dc=eis,dc=cas,dc=cz"
      userFilter="(&(cn={user})(employeenumber={ID-foo-number1}*)(orclisenabled=ENABLED))";

   org.ldaptive.jaas.LdapLoginModule sufficient
      ldapUrl="ldap://localhost:50000"
      baseDn="cn=Users,dc=eis,dc=cas,dc=cz"
      userFilter="(&(cn={user})(employeenumber={ID-foo-number2}*)(orclisenabled=ENABLED))";

   org.ldaptive.jaas.LdapLoginModule sufficient
      ldapUrl="ldap://localhost:50000"
      baseDn="cn=Users,dc=eis,dc=cas,dc=cz"
      userFilter="(&(cn={user})(employeenumber={ID-foo-number3}*)(orclisenabled=ENABLED))";

};
```

## metadata-providers.xml
V souboru `metadata-providers.xml` je definice zdrojů metadat pro IDP. Nastavení provedeme zkopírováním z templates. Kopírujeme soubor `metadata-provaiders.xml` a podpisový klíč `metadata.eduid.cz.crt.pem`.
```
cp /opt/templates/shibboleth/metadata-providers.xml conf/
cp /opt/templates/shibboleth/metadata.eduid.cz.crt.pem credentials/
```

## attribute-resolver.xml
V souboru `attribute-resolver.xml` je definice získávání atributů z LDAPu, mysql, statické konfigurace. Pro náš účel použijeme matrici ze souboru `/opt/templates/shibboleth/attribute-resolver.xml`.
```
cp /opt/templates/shibboleth/attribute-resolver.xml conf
```
V souboru je třeba upravit `{ID-foo-number}`, v případě složitých ústavů se tento řádek zopakuje několikrát. Dále se upraví `{NAME}` na hodnotu odpovídající ústavu. Další je název scriptu pro nastavení `eduPersonEntitlement`. Nastavíme hodnotu `{Foo}`.

### Script eduPersonEntitlement
Vyrobíme kopii `/opt/idp/common/script/eduPersonEntitlementFoo.js` a upravíme osobní čísla zájmových osob. Kritický řádek pro 3 osoby vypadá :
```
if ((originalValue == "OS1") || (originalValue == "OS3") || (originalValue == "OS3")) {
```
## attribute-filter
Soubor `attribute-filter.xml` použijeme z `/opt/templates/shibboleth/attribute-filter.xml`.
```
cp /opt/templates/shibboleth/attribute-filter.xml conf
```
Netřeba žádných změn.

## Logování
V souboru `conf/logback.xml` změníme úroveň logování na `WARN` a `DEBUG`.
```
    <!-- Logging level shortcuts. -->
    <variable name="idp.loglevel.idp" value="${idp.loglevel.idp:-WARN}" />
    <variable name="idp.loglevel.ldap" value="${idp.loglevel.ldap:-DEBUG}" />
    <variable name="idp.loglevel.messages" value="${idp.loglevel.messages:-WARN}" />
    <variable name="idp.loglevel.encryption" value="${idp.loglevel.encryption:-INFO}" />
    <variable name="idp.loglevel.opensaml" value="${idp.loglevel.opensaml:-WARN}" />
    <variable name="idp.loglevel.props" value="${idp.loglevel.props:-INFO}" />
    <variable name="idp.loglevel.httpclient" value="${idp.loglevel.httpclient:-INFO}" />
```
## idp.cookie.secure 
V souboru `conf/idp.properties` je třeba upravit
```
idp.cookie.secure = true
```

## Konfigurace eduPersonTargetedID 
Konfigurace nutné v souboru `attribute-resolver.xml` již jsou v matrici.

Konfigurační soubor `global.xml` použijeme z matrice.
```
cp /opt/templates/shibboleth/global.xml conf/
```
Konfigurační soubor `saml-nameid.properties` použijeme z matrice.
```
cp /opt/templates/shibboleth/saml-nameid.properties conf/
```
Konfigurační soubor `saml-nameid.xml` použijeme z matrice.
```
cp /opt/templates/shibboleth/saml-nameid.xml conf/
```
Soubor `subject-c14n.xml` použijeme z matrice.
```
cp /opt/templates/shibboleth/subject-c14n.xml conf/c14n
```
V metadatech IdP je potřeba zadat, že IdP podporuje persistentní identifikátor.
```
vi metadata/idp-metadata.xml
```
Přidejte tedy do elementu `<IDPSSODescriptor>` následující řádky.  Patří to před:
```
        <SingleSignOnService Binding="urn:mace:shibboleth:1.0:profiles:AuthnRequest" Location="https://idp.usd.cas.cz/idp/profile/Shibboleth/SSO"/>
        <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" req-attr:supportsRequestedAttributes="true" Location="https://idp.usd.cas.cz/idp/profile/SAML2/POST/SSO"/>
        <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign" req-attr:supportsRequestedAttributes="true" Location="https://idp.usd.cas.cz/idp/profile/SAML2/POST-SimpleSign/SSO"/>
        <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" req-attr:supportsRequestedAttributes="true" Location="https://idp.usd.cas.cz/idp/profile/SAML2/Redirect/SSO"/>

```       
```
<NameIDFormat>urn:mace:shibboleth:1.0:nameIdentifier</NameIDFormat>
<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:transient</NameIDFormat>
<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>
```
Přidáme knihovnu JSTL do shibbolethu.
```
cp /opt/src/jstl-1.2.jar edit-webapp/WEB-INF/lib
```
Přegenerujte JAR Shibbolethu a restartujeme jetty.
```
chown -R idp:idp .
./bin/build.sh
/etc/init.d/jetty restart
```
## Perzonifikace login stránky a log a chybových hlášek
Logo umístíme do adresáře `edit-webapp/images`, optimální výška loga je 100px.
Soubory se "zprávami" umístíme z matrice do adresáře `messages`.
```
cp /opt/templates/shibboleth/messages/messages* messages/
```
V souborech `messages.properties` a `messages_cs.properties` doplníme správné `NAME` (2x) a `LOGO`.
Nahradíme logovací stánku upravenou o vložený text.
```
cp /opt/templates/shibboleth/login.vm views/
```
Přegenerujte JAR Shibbolethu a restartujeme jetty.
```
chown -R idp:idp .
./bin/build.sh
/etc/init.d/jetty restart
```
## Úprava metadat
Metadata nalezneme v adresáři metadata. V souboru je třeba doplnit několik údajů.
Hned za `EntityDescriptor` přijde vložení extension pro edugain.
```
    <Extensions xmlns="urn:oasis:names:tc:SAML:2.0:metadata">
        <eduidmd:RepublishRequest xmlns:eduidmd="http://eduid.cz/schema/metadata/1.0">
            <eduidmd:RepublishTarget>http://edugain.org/</eduidmd:RepublishTarget>
        </eduidmd:RepublishRequest>
    </Extensions>
```
Do `IDPSSODescriptor` extension vložíme informace o ústavu a logách.
```
            <mdui:UIInfo xmlns:mdui="urn:oasis:names:tc:SAML:metadata:ui">
                 <mdui:DisplayName xml:lang="en">English name institution AS CR</mdui:DisplayName>
                 <mdui:DisplayName xml:lang="cs">Český název ústavu AV ČR</mdui:DisplayName>
                 <mdui:Description xml:lang="en">Identity Provider FOO AV CR employees.</mdui:Description>
                 <mdui:Description xml:lang="cs">Identity Provider pro zaměstnance FOO AV ČR</mdui:Description>
                 <mdui:InformationURL xml:lang="en">http://www.foo.cas.cz/</mdui:InformationURL>
                 <mdui:InformationURL xml:lang="cs">http://www.foo.cas.cz/</mdui:InformationURL>
                 <mdui:Logo height="44" width="XX">https://gedeon.cas.cz/loga/logo-foo-44.png</mdui:Logo>
                 <mdui:Logo height="XX" width="XX">https://gedeon.cas.cz/loga/logo-foo-xx.png</mdui:Logo>
           </mdui:UIInfo>
```
Loga jsou v adresáři `/opt/jetty/webapps/root/loga/`

Přidáme informace o identifikatorech `NameIDFormat`  Pokud jsme to už neudělali dříve. Patří to před:
```
        <SingleSignOnService Binding="urn:mace:shibboleth:1.0:profiles:AuthnRequest" Location="https://idp.usd.cas.cz/idp/profile/Shibboleth/SSO"/>
        <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" req-attr:supportsRequestedAttributes="true" Location="https://idp.usd.cas.cz/idp/profile/SAML2/POST/SSO"/>
        <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign" req-attr:supportsRequestedAttributes="true" Location="https://idp.usd.cas.cz/idp/profile/SAML2/POST-SimpleSign/SSO"/>
        <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" req-attr:supportsRequestedAttributes="true" Location="https://idp.usd.cas.cz/idp/profile/SAML2/Redirect/SSO"/>

```

```
	<NameIDFormat>urn:mace:shibboleth:1.0:nameIdentifier</NameIDFormat>
	<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:transient</NameIDFormat>
        <NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>
```
Skoro na konec, před závěrečný tag `</EntityDescriptor>`, vložíme informace o ústavu a technickém kontaktu.
```
   <Organization>
      <OrganizationName xml:lang="en">foo AV CR</OrganizationName>
      <OrganizationName xml:lang="cs">foo AV ČR</OrganizationName>
      <OrganizationDisplayName xml:lang="en">English name institution</OrganizationDisplayName>
      <OrganizationDisplayName xml:lang="cs">Český název ústavu AV ČR</OrganizationDisplayName>
      <OrganizationURL xml:lang="en">http://www.foo.cas.cz/</OrganizationURL>
      <OrganizationURL xml:lang="cs">http://www.foo.cas.cz/</OrganizationURL>
    </Organization>

    <ContactPerson contactType="technical">
      <GivenName>Petr</GivenName>
      <SurName>Vaníček</SurName>
      <EmailAddress>mailto:vanicekp@utia.cas.cz</EmailAddress>
    </ContactPerson>
```
