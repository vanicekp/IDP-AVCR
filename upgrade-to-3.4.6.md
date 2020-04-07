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
```
vi conf/idp.properties
```
Změna položky `idp.cookie.secure =  true`


#### attribute-resolver.xml
Pro složitost a množství změn je lepší nahradit soubor novým z matrice a změnit jen položky: {ID-foo-number}, {Foo}, {NAME}.
```
<?xml version="1.0" encoding="UTF-8"?>
<AttributeResolver
        xmlns="urn:mace:shibboleth:2.0:resolver"
        xmlns:sec="urn:mace:shibboleth:2.0:security"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="urn:mace:shibboleth:2.0:resolver http://shibboleth.net/schema/idp/shibboleth-attribute-resolver.xsd
                            urn:mace:shibboleth:2.0:security http://shibboleth.net/schema/idp/shibboleth-security.xsd">

    <!-- ========================================== -->
    <!--      Attribute Definitions                 -->
    <!-- ========================================== -->
<!-- eduPersonPrincipalName -->
<AttributeDefinition id="eduPersonPrincipalName" xsi:type="Scoped" scope="%{idp.scope}">
    <InputDataConnector ref="myLDAP" attributeNames="cn"/>
    <AttributeEncoder xsi:type="SAML1ScopedString" name="urn:mace:dir:attribute-def:eduPersonPrincipalName" encodeType="false" />
    <AttributeEncoder xsi:type="SAML2ScopedString" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.6" friendlyName="eduPersonPrincipalName" encodeType="false" />
</AttributeDefinition>

<!-- commonName -->
    <AttributeDefinition xsi:type="Template" id="commonName">
        <InputDataConnector ref="myLDAP" attributeNames="givenname sn"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:cn" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.3" friendlyName="cn" />
        <Template>${givenname} ${sn}</Template>
    </AttributeDefinition>

<!-- givenName -->
   <AttributeDefinition xsi:type="Simple" id="givenName">
        <InputDataConnector ref="myLDAP" attributeNames="givenname"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:givenName" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.42" friendlyName="givenName" />
    </AttributeDefinition>

<!-- displayName -->
   <AttributeDefinition xsi:type="Template" id="displayName">
   <InputDataConnector ref="myLDAP" attributeNames="givenname sn"/>
   <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:displayName" />
   <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.16.840.1.113730.3.1.241" friendlyName="displayName" />
        <Template>${givenname} ${sn}</Template>
    </AttributeDefinition>

<!-- surname -->
<AttributeDefinition id="surname" xsi:type="Simple">
    <InputDataConnector ref="myLDAP" attributeNames="sn"/>
    <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:sn" encodeType="false" />
    <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.4" friendlyName="sn" encodeType="false" />
</AttributeDefinition>

<!-- mail -->
    <AttributeDefinition xsi:type="Simple" id="email">
    <InputDataConnector ref="myLDAP" attributeNames="mail"/>
    <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:mail" />
    <AttributeEncoder xsi:type="SAML2String" name="urn:oid:0.9.2342.19200300.100.1.3" friendlyName="mail" />
    </AttributeDefinition>

<!-- commonName#ASCII pro TCS-P -->
    <AttributeDefinition xsi:type="ScriptedAttribute" id="commonNameASCII">
    <InputAttributeDefinition ref="commonName" />
    <AttributeEncoder xsi:type="SAML2String" name="http://eduid.cz/attributes/commonName#ASCII" friendlyName="commonNameASCII" />
    <ScriptFile>/opt/idp/common/script/commonNameASCII.js</ScriptFile>
    </AttributeDefinition>

<!-- Schema: eduPerson attributes -->
    <AttributeDefinition xsi:type="Simple" id="eduPersonAffiliation">
    <InputDataConnector ref="staticAttributes" attributeNames="eduPersonAffiliation" />
    <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:eduPersonAffiliation" />
    <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.1" friendlyName="eduPersonAffiliation" />
    </AttributeDefinition>

<!-- eduPersonScopedAffiliation -->
    <AttributeDefinition xsi:type="Scoped" id="eduPersonScopedAffiliation" scope="%{idp.scope}">
    <InputDataConnector ref="staticAttributes" attributeNames="eduPersonAffiliation"/>
    <AttributeEncoder xsi:type="SAML1ScopedString" name="urn:mace:dir:attribute-def:eduPersonScopedAffiliation" />
    <AttributeEncoder xsi:type="SAML2ScopedString" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.9" friendlyName="eduPersonScopedAffiliation" />
    </AttributeDefinition>

<!-- employeeNumber -->
    <AttributeDefinition xsi:type="Simple" id="employeeNumber">
        <InputDataConnector ref="myLDAP" attributeNames="employeenumber"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:employeeNumber" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.16.840.1.113730.3.1.3" friendlyName="employeeNumber" />
    </AttributeDefinition>

<!-- unstructuredName pro TCS-P -->
    <AttributeDefinition xsi:type="Mapped" id="unstructuredName">
        <InputDataConnector ref="myLDAP" attributeNames="employeenumber"/>
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.2.840.113549.1.9.2" friendlyName="unstructuredName" />
        <ValueMap>
            <ReturnValue>$1</ReturnValue>
            <SourceValue>{ID-foo-number}(.+)</SourceValue>
        </ValueMap>
    </AttributeDefinition>

<!-- eduPersonUniqueId -->
<AttributeDefinition id="eduPersonUniqueId" xsi:type="Scoped" scope="%{idp.scope}">
    <InputAttributeDefinition ref="unstructuredName" />
    <DisplayName xml:lang="cs">Unikátní, neměnný identifikátor uživatele</DisplayName>
    <DisplayName xml:lang="en">Unique life-time persistent user identifier</DisplayName>
    <AttributeEncoder xsi:type="SAML2ScopedString" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.13" friendlyName="eduPersonUniqueId" encodeType="false" />
</AttributeDefinition>

<!-- uniqueIdentifier -->
    <AttributeDefinition xsi:type="Simple" id="uniqueIdentifier">
    <InputDataConnector ref="myLDAP" attributeNames="businesscategory" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:uniqueIdentifier" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:0.9.2342.19200300.100.1.44" friendlyName="uniqueIdentifier" />
    </AttributeDefinition>

<!-- mail  pro TCS-P -->
    <AttributeDefinition xsi:type="Simple" id="tcsmail">
        <InputDataConnector ref="myLDAP" attributeNames="mail" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:tcsmail" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.2.840.113549.1.9.1" friendlyName="tcsmail" />
    </AttributeDefinition>

<!-- organizationName -->
     <AttributeDefinition id="organizationName" xsi:type="Simple">
        <InputDataConnector ref="staticAttributes" attributeNames="organizationName"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:o" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.10" friendlyName="o" />
     </AttributeDefinition>

<!-- schacHomeOrg -->
     <AttributeDefinition id="schacHomeOrg" xsi:type="Simple">
        <InputDataConnector ref="staticAttributes" attributeNames="schacHomeOrg"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:oid:1.3.6.1.4.1.25178.1.2.9" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.25178.1.2.9" friendlyName="schacHomeOrg" />
     </AttributeDefinition>

<!-- eduPersonEntitlement pro TCS-P -->
   <AttributeDefinition xsi:type="ScriptedAttribute" id="eduPersonEntitlement" >
        <InputAttributeDefinition ref="unstructuredName" />
        <InputAttributeDefinition ref="uniqueIdentifier" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:eduPersonEntitlement" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.7" friendlyName="eduPersonEntitlement" />
        <ScriptFile>/opt/idp/common/script/eduPersonEntitlement{Foo}.js</ScriptFile>
    </AttributeDefinition>

<!-- eduPersonTargetedID  -->
     <AttributeDefinition id="eduPersonTargetedID" xsi:type="SAML2NameID"  nameIdFormat="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
        <InputDataConnector ref="myStoredId" attributeNames="persistentID"/>
        <AttributeEncoder xsi:type="SAML1XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" />
        <AttributeEncoder xsi:type="SAML2XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" friendlyName="eduPersonTargetedID" />
     </AttributeDefinition>


    <!-- ========================================== -->
    <!--      Data Connectors                       -->
    <!-- ========================================== -->

    <!--
        LDAP Connector
        https://wiki.shibboleth.net/confluence/display/SHIB2/ResolverLDAPDataConnector
    -->
    <DataConnector id="myLDAP" xsi:type="LDAPDirectory"
        ldapURL="ldap://localhost:50000"
        baseDN="cn=Users,dc=eis,dc=cas,dc=cz"
        >
        <FilterTemplate>
            <![CDATA[
                (cn=$resolutionContext.principal)
            ]]>
        </FilterTemplate>
    </DataConnector>

<!-- Static Data Connector -->

<DataConnector id="staticAttributes" xsi:type="Static">
    <Attribute id="organizationName">
        <Value>{NAME}</Value>
    </Attribute>
        <Attribute id="eduPersonAffiliation">
            <Value>staff</Value>
            <Value>employee</Value>
            <Value>member</Value>
        </Attribute>
    <Attribute id="schacHomeOrg">
        <Value>%{idp.scope}</Value>
    </Attribute>
</DataConnector>

<DataConnector id="myStoredId"
    xsi:type="StoredId"
    generatedAttributeID="persistentID"
    salt="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    queryTimeout="P0Y0M0DT0H0M0.000S">
    <BeanManagedConnection>shibboleth.MySQLDataSource</BeanManagedConnection>
    <InputAttributeDefinition ref="eduPersonPrincipalName" />
</DataConnector>
```
