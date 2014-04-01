# IdP pro AVČR krok za krokem (for dummies)

## Úvod

Návod popisuje podrobně potřebné kroky pro přidání další instance Shibboleth IdP pro součást AVČR. Podrobnější informace lze získat v dokumentu:

* [IdP pro AVČR](https://gist.github.com/ivan-novakov/3f3a31c117fc09c506bc)

Předpoklady:

* chceme zřídit IdP instanci pro fiktivní součást **FOO AVČR**
* hostname bude **idp.foo.cas.cz**

## Apache

Ověříme dostupnost CNAME  idp.foo.cas.cz

Vytvoříme nový virtuál na základě šablony `/opt/templates/apache/idp-xxx.conf`:

```
cp /opt/templates/apache/idp-xxx.conf /etc/httpd/virtuals.d/idp-foo.conf
```

V nově vytvořeném souboru `idp-foo.conf` nastavíme proměnnou `{HOSTNAME}` na **idp.foo.cas.cz**.

Dále je třeba založit adresář pro logy

```
mkdir /var/log/httpd/idp.foo.cas.cz
```

Pro aktivaci změn musíme Apache restartovat:

```
/etc/init.d/httpd restart
```

## Tomcat

Do konfiguračního souboru `/etc/tomcat6/server.xml` přidáme definici hostu:

```xml
<Host name="idp.foo.cas.cz"  appBase="webapps"
    unpackWARs="true" autoDeploy="true"
    xmlValidation="false" xmlNamespaceAware="false">
</Host>
```

Vytvoříme příslušný adresář:

```
mkdir /etc/tomcat6/Catalina/idp.foo.cas.cz
```

Změníme vlastníka
```
chown tomcat /etc/tomcat6/Catalina/idp.foo.cas.cz
````

Pro aktivaci změn muséme Tomcat restartovat:

```
/etc/init.d/tomcat6 restart
```

## Shibboleth

### Zdrojový adresář

Vytvoříme kopii zdrojového adresáře:

```
cd /opt/dist
cp -r shibboleth-identityprovider-2.4.x idp.foo.cas.cz-source
```

Pro jednoduchost si uložíme zdrojový adresář do proměnné `IDP_SRC`:

```
export IDP_SRC=/opt/dist/idp.foo.cas.cz-source
```

Zkopírujeme MySQL knihovnu do adresáře `$IDP_SRC/lib/`:

```
cp /opt/dist/lib/mysql-connector-java-x.x.x/mysql-connector-java-5.x.x.jar $IDP_SRC/lib/
```

V souboru `$IDP_SRC/src/main/webapp/WEB-INF/web.xml` je potřeba v definici servletu _UsernamePasswordAuthHandler_ přidat u parametru _jaasConfigName_ nějakou unikátní hodnotu, což v našem případě může být **ShibUserPassAuthFoo** (prakticky to znamená, že přidáme celou sekci _init-param_, která ve výchozím stavu souboru chybí):

```xml
<servlet>
    <servlet-name>UsernamePasswordAuthHandler</servlet-name>
    <servlet-class>edu.internet2.middleware.shibboleth.idp.authn.provider.UsernamePasswordLoginServlet</servlet-class>
    <load-on-startup>3</load-on-startup>
    <init-param>
        <param-name>jaasConfigName</param-name>
        <param-value>ShibUserPassAuthFoo</param-value>
    </init-param>
</servlet>
```

### Přihlašovací stránka

Přihlašovací stránku upravíme editací souboru  `$IDP_SRC/ rc/main/webapp/login.jsp`

Logo je v adresáři `$IDP_SRC/src/main/weapp/images`

### Instalace

Před spuštěním instalace je potřeba nastavit proměnnou protředí `JAVA_HOME`:

```
export JAVA_HOME=/usr/lib/jvm/jre-1.7.0-openjdk.x86_64
```

Spustíme instalační skript:

```
cd $IDP_SRC
* cílový adresář - `/opt/idp/idp.foo.cas.cz`
* hostname - `idp.foo.cas.cz`
* heslo do keystore - libovolná hodnota, keystore se prakticky nepoužívá

```
Where should the Shibboleth Identity Provider software be installed? [/opt/shibboleth-idp]
/opt/idp/idp.foo.cas.cz
What is the fully qualified hostname of the Shibboleth Identity Provider server? [idp.example.org]
idp.foo.cas.cz
A keystore is about to be generated for you. Please enter a password that will be used to protect it.
heslo
```

Pro jednoduchost si uložíme adresář s instalaci Shibboleth IdP do proměnné `IDP_HOME`:

```
export IDP_HOME=/opt/idp/idp.foo.cas.cz
```

### Metadata

Instalační skript vygeneruje základní podobu metadat a uloží je v souboru `$IDP_HOME/metadata/idp-metadata.xml`. Před registraci do eduID.cz je potřeba přidat potřebné elementy definované v [profilu metadat pro eduID.cz](http://www.eduid.cz/en/tech/metadata-profile). Neméně důležité je správně nastavit _Scope_ element, který je po instalaci nastaven jako `cas.cz`
. Např. pro _idp.ssc.cas.cz_ bude hodnota _Scope_ `ssc.cas.cz` (**pozor jsou tam dva výskyty**):

```xml
<Extensions>
    <shibmd:Scope regexp="false">ssc.cas.cz</shibmd:Scope>
</Extensions>
```

Bylo by také vhodné, když už IdP podporuje persistentní ID, promítnout to do metadat přidáním dalšího formátu _NameID_ (viz [dokumentace na eduID.cz](http://eduid.cz/cs/tech/idp/persistent-id#aktualizace_metadat)):

```xml
<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>
```

Pro inspiraci se můžete podívat na příklad metadat pro SSC AVČR - `/opt/templates/metadata/idp.ssc.cas.cz-metadata.xml`.

Loga dáváme do adresáře `/var/www/html/loga`

### Konfigurace

#### login.config

Do souboru `/opt/idp/common/conf/login.config` je potřeba přidat konfiguraci pro autentizaci pomocí LDAP. Nazev konfigurační sekce musí odpovídat hodnotě, kterou jsme uvedli u parametru _jaasConfigName_ v souboru `$IDP_SRC/src/main/webapp/WEB-INF/web.xml`. V našem případě je to **ShibUserPassAuthFoo**:

```
// idp.foo.cas.cz
ShibUserPassAuthFoo {
   edu.vt.middleware.ldap.jaas.LdapLoginModule required
      ldapUrl="ldaps://fis1.eis.cas.cz:3232"
      baseDn="cn=Users,dc=eis,dc=cas,dc=cz"
      userFilter="(&(cn={0})(employeenumber={PREFIX}*)(businesscategory=EduID))";
};
```

**Pozor:** Je potřeba nastavit hodnotu `{PREFIX}` na dvojčíslí odpovídající dané součásti AVČR. Například pro SSC AVČR hodnota _userFilter_ vypadá takto:

```
userFilter="(&(cn={0})(employeenumber=47*)(businesscategory=EduID))
```

#### handler.xml

V souboru `$IDP_HOME/conf/handler.xml` je potřeba odkomentovat _UsernamePassword_ handler a nastavit cestu k společnému JAAS konfiguračnímu souboru:

```xml
<!--  Username/password login handler -->
<ph:LoginHandler xsi:type="ph:UsernamePassword"
    jaasConfigurationLocation="file:///opt/idp/common/conf/login.config">
    <ph:AuthenticationMethod>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</ph:AuthenticationMethod>
</ph:LoginHandler>
```

Anebo je možné použít rovnou šablonu `/opt/templates/shibboleth/handler.xml` ,  POZOR je třeba  nastavit hodnotu proměné {HOSTNAME} na idp.foo.cas.cz.


```
cp /opt/templates/shibboleth/handler.xml $IDP_HOME/conf
```

#### logging.xml

V souboru `$IDP_HOME/conf/logging.xml` je dobré nastavit během konfigurace logování na _INFO_ nebo _DEBUG_. Píše to sice spoustu balastu (zvlášť pod _DEBUG_), ale pomůže to diagnostikovat případné problémy.

Pozor musí se nastavit práva na adresář `$IDP_HOME/logs`

```
mkdir tomcat $IDP_HOME/logs
```

#### relying-party.xml

Pro soubor `$IDP_HOME/conf/relying-party.xml` je možné použít šablonu `/opt/templates/shibboleth/relying-party.xml` a tam nastavit "proměnné":

* ENTITY_ID - entity ID instance, např. `https://idp.ssc.cas.cz/idp/shibboleth`
* HOSTNAME - hostname instance, např. `idp.ssc.cas.cz`.

V šabloně je nastavené stahování metadat pro testovací federaci czTestFed i pro eduID.cz. V obou případech se metadata zalohují v adresáři `$IDP_HOME/metadata/backup`. Je potřeba zajistit, že příslušný adresář existuje a je zapisovatelný pro Tomcat. Tedy v našem případě je potřeba udělat:

```
mkdir $IDP_HOME/metadata/backup
chown tomcat $IDP_HOME/metadata/backup
```

#### attribute-resolver.xml

Pro soubor `$IDP_HOME/conf/attribute-resolver.xml` lze použít šablonu `/opt/templates/attribute-resolver.xml` a nahradit "proměnné":

* SCOPE - příslušna hodnota _Scope_ tak, jak je uvedena v metadatech (viz výše), např. pro instanci _idp.ssc.cas.cz_ je to `ssc.cas.cz`. A je to tam 3x.
* ORGANIZATION - název organizace pro nastavení statického atributu
* XXXXX - na modifikaci jména scriptu `eduPersonEntitlementXxxx.js` 

A zkopírovat script s příslušným jménem a modifikovat hodnotu/y unstructuredname.

#### attribute-filter.xml

Pro soubor `$IDP_HOME/conf/attribute-filter.xml` lze použít šablonu `/opt/templates/shibboleth/attribute-filter.xml` bez nutnosti dalších úprav.

### Aktivace

Abychom aktivovali aplikaci, použijeme tzv. _context deployment fragment_. Je to v podstate XML soubor, který se nachází v aplikačním adresáři a který definuje, kde se nachází aplikační data (WAR soubor).

V našem případě je potřeba vytvořit soubor `/etc/tomcat6/Catalina/idp.foo.cas.cz/idp.xml` s následujícím obsahem:

```xml
<Context docBase="/opt/idp/idp.foo.cas.cz/war/idp.war"
         privileged="true"
         antiResourceLocking="false"
         antiJARLocking="false"
         unpackWAR="false"
         swallowOutput="true"
         cookies="false" />
```

Poté bychom měli Tomcat restartovat:

```
/etc/init.d/tomcat6 restart
```

### Pozdější změny

Pokud změníme něco v konfiguraci, je potřeba restartovat Tomcat.

Pokud změníme něco v zdrojovém adresáři `$IDP_SRC`, například upravíme přihlašovací stránku, je potřeba znovu spustit instalační skript `install.sh`. Ten se zeptá opět na cílový adresář, ale už by nám měl nabídnout adresář, kde byla instance už nainstalovaná, takže stačí volbu odklepnout. Skript pak zjistí, že v daném adresáři už byla provedena instalace a zeptá
se zda se má přepsat konfigurace. Odpovíme záporně (je to výchozí volba) a poté skript přepíše pouze binární soubory. Tomcat by měl detekovat změnu automaticky a restartovat aplikaci. Pokud se tak nestane, restartujte Tomcat ručně.



./install.sh
```

Skript po nás chce některé údaje. Nastavíme tyto hodnoty:
