# Implementaci�n del servidor de correo para linux - Ubuntu v.20
Este manual se realiza con el fin de apoyar a todo aquel administrador con la intenci�n de crear un 
servidor de correo desde cero con un `sistema operativo tipo ubuntu`, est� guia es la primera parte de
tres y en la misma se mencionar�n toda la configuraci�n y administraci�n de toda la paqueteria necesaria.

## Paso 1: Configuraci�n de ip est�tica para la maquina servidor.
ingresamos al archivo ubicado en `/etc/network/` llamado `interfaces` y lo editamos con el comando `cat`
```bash
 cat /etc/network/interfaces
 ```
acontinuación se adjunta una imagen con los cambios.

![Configuración de interfaces](./imgs/interfaces.png)


## Paso 2: Configuración del servidor DNS

### instalacion de paquetes del servicio DNS BIND9
Para instalar esta paquetería debemos ingresar al sistema como super usuario, para esto utilizaremos el comando `sudo -i`
luego haremos una actualización a toda la paquetería de nuestro sistema y por ultimo instalaremos el servicio dns llamado `BIND9`
```bash
 sudo -i
 apt-get update
 apt-get install BIND9
 ```

 ### Configuración del servicio DNS
 Una vez instalado el servicio nos iremos a checar la carpeta `/etc/bind/` y lo primero será crear 2 copias del archivo db.local, 
 uno para crear la zona directa del dns y otra para la zona inversa, yo por estandar utilizo en la zona directa el nombre 
 db."nombre del dominio que deseo crear" y la zona inversa "db.1.168.192.in-addr.arpa" uso mis indicador de red inversa más "in-addr.arpa"

 luego de crear estos archivo en el directorio vamos a configurar el archivo con el nombre `named.conf.local` del mismo directorio

 ```bash
nano /etc/bind/named.conf.local
 ```

 ![Configuración de dns](./imgs/named.conf.png)

 Creamos la zona directa con nuestro nombre de dominio, que sea tipo master y la ubicación del archivo anteriormente creado para la zona
 directa y otra zona para configurar la inversa,igual de tipo master con la ubicación del archivo creado para la zona inversa.
 salimos y guardamos los cambios.

 #### Configuración de la zona directa
una vez configurado el archivo principal procedemos a configurar el archivo creado para la zona directa
 ```bash
nano /etc/bind/db.midominio.com
 ```
 Acá configuraremos para el servicio traduce una cadena de nombre a una dirección ip especifica donde se encuentra el equipo o servicio

  ![Configuración de dns directa](./imgs/DirectZone.png)

acá configuré el servicio de mail y web para el servidor, para poder acceder por medio de un nombre a esos servicios alojados en el servidor.

  #### Configuración de la zona inversa
seguimos de la misma manera con la configuración de la zona inversa, esta es todo lo contrario a la directa, lo que hará es tomar una dirección
ip y traducirla a una cadena de nombres.
 ```bash
nano /etc/bind/db.1.168.192.in-addr.arpa
 ```
 ![Configuración de dns directa](./imgs/InverZone.png)

una vez realizado estos cuatro pasos, reiniciamos el servicio DNS para que quede funcionando con nuestra configuración
 ```bash
systemctl restart bind9
 ```
## Instalación y obtención de certificados
SS/TLS es para obtener certificados para el correo y no tener problemas al momento de comunicarse con entidades terceras, lo que hace estos protocolos
es cifrar todo el contenido de los correos para transportarlos atraves de internet

### Instalación de certbot
Con esta app obtendremos certificados verificados por SSL.com
```bash
sudo snap install --classic certbot
 ```

