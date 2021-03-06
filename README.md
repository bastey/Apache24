# apache-proxy #
Serveur Apache en frontal d'une application sur port 8090 pour tester le Basic Auth et le SSL one way

Installation de Apache sur Windows (httpd-2.4.28-Win64-VC15.zip):

    c:
    git clone https://github.com/bastey/Apache24.git

Installation en tant que service Windows

    cd c:\Apache24\bin
    httpd.exe -k install
    sc start Apache2.4


## TP1 : Configurer le Basic Authication ##

Résultat final :

        git checkout basic_auth

Configurer le Basic Auth :

1. Générer un fichier contenant le login/password

        cd C:\Apache24\bin
        htpasswd -c .\..\conf\http.passwords toto

2. Ajouter dans httpd.conf

        #Activation Basic Auth
        <Location "/">
            AuthType Basic
            AuthName "Restricted area"
            AuthUserFile c:/Apache24/conf/http.passwords
            Require user toto
        </Location>

3. Redémarrer Apache

        sc stop Apache2.4
        sc start Apache2.4

4. Tester requête http://localhost:80/example/v1/hotel avec et sans login:password = toto:titi

HEADER HTTP : Authorization = "Basic login:password" avec login:password en base64  
Sans header Autorization : Erreur HTTP 401  
Avec header Autorization : Erreur HTTP 200

## TP2 : Configuer Le SSL one way ##

Résultat final :

        git checkout basic_auth_and_ssl_one_way

On veut sécuriser un serveur qui expose 1 webservcie REST par du Basic Auth + SSL 1 way  
Il faut  
- un Keystore contenant la clé privée et le certificat (signé par CA ou auto signé)  
- définir un login:pwd que je vais communiquer (secrètement) à mes appelants.  

Installer open SSL sur windows :  
set OPENSSL_CONF=C:\openssl\bin\openssl.cfg  

1. **Générer une clé privée (openSSL)**

        cd C:\certificat  
        C:\openssl\bin\openssl.exe genrsa -des3 -out server.key 2048  
Définir un mot de passe pour la clé privée  
Supprimer le passphrase (facultatif) :

        copy server.key server.key.org  
        C:\openssl\bin\openssl.exe rsa -in server.key.org -out server.key  

2. **Faire le CSR (demande de certificat)**  
pré-requis : connaître les infos du serveur sur lequel sera installé le certificat

        C:\openssl\bin\openssl.exe req -new -key server.key -out server.csr  
        Country name            = FR  
        State               = France  
        Locality Name           = Rennes  
        Organization Name       = XXX  
        Organizational Unit Name        = XXX  
        Common Name         = <FQDN à protéger>  = hostname of server or BigIP to protect  
        Email address           = <adresse mail du demandeur>  
        Challenge password      = <vide>  
        An optional company Name    = <vide>  

3. **Récupérer le certificat signé**  
    a) Pour un certificat signé, on récupère une fichier .cer qui contient la clé public ainsi que les infos de l'autorité de certification  
    Possibilité de convertir en .crt (format PEM) :

        C:\openssl\bin\openssl.exe x509 -inform der -in server.cer -out server.crt  

    b) Générer un certificat auto-signé

        C:\openssl\bin\openssl.exe x509 -req -days 365 -in C:\certificat\server.csr -signkey C:\certificat\server.key -out C:\certificat\server.cer

4. **Ajouter la clé privée et le certificat dans un Keystore** (format PKCS#12)  
Définir un mot de passe pour le keystore

        C:\openssl\bin\openssl.exe pkcs12 -export -out C:\certificat\server_keystore.p12 -inkey C:\certificat\server.key -in C:\certificat\server.cer  

5. **Configuration Côté Serveur**  
Dans httpd.conf, décommenter  

        LoadModule ssl_module modules/mod_ssl.so  
        LoadModule socache_shmcb_module modules/mod_socache_shmcb.so  

        Include conf/extra/httpd-ssl.conf

Copier fichier .cer et .key dans c:/Apache24/conf/certificat/  

Dans extra/httpd-ssl.conf :  
    Modifier

        SSLCertificateFile "c:/Apache24/conf/certificat/server.cer"  
        SSLCertificateKeyFile "c:/Apache24/conf/certificat/server.key"  

    Ajouter

        ProxyPass / http://localhost:8090/  
        ProxyPassReverse / http://localhost:8090/  
        ProxyPreserveHost On

6. **Redémarrer Apache**

        sc stop Apache2.4
        sc start Apache2.4

7. **Ajouter le certificat Côté client**  
Chrome > Paramètres, Certificats : ajouter le keystore p12  
Le certificat se trouve dans Apache24/conf/certificat/server_keystore.p12  
Redémarrer Chrome  
Accéder à l'URL https://localhost depuis CHROME et accepter le certificat non signé  
Puis tester dans POSTMAN l'URL : https://localhost:443/example/v1/hotels avec toto:titi
