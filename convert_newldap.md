# Konverze IdP pro openLDAP z EIS3
## Příprava virtuálu
Prostá konverze virtuálu na jiné jméno a změna IP adresy a hostnem použito nmtui. Přesun všech `idp*.xml` z `/opt/jetty/webapps/` do zálohy. A start jetty.
Zakomentování hlídacích scriptů v crontab. Úprava scriptů a odkomentování monitorovacího.
```
systemctl disable stunnel
systemctl stop stunnel
```
Úprava `JAVA_HOME` na `/usr/lib/jvm/java-11-openjdk-11.0.6.10-0.el8_1.x86_64` v souboru `.bashrc`.

## Upgrade IdP na 4.1.0
```
cd dist/
cp -r ../src/shibboleth-identity-provider-4.1.0 idp.foo.cas.cz-source
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

## Konverze konfigurace
###Změna typu autentifikace z jaas na LDAP, je to v souboru `conf/authn/password-authn-config.xml`.

```
    <!-- Choose an import based on the back-end you want to use. -->
    <!-- <import resource="jaas-authn-config.xml" /> -->
    <!-- <import resource="krb5-authn-config.xml" /> -->
    <import resource="ldap-authn-config.xml" />
```

###Konfigurace LDAP a LDAP účtu pro prohledávání stromu, soubor `conf/ldap.properties`.
```
# LDAP authentication configuration, see authn/ldap-authn-config.xml
# Note, this doesn't apply to the use of JAAS

## Authenticator strategy, either anonSearchAuthenticator, bindSearchAuthenticator, directAuthenticator, adAuthenticator
idp.authn.LDAP.authenticator                   = bindSearchAuthenticator

## Connection properties ##
idp.authn.LDAP.ldapURL                          = ldaps://ldap1.eis.cas.cz
idp.authn.LDAP.useStartTLS                     = false
#idp.authn.LDAP.useSSL                          = false
#idp.authn.LDAP.connectTimeout                  = 3000

## SSL configuration, either jvmTrust, certificateTrust, or keyStoreTrust
idp.authn.LDAP.sslConfig                       = certificateTrust
## If using certificateTrust above, set to the trusted certificate's path
idp.authn.LDAP.trustCertificates                = %{idp.home}/credentials/ldap-server.crt
## If using keyStoreTrust above, set to the truststore path
idp.authn.LDAP.trustStore                       = %{idp.home}/credentials/ldap-server.truststore

## Return attributes during authentication
## NOTE: there is a separate property used for attribute resolution
idp.authn.LDAP.returnAttributes                 = passwordExpirationTime,loginGraceRemaining

## DN resolution properties ##

# Search DN resolution, used by anonSearchAuthenticator, bindSearchAuthenticator
# for AD: CN=Users,DC=example,DC=org
idp.authn.LDAP.baseDN                           = cn=users,dc=eis,dc=cas,dc=cz
#idp.authn.LDAP.subtreeSearch                   = true
idp.authn.LDAP.userFilter                       = (&(cn={user})(employeeNumber=47*)(orclisenabled=ENABLED))
# bind search configuration
# for AD: idp.authn.LDAP.bindDN=adminuser@domain.com
idp.authn.LDAP.bindDN                           = cn=eduid,cn=tech,dc=eis,dc=cas,dc=cz
idp.authn.LDAP.bindDNCredential                 = XXXXXXXXXXXXXXXX
idp.ldaptive.provider            = org.ldaptive.provider.unboundid.UnboundIDProvider
# Format DN resolution, used by directAuthenticator, adAuthenticator
# for AD use idp.authn.LDAP.dnFormat=%s@domain.com
idp.authn.LDAP.dnFormat                         = uid=%s,ou=people,dc=example,dc=org

