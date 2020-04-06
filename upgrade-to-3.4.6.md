# Upgrade IdP na 3.4.6 IdP

Nejprve instalace nové verze shibbolethu:

```
cd /opt/dist
cp -r ../src/shibboleth-identity-provider-3.4.6 idp.ssc.cas.cz-source
cd idp.ssc.cas.cz-source
```
Úprava web.xml
```
vi webapp/WEB-INF/web.xml
```
```
<context-param>
    <param-name>idp.home</param-name>
    <param-value>/opt/idp/idp.foo.cas.cz</param-value>
</context-param>
```
#### Instalce
```
./bin/install.sh
Source (Distribution) Directory: [/opt/dist/idp.foo.cas.cz-source]
 
Installation Directory: [/opt/shibboleth-idp]
/opt/idp/idp.foo.cas.cz
```
#### Změna vlastníka adresáře
```
cd /opt/idp/idp.foo.cas.cz
chown -R idp:idp .
```
#### Restart a mělo by to naběhnout
```
time /etc/init.d/jetty restart
```

### Úpravy pro omezení chybových hlášek
#### idp.properties
