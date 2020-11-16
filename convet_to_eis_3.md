# Konverce IdP pro openLDAP z EIS3
## Příprava virtuálu
Prostá konverze virtuálu na jiné jméno a změna IP adresy a hostnem použito nmtui. Přesun všech idp*.xml z /opt/jetty/webapps/ do zálohy. A start jetty.
Zakomentování hlídacích scriptů v crontab. Úprava scriptů a odkomentování monitorovacího.
'''
systemctl disable stunnel
systemctl stop stunnel
'''
## Konverze konfigurace
