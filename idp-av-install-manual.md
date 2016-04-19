# Oracle JDK

Ačkoliv je v linuxových distribucích mnohdy možnost nainstalovat Javu pomocí b
alíčkovacího systému dané distribuce, např. OpenJDK, silně doporučujeme to, co S
hibboleth konzorcium. Použijeme tedy Javu od Oracle. Čas od času se objeví nějaký pr
oblém způsobený použitím např. OpenJDK. Budeme-li žádat o podporu, může se stát, že b
udeme vyzváni, abychom problém reprodukovali s využitím Javy od společnosti Oracle.

Po stažení Oracle JDK umístíme archiv do adresáře /opt/src a pomocí následujících příkazů J
DK nainstalujeme:

### Rozbalení a instalace Oracle JDK
```
cd /opt
tar -xzf src/jdk-8u77-linux-x64.tar.gz
alternatives --install /usr/bin/java java /opt/jdk1.8.0_77/bin/java 120
alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_77/bin/javac 120
echo export JAVA_HOME=/opt/jdk1.8.0_77 >> ~/.bashrc
```
O korektním nastavení právě nainstalovaného Oracle JDK se můžeme přesvědčit následovně.

### Příkaz pro zobrazení aktuálně využívané verze Javy
```
alternatives --display java
```

### Výstup příkazu pro zobrazení aktuálně využívané verze Javy
```
java - auto mode
  link currently points to /opt/jdk1.8.0_77/bin/java
/opt/jdk1.8.0_77/bin/java - priority 120
Current 'best' version is '/opt/jdk1.8.0_77/bin/java'.
```
Můžeme se také přesvědčit zavoláním příkazu java.

### Příkaz pro zobrazení aktuálně využívané verze Javy
```java -version```

### Výstup příkazu pro zobrazení aktuálně využívané verze Javy
```
java version "1.8.0_77"
Java(TM) SE Runtime Environment (build 1.8.0_77-b03)
Java HotSpot(TM) 64-Bit Server VM (build 25.77-b03, mixed mode)
```

Zda máme správně nastavenou proměnnou prostřední $JAVA_HOME zjistíme, pokud znovu načteme konfigurační soubor interpretu BASH a vyžádáme si hodnotu proměnné $JAVA_HOME.

### Příkaz pro zobrazení hodnoty proměnné $JAVA_HOME
```
source ~/.bashrc && echo $JAVA_HOME
```

### Výstup příkazu pro zobrazení hodnoty proměnné $JAVA_HOME
```
/opt/jdk1.8.0_77
```
## Java Cryptography Extension (Unlimited Strength Jurisdiction Policy Files)

Po nainstalování Oracle JDK je ještě potřeba doinstalovat tzv. JCE US (Java Cryptography Extension Unlimited Strength), které zajistí možnost využít silnější šifrování.

Stáhneme JCE a umístíme jej do adresáře /opt/src.
```
http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html`
```
### Rozbalení archivu s JCE US
```
cd /opt
unzip -x src/jce_policy-8.zip
```

Pro „instalaci“ JCE US stačí zkopírovat dva následující JAR (Java ARchive) soubory do Oracle JDK:

# Instalace JCE US
```
cp UnlimitedJCEPolicyJDK8/US_export_policy.jar jdk1.8.0_77/jre/lib/security/
cp UnlimitedJCEPolicyJDK8/local_policy.jar jdk1.8.0_77/jre/lib/security/
```
Adresář s rozbaleným rozšířením JCE US můžeme nyní smazat.

### Smazání rozbaleného archivu s JCE US
```
rm -rf UnlimitedJCEPolicyJDK8/
```
