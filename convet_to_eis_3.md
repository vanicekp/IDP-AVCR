# Konverce IdP pro openLDAP z EIS3
## Příprava virtuálu
Prostá konverze virtuálu na jiné jméno a změna IP adresy a hostnem použito nmtui. Přesun všech `idp*.xml` z `/opt/jetty/webapps/` do zálohy. A start jetty.
Zakomentování hlídacích scriptů v crontab. Úprava scriptů a odkomentování monitorovacího.
```
systemctl disable stunnel
systemctl stop stunnel
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
idp.authn.LDAP.ldapURL                          = ldaps://auth1.eis.cas.cz
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
Je to v templates `cp /opt/templates/shibboleth/eis3/ldap.properties conf/`.