# LDAP attribute configuration, see attribute-resolver.xml
# Note, this likely won't apply to the use of legacy V2 resolver configurations
idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
idp.attribute.resolver.LDAP.bindDNCredential    = %{idp.authn.LDAP.bindDNCredential:undefined}
idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
idp.attribute.resolver.LDAP.searchFilter        = (cn=$resolutionContext.principal)
idp.attribute.resolver.LDAP.connectTimeout      = 300
idp.attribute.resolver.LDAP.responseTimeout     = 300
#idp.attribute.resolver.LDAP.returnAttributes    = cn,homephone,mail

# LDAP pool configuration, used for both authn and DN resolution
#idp.pool.LDAP.minSize                          = 3
#idp.pool.LDAP.maxSize                          = 10
#idp.pool.LDAP.validateOnCheckout               = false
#idp.pool.LDAP.validatePeriodically             = true
#idp.pool.LDAP.validatePeriod                   = 300
#idp.pool.LDAP.prunePeriod                      = 300
#idp.pool.LDAP.idleTime                         = 600
#idp.pool.LDAP.blockWaitTime                    = 3000
#idp.pool.LDAP.failFastInitialize               = false
```
Je to v templates `cp /opt/templates/shibboleth/eis3/ldap.properties conf/`. Pozor na úpravu filtru je shodný s filtrem v jaas.conf a nebo se dají použít operátory pro ldapsearch viz `http://www.ldapexplorer.com/en/manual/109010000-ldap-filter-syntax.htm`, pro příklad kombinovaný filtr pro SSČ.
```
idp.authn.LDAP.userFilter                       = (&(cn={user})(|(employeeNumber=90*)(employeeNumber=50*)(employeeNumber=47*)(employeeNumber=44*))(orclisenabled=ENABLED))
```

a nezapomenout nakopírovat certifikát pro LDAP CA `cp /opt/templates/shibboleth/eis3/ldap-server.crt credentials/`.

### Varování při nové instalci
Při nové čerstvé instalci je nově `idp.authn.LDAP.bindDNCredential` v souboru `credentials/secrets.properties` a taky je tam `idp.persistentId.salt` ze souboru 
`conf/saml-nameid.properties`. Hlavně `idp.authn.LDAP.bindDNCredential` je problém protože přepisuje hodnotu z `idp.authn.LDAP.bindDNCredential`.

### Změna filtrace uvolňování attributů
```
cd conf/intercept
cp consent-intercept-config.xml.idpnew consent-intercept-config.xml
cd ../..
```


### Úprava attribute-resolver.xml
#### Úprava LDAP Connector
Musí se změnit LDAP Connector tak aby používal nový LDAP
```
  <!-- LDAP Connector -->
    <DataConnector id="myLDAP" xsi:type="LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}"
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
        useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}"
        connectTimeout="%{idp.attribute.resolver.LDAP.connectTimeout}"
                trustFile="%{idp.attribute.resolver.LDAP.trustCertificates}"
        responseTimeout="%{idp.attribute.resolver.LDAP.responseTimeout}">
        <FilterTemplate>
            <![CDATA[
                %{idp.attribute.resolver.LDAP.searchFilter}
            ]]>
        </FilterTemplate>
            <ConnectionPool
            minPoolSize="%{idp.pool.LDAP.minSize:3}"
            maxPoolSize="%{idp.pool.LDAP.maxSize:10}"
            blockWaitTime="%{idp.pool.LDAP.blockWaitTime:PT3S}"
            validatePeriodically="%{idp.pool.LDAP.validatePeriodically:true}"
            validateTimerPeriod="%{idp.pool.LDAP.validatePeriod:PT5M}"
            expirationTime="%{idp.pool.LDAP.idleTime:PT10M}" />
    </DataConnector>
```
#### Úprava attributů
Protože LDAP EIS3 má jinak pojmenované attributy je třeba provést následující záměny v konfiguračním souboru:
```
businesscategory  -->  businessCategory    1x
employeenumber -->  employeeNumber    2x
givenname --> givenName  5x
```

### Konfigurace jetty
`cp /root/webapps/idp.Foo.cas.cz.xml /opt/jetty/webapps/`

