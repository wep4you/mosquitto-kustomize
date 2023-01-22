# Mosquitto MQTT

Lightweight MQQT Broker

## Sources

https://mosquitto.org
https://github.com/eclipse/mosquitto

## Sample Security and Authentication
### Generate Certificates with Self Signed Root CA and Client Auth

Runs local so be sure that cfssl is installed. For Mac use brew

    brew install cfssl

Run the playbook, the certificates will be generated to (./certificates/certs).

Change the Settings to your needs, in this example also 3 Test Client Zertificate
will be generated, add more for your needed devices, the name will used as name in MQTT

    ansible-playbook ./certificates/base-certificates.yml


### Setup Sealed Secrets

You have to have Selead Secret operator running for Sealed Secrets to work!

Generate all needed Secrets locally and seal them, so that you are able to store it in GIT. Just store the yaml Files with the sealed Secrets,
do not save your Real Passwords or the Secrets from Kubernetes, they are just Base64 Encoded, its easy to decode and therefore not safe!
For sealing you have to install Sealed Secrets from Bitnami. Find a tutorial with Kustomize and Argo CD in my repo [Sealed Secrets Kustomize](https://github.com/wep4you/sealed-secrets-kustomize) 

A sample password file is stored in  (overlays/sample/password_file.txt). Change the settings there and use strong passwors. Also make shure,
to not check in this file, this should only be present locally!

Now generate a Sealed Secret for your file, this new generated file could be checked in.

    kubectl --namespace mqtt create secret generic mosquitto-password --dry-run=client \
    --from-file password_file.txt=./overlays/wep4you/password_file.txt \
    --from-file ca.crt=./certificates/certs/ca.pem \
    --from-file server.key=./certificates/certs/server-key.pem \
    --from-file server.crt=./certificates/certs/server.pem \
    --output json | kubeseal | tee overlays/wep4you/sealedsecret.yml    


Test your settings with Client Certificate

    mosquitto_pub \
    --cafile ./certificates/certs/ca.pem \
    --cert ./certificates/certs/client-1.pem \
    --key ./certificates/certs/client-1-key.pem \
    -d -h 172.16.207.215 -p 1883 -t test -m "hello world"



## Install with Argo CD

Use a central repository which links to all Apps which has to be deployed,
and add the App desciption there, or deploy the App for manually.
Find a sample in my [App of Apps Repo](https://github.com/wep4you/k8s-apps.git),

## Install manual with kubectl

To add your own configuration, copy ```overlays/sample``` to a new folder and change the files to your needs.
It's standard Kustomize format you can use.

    ```
    kubectl apply -k overlays/sample
    ```

## Certs manual

### CA

First create a key for the CA

´´´
openssl genrsa -des3 -out ./certificates/certs/ca.key 2048
´´´

Create a certificate for the CA using the CA key that we created in step 1
´´´
openssl req -new -x509 -days 1826 -key ./certificates/certs/ca.key -out ./certificates/certs/ca.crt
´´´

### Server Certs

Now we create a server key pair that will be used by the broker

´´´
openssl genrsa -out ./certificates/certs/server.key 2048
´´´

Now we create a certificate request .csr. When filling out the form the common name is important and is usually the domain name of the server.

´´´
openssl req -new -out ./certificates/certs/server.csr -key ./certificates/certs/server.key
´´´

Now we use the CA key to verify and sign the server certificate. This creates the server.crt file

´´´
openssl x509 -req -in ./certificates/certs/server.csr -CA ./certificates/certs/ca.crt -CAkey ./certificates/certs/ca.key -CAcreateserial -out ./certificates/certs/server.crt -days 360
´´´

### Client Certs

The first step is to create a client private key.

´´´
openssl genrsa -out ./certificates/certs/client.key 2048
´´´

Next create a certificate request and use the client private key to sign it.

´´´
openssl req -new -out ./certificates/certs/client.csr -key ./certificates/certs/client.key
´´´

Now we complete the request and create a client certificate. The command is:

´´´
openssl x509 -req -in ./certificates/certs/client.csr -CA ./certificates/certs/ca.crt -CAkey ./certificates/certs/ca.key -CAcreateserial -out ./certificates/certs/client.crt -days 360
´´´

### Add the Certs into a sealed-secret

´´´
    kubectl --namespace mqtt create secret generic mosquitto-password --dry-run=client \
    --from-file password_file.txt=./overlays/wep4you/password_file.txt \
    --from-file ca.crt=./certificates/certs/ca.crt \
    --from-file server.key=./certificates/certs/server.key \
    --from-file server.crt=./certificates/certs/server.crt \
    --output json | kubeseal | tee overlays/wep4you/sealedsecret.yml    
´´´

### Test it

´´´
    mosquitto_pub \
    --cafile ./certificates/certs/ca.crt \
    --cert ./certificates/certs/client.crt \
    --key ./certificates/certs/client.key \
    -d -h 172.16.207.215 -p 1883 -t test -m "hello world"
´´´