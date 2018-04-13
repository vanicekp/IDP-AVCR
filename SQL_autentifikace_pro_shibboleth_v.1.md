# Autentifikace Shibbolethu proti mysql databázi

## předpokládejme tabulku v mysql

```
mysql> show fields from  user;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| user  | varchar(255) | YES  |     | NULL    |       |
| pass  | varchar(255) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
```

Návod vychází z http://shibboleth.1660669.n2.nabble.com/IdP-user-authendication-using-mysql-user-table-td7620120.html.

Stahneme z 
```  
https://github.com/tauceti2/jaas-rdbms
```

A vytvoříme soubor `tagishauth.jar` příkazem make. Byla nutná editace makefile, nebyl k dispozici příkaz `jar` ale `gjar`.

Soubor nakopírujeme do 
```
cp jaas-rdbms-master/tagishauth.jar /opt/idp/idp.foo.cas.cz/webapp/WEB-INF/lib/
```
a přegenerujeme `idp.war`
```
./bin/build.sh
```

v adresáři `conf/authn` vytvoříme nový soubor `jaas.config` s obsahem
```
ShibUserPassAuth {
   com.tagish.auth.DBLogin required debug=true dbDriver="com.mysql.jdbc.Driver"
       dbURL="jdbc:mysql://localhost:3306/xxxjmenodatabazexxx"
       dbUser="xxxuserxxx" dbPassword="xxxhesloxxx" userTable="user"
       userColumn="user" passColumn="pass";
};
```
Po restartu by to mělo chodit.

Pozor bude pak nutne zcela překopat získávání atributů z ldapu.

#### todo

algorithm="SHA-256" ????

https://www.eduid.cz/cs/tech/idp/shibboleth/mysqlauth


## Konfigurace čtení atributů z mysql databáze vzorový příklad.
### Tabulka v databázi
```
mysql> show fields from  tel;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| user  | varchar(255) | YES  |     | NULL    |       |
| tel   | varchar(255) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
2 rows in set (0.00 sec)
```

### Konfigurace Data connectoru
```
<!-- MySQL Data Connector -->
<DataConnector id="mySQL"
    xsi:type="RelationalDatabase">
    <ApplicationManagedConnection
        jdbcDriver="com.mysql.jdbc.Driver"
        jdbcURL="jdbc:mysql://localhost:3306/shibboleth"
        jdbcUserName="user"
        jdbcPassword="heslo"/>

        <QueryTemplate>
            <![CDATA[
                SELECT * FROM tel WHERE user='$requestContext.principalName'
            ]]>
        </QueryTemplate>

        <Column columnName="tel" attributeID="telephoneNumber" />
```
### Konfigurace atributu
```
    <AttributeDefinition xsi:type="Simple" id="telephoneNumber" sourceAttributeID="telephoneNumber">
        <Dependency ref="mySQL" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:telephoneNumber" encodeType="false" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.20" friendlyName="telephoneNumber" encodeType="false" />
    </AttributeDefinition>
```



