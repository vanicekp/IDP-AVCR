#Přenos stávající instalace  IdP z Centos 6 na 8 a upgrade 

Je nutné mít instalaci ve veryi IdP 3.4.6 která při restartu negeneruje žádná varování v logu IdP.

## Příprava prostředí
Přeneseme z staré instalace minimálně adresář `/opt/template`, `/opt/idp/common`.

## Přenos IdP
```
mv idp/idp.foo.cas.cz /opt/idp
```
a
```
cp webapps/idp.foo.cas.cz.xml /opt/jetty/webapps/
```
a  restart jetty
```
/etc/init.d/jetty restart
```
Pokud v logu IdP není žádný zjevný problém můžeme pokračovat upgradem IdP na verzi 4.0.0

## Upgrade IdP
```
cd dist/
cp -r ../src/shibboleth-identity-provider-4.0.0 idp.foo.cas.cz-source
cd idp.ssc.cas.cz-source/
```
Editace `web.xml` doplní se za `<display-name>`
```
vi webapp/WEB-INF/web.xml
```
```
<context-param>
    <param-name>idp.home</param-name>
    <param-value>/opt/idp/idp.foo.cas.cz</param-value>
</context-param>
```
Spustí se instalace:
```
./bin/install.sh
```
Potvrdí se zdrojový adresář, zapíše cílový `/opt/idp/idp.foo.cas.cz`.
Nakonec restart a konrola logů.
```
/etc/init.d/jetty restart
```
