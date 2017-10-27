# apache-proxy
Serveur Apache en frontal d'une application sur port 8090 pour tester le Basic Auth et le SSL one way

Installation de Apache sur Windows (httpd-2.4.28-Win64-VC15.zip):

c:
git clone https://github.com/bastey/Apache24.git


Installation en tant que service Windows

cd c:\Apache24
httpd.exe -k install
sc start Apache2.4


TP1 : Configurer le Basic Authication
git checkout basic_auth

Configurer le Basic Auth :
1) Générer un fichier contenant le login/password
	cd C:\Apache24\bin
	htpasswd -c .\..\conf\http.passwords toto
	
2) Ajouter dans httpd.conf
	#Activation Basic Auth
	<Location "/"> 
	    AuthType Basic
	    AuthName "Restricted area"
	    AuthUserFile c:/Apache24/conf/http.passwords
	    Require user toto
	</Location>

3) Redémarrer Apache
sc stop Apache2.4
sc start Apache2.4

4) Tester requete http://localhost:80/example/v1/hotels
HEADER HTTP : Authorization = "Basic login:password" avec login:password en base64
Sans header Autorization : Erreur HTTP 401
Avec header Autorization : Erreur HTTP 200

TP2 : Configuer Le SSL one way
===============================
git checkout basic_auth_and_ssl_one_way


Je suis un serveur qui expose 1 WS REST que je veux sécuriser par du Basic Auth + SSL 1 way
Il me faut donc un Keystore contenant la clé privée et le certificat (signé par CA ou auto signé)
Il me faut aussi définir un login:pwd que je vais communiquer (secrètement) à mes appelants.

Installer open SSL
sur windows :
set OPENSSL_CONF=C:\openssl\bin\openssl.cfg

1) Générer une clé privée (openSSL)
------------------------------------
Définir un mot de passe pour la clé privée
cd C:\certificat
C:\openssl\bin\openssl.exe genrsa -des3 -out prod_1.key 2048
passphrase facultatif
	Supprimer le mot de passphrase
	copy prod_1.key prod_1.key.org
	C:\openssl\bin\openssl.exe rsa -in prod_1.key.org -out prod_1.key


2) Faire le CSR (demande de certificat)
---------------------------------------
pré-requis : connaître les infos du serveur sur lequel sera installé le certificat
C:\openssl\bin\openssl.exe req -new -key prod_1.key -out prod_1.csr
Country name 			= FR
State 				= France
Locality Name 			= Rennes
Organization Name 		= XXX
Organizational Unit Name		= XXX
Common Name			= <FQDN à protéger>  = hostname of server or BigIP to protect
Email address			= <adresse mail du demandeur>
Challenge password		= <vide>
An optional company Name 	= <vide>


3) Récupérer le certificat signé
---------------------------------
	3a) Pour un certificat signé, on récupère une fichier .cer qui contient la clé public ainsi que les infos de l'autorité de certification

	possibilité de convertir en .crt (format PEM)
	C:\openssl\bin\openssl.exe x509 -inform der -in prod_1.cer -out prod_1.crt


	3b) Générer un certificat auto-signé
	C:\openssl\bin\openssl.exe x509 -req -days 365 -in C:\certificat\prod_1.csr -signkey C:\certificat\prod_1.key -out C:\certificat\prod_1.cer


4) Ajouter la clé privée et le certificat dans un Keystore (format PKCS#12)
---------------------------------------------------------------------------
Définir un mot de passe pour le keystore
C:\openssl\bin\openssl.exe pkcs12 -export -out C:\certificat\prod_1_keystore.p12 -inkey C:\certificat\prod_1.key -in C:\certificat\prod_1.cer


5) Configuration Côté Serveur :
--------------------------------
httpd.conf
	LoadModule ssl_module modules/mod_ssl.so
	LoadModule socache_shmcb_module modules/mod_socache_shmcb.so

	Décommenter
		Include conf/extra/httpd-ssl.conf

	Copier fichier prod_1.cer et prod1.key dans c:/Apache24/conf/certificat/

extra/httpd-ssl.conf
	Modifier
		SSLCertificateFile "c:/Apache24/conf/certificat/prod_1.cer"
		SSLCertificateKeyFile "c:/Apache24/conf/certificat/prod_1.key"
	Ajouter
		ProxyPass / http://localhost:8090/
		ProxyPassReverse / http://localhost:8090/
		ProxyPreserveHost On

6) Redémarrer Apache
sc stop Apache2.4
sc start Apache2.4


7) Ajouter le certificat Côté client :
------------------------------
	Chrome > Paramètres, Certificats : ajouter le keystore p12