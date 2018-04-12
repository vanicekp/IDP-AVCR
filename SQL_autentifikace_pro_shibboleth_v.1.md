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



