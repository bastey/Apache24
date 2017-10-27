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