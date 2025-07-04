### Création d'une image docker basée sur l'image de nginx en y ajoutant le fichier "index.html"
```bash
docker docker build . -t <DOCKERUSERNAME>/hello-risf:0.0.1
docker push <DOKCERUSERNAME>/hello-risf:0.0.1
```

### Création du premier microservice avec le fichier manifest "hello-risf.yaml"
```bash
kubectl apply -f hello-risf.yaml
```

### Création du Persistent Volume(PV) et écriture du .html dans le PV avec les fichiers "html-pv.yaml" et "html-writing-pv.yaml"
```bash
kubectl apply -f html-pv.yaml -f html-writing-pv.yaml
```
### Création du deuxième microservice avec le fichier "hello-itsf.yaml"
```bash
kubectl apply -f hello-itsf.yaml
```

### Création de la PKI pour certifier les clés CA
```bash
mkdir -p pki/{ca,certs,private,csr}
cd pki

openssl genrsa -out ca/ca-key.pem 4096
```
# Création du certificat CA
'ca/ca.conf'
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
C = FR
ST = PACA
L = Nice
O = test
OU = IT
CN = Louis test

[v3_ca]
basicConstraints = critical,CA:TRUE
keyUsage = critical,digitalSignature,keyCertSign,cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always

```bash
openssl req -new -x509 -days 3650 -key ca/ca-key.pem -out ca/ca-cert.pem -config ca/ca.conf
```

# Création du certificat des deux domaines
'certs/domains.conf'
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = FR
ST = PACA
L = Nice
O = test
OU = IT
CN = hello-risf.local.domain

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation,digitalSignature,keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = hello-risf.local.domain
DNS.2 = hello-itsf.local.domain
DNS.3 = *.hello-risf.local.domain
DNS.4 = *.hello-itsf.local.domain

# Génération des clés privées pour les domaines
```bash
openssl genrsa -out private/domains-key.pem 2048
```

# Création de la certificate signing request (CSR)
```bash
openssl req -new -key private/domains-key.pem -out csr/domains.csr -config certs/domains.conf
```

# Création de l'extension du fichier de certificat 
'certs/domains-ext.conf'
[v3_req]
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = hello-risf.local.domain
DNS.2 = hello-itsf.local.domain
DNS.3 = *.hello-risf.local.domain
DNS.4 = *.hello-itsf.local.domain

# Signature du certificat avec CA
```bash
openssl x509 -req -in csr/domains.csr -CA ca/ca-cert.pem -CAkey ca/ca-key.pem -CAcreateserial -out certs/domains-cert.pem -days 365 -extensions v3_req -extfile certs/domains-ext.conf
```

# Vérification et affichage des informations du certificat
```bash
openssl verify -CAfile ca/ca-cert.pem certs/domains-cert.pem
```

### Création du composant secret avec les certifications signées
```bash
kubectl create secret tls multi-domain-tls   --cert=certs/domains-cert.pem   --key=private/domains-key.pem
```
### Activation de l'addon ingress
```bash
minikube addons enable ingress
```
### Création de l'ingress avec le fichier "ingress.yaml"
```bash
kubectl apply -f ingress.yaml
```
### Obtention de l'adresse IP du Node
```bash
minikube ip = <<VOTRENODEIP>>
```
### Mapping de l'adresse du Node avec les domaines

Ouvrir Notepad en tant qu'administrateur avec les touches "Win+R" puis taper notepad et utiliser "Ctrl+Shift+Enter", maintenant que notepad est lancé, appuyer sur la touche "Open" et entrer le path "C:\Windows\System32\drivers\etc" puis sélectionner à la place de "Text documents (*.txt)" --> "All files(*.*)" ensuite sélectionner le fichier "hosts"

Une fois dans l'éditeur du fichier "hosts" ajoutez les lignes puis sauvegarder: 

<<VOTRENODEIP>>    hello-risf.local.domain
<<VOTRENODEIP>>    hello-itsf.local.domain

Vous pouvez désormais accéder au contenu des deux microservices déployés en utilisant les domaines dans votre navigateur web.

### Commandes Utiles

À utiliser lors d'un restart du cluster :
```bash
kubectl delete job html-setup
kubectl apply -f html-writing-pv.yaml
kubectl logs job/html-setup -c copy-html
kubectl rollout restart deployment hello-itsf
```

À utiliser pour vérifier les UID des déploiements :
```bash
kubectl exec deploy/hello-itsf -- id
kubectl exec deploy/hello-risf -- id
```
