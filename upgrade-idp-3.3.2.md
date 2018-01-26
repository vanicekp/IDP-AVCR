# Postup aktualizace idp.foo.cas.cz na nejnovější verzi  shibboleth-identity-provider-3.3.2

### Nejprve zazálohovat stav:

```
cd /opt/idp
tar czvf idp.foo.cas.cz.tar.gz idp.foo.cas.cz/
```
### Vytvoření nové distribuce
```
cd /opt/dist
mv idp.foo.cas.cz-source idp.foo.cas.cz-source.3
```
```
cp -r ../src/shibboleth-identity-provider-3.3.2 idp.foo.cas.cz-source
cd idp.foo.cas.cz-source

vi webapp/WEB-INF/web.xml
```
Za parametr `<display-name>` vložíme
```
<context-param>
    <param-name>idp.home</param-name>
    <param-value>/opt/idp/idp.foo.cas.cz</param-value>
</context-param>
```
Spustíme instalaci
```
./bin/install.sh
```
a vyplníme 
```
Source (Distribution) Directory (press <enter> to accept default): [/opt/dist/idp.foo.cas.cz-source]

Installation Directory: [/opt/shibboleth-idp]
/opt/idp/idp.foo.cas.cz
Rebuilding /opt/idp/idp.foo.cas.cz/war/idp.war ...
...done

BUILD SUCCESSFUL
Total time: 23 seconds
```

Dále upravíme práva:
```
cd /opt/idp/idp.foo.cas.cz
chown -R idp:idp .
```

### A nakonec úprava souboru `attribute-resolver.xml`
Je třeba upravit syntaxi tak aby odpovídala novým požadavkům. Ono by to se starou chodilo, ale nová je jednoduší a přehlednější.

Napřed záloha
```
cp attribute-resolver.xml attribute-resolver.xml.bak
```
#### Úprava má dva kroky. 
První je úprava hlavičky souboru.

Vymění se celá hlavička až k komentáři 
```
    <!-- ========================================== -->
    <!--      Attribute Definitions                 -->
```
za 
```
<?xml version="1.0" encoding="UTF-8"?>
<AttributeResolver
        xmlns="urn:mace:shibboleth:2.0:resolver"
        xmlns:sec="urn:mace:shibboleth:2.0:security"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="urn:mace:shibboleth:2.0:resolver http://shibboleth.net/schema/idp/shibboleth-attribute-resolver.xsd
                            urn:mace:shibboleth:2.0:security http://shibboleth.net/schema/idp/shibboleth-security.xsd">
```


#### Druhá část je taková že se cely file prožene sed-em na odstranění přebytečných tagů.

```
cat attribute-resolver.xml | sed -e "s/resolver://g" | sed -e "s/enc://g" | sed -e "s/ad://g" | sed -e "s/dc://g" |sed -e "s/\"Script\"/\"ScriptedAttribute\"/g" > attribute-resolver.xml.1
```
a nakopíruje se nový konfigurák.
```
cp attribute-resolver.xml.1 attribute-resolver.xml
```
A restartne se jetty
```
/etc/init.d/jetty restart
```
Při restartu se v logu `logs/idp-proces.log` nesmí objevit žádný error.