### Obtención de certificados
Ejecutamos el siguiente comando para obtener un certificado verificado, solo realimos cambios al **email-address** y al **domain server** 
```bash
sudo certbot certonly --manual --server https://acme.ssl.com/sslcom-dv-rsa --manual-public-ip-logging-ok --config-dir /etc/ssl-com --logs-dir /var/log/ssl-com --agree-tos --no-eff-email --email **EMAIL-ADDRESS** --eab-hmac-key HMAC-KEY --eab-kid ACCOUNT-KEY -d **DOMAIN.NAME**

Estos certificados se guardarán en la dirección `/etc/letsencrypt/live/*nombre del dom*/`

## Instalación y configuración del servidor de correos Postfix
El servicio de postfix es un MTA nos ayudará al enrutamiento y envio de correo de forma sencilla y con pocos pasos.

### Instalación de postfix
Para instalar este servicio debemos ejecutar el siguiente comando
```bash
apt-get install postfix
 ```
 luego de ejecutar este comando saldrá una ventana solicitando seleccionar una opción, nosotros seleccionaremos la primera para configurar todo 
 el servicio desde cero.

### Configuración de postfix
una vez instalado el postfix debemos realizar la configuración al archivo principal ubicado en `etc/postfix/` llamado `main.cf`
acontinuación se mencionará la linea que debe estar descomentada y configurada, en el caso de no tener el comando solo deberá transcribirlo.

		31 : compatibility_level = 2
		56 : command_directory = /usr/sbin
		62 : daemon_directory = /usr/lib/postfix/sbin
		68 : data_directory = /var/lib/postfix
		79 : mail_owner = postfix
		95 : myhostname = *nombre del host*
		103 : mydomain = *nombre del dominio*
		123 : myorigin = $mydomain
		138 : inet_interfaces = all
		186 : mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
		228 : local_recipient_maps = unix:passwd.byname $alias_maps
		241 : unknown_local_recipient_reject_code = 550
		270 : mynetworks_style = subnet
		287 : mynetworks = 127.0.0.0/8, *agregar su red local*
		407 : alias_maps = hash:/etc/aliases
		418 : alias_database = hash:/etc/aliases
		440 : home_mailbox = Maildir/
		577 : smtpd_banner = $myhostname ESMTP
		620 - 622: debugger_command =
         PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin
         ddd $daemon_directory/$process_name $process_id & sleep 5
		650: sendmail_path = /usr/sbin/postfix
		655 : newaliases_path = /usr/bin/newaliases
		660 : mailq_path = /usr/bin/mailq
		666 : setgid_group = postdrop
		684 : inet_protocols = ipv4

		message_size_limit= 20971520
		#Mailbox size limit
		mailbox_size_limit=2147483648
		
		smtpd_sasl_type = dovecot
		smtpd_sasl_path = private/auth
		smtpd_sasl_auth_enable = yes
		smtpd_sasl_security_options = noanonymous
		smtpd_sasl_local_domain = $myhostname
		smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination,permit_sasl_authenticate>

		smtp-use_tls = yes
		smtpd_tls_cert_file = /etc/letsencrypt/live/*nombre del dom*/fullchain.pem
		smtpd_tls_key_file = /etc/letsencrypt/live/*nombre del dom*/privkey.pem
		smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
		stmp_tls_CAfiel = /etc/letsencrypt/live/*nombre del dom*/fullchain.pem
		smtp_tls_security_level = may
		smtp_tls_loglevel = 1


Una vez editado este archivo se guarda y dirigimos al siguiente que será el `master.cf`, de igual manera se mencionará la linea que deberá estar 
descomentada

		12 : smtp      inet  n       -       y       -       -       smtpd
		17 : submission inet n       -       y       -       -       smtpd
		18 : -o syslog_name=postfix/submission
		20 : -o smtpd_sasl_auth_enable=yes
		21 : -o smtpd_tls_auth_only=yes
		29 : smtps     inet  n       -       y       -       -       smtpd
		30 : -o syslog_name=postfix/smtps
		31 : -o smtpd_tls_wrappermode=yes

el resto del archivo queda igual, o realizar cambios, ya con esto nuestro servicio postfix estará configurado.

## Instalación y configuración del servicio Dovecot
Dovecot es una MDA que tiene la función de almacenar y servir correos por los protocolos IMAP Y POP3.

### Instalación del servicio dovecot
En esta sección instalaremos 3 paquetes necesarios, el paquete principal y luego para habilitar el funcionamiento en los protocoloes imap y pop3
```bash
apt-get install dovecot-core dovecot-imapd dovecot-pop3d 
 ```

### Configuración de dovecot
El primer archivo en editar será `10-auth.conf` en la ruta  `/etc/dovecot/conf.d/`
```bash
nano -c /etc/dovecot/conf.d/10-auth.conf 
 ```
		10 : disable_plaintext_auth = no

Luego editamos el archivo `10-ssl.conf` 
```bash
nano -c /etc/dovecot/conf.d/10-ssl.conf 
 ```
		6 : ssl = yes
		12 : ssl_cert = </etc/letsencrypt/live/almond.isw612.xyz/fullchain.pem
		13 : ssl_key = </etc/letsencrypt/live/almond.isw612.xyz/privkey.pem
		34 : ssl_client_ca_dir = /etc/ssl/certs
		53 : ssl_dh = </usr/share/dovecot/dh.pem

el siguiente archivo será `10-mail.conf`

		24 : mail_location = maildir:~/Maildir

con estas configuraciones ya podremos enviar correos y comunicarnos con servicios de terceros sin ningun problemas, ademas de utilizar clientes para correo
electronico desde cualquier lugar.