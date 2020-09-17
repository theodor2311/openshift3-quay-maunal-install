# Quay Maual Installation with Clair Integration
## No longer working!!


## Create Project
```bash
oc new-project quay-enterprise
```
## Create postgreSQL database
```bash
oc new-app postgresql-ephemeral \
-p POSTGRESQL_USER=quayuser \
-p POSTGRESQL_PASSWORD=quaypass \
-p POSTGRESQL_DATABASE=quay \
-p POSTGRESQL_VERSION=10
```
## Wait for postgreSQL startup and create extension and clair database
```bash
while true;do if oc describe ep | grep -q "NotReadyAddresses:  <none>";then break;else sleep 1;fi;done
oc rsh $( oc get pods | grep postgresql | grep -v deploy | cut -d " " -f1) /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | psql -d quay'
oc rsh $( oc get pods | grep postgresql | grep -v deploy | cut -d " " -f1) /bin/bash -c 'echo "create database clair" | psql'
```
## Create a pull secret to retrieve Quay images from quay.io
```bash
docker login -u=<QUAY.IO_LOGIN> -p=<QUAY.IO_PASSWORD> quay.io
oc create secret generic redhat-quay-pull-secret --from-file=".dockerconfigjson=$HOME/.docker/config.json" --type='kubernetes.io/dockerconfigjson' --namespace=quay-enterprise
```
## Create Quay resources
```bash
oc adm policy add-scc-to-user anyuid system:serviceaccount:quay-enterprise:default
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-config-secret.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-servicetoken-role-k8s1-6.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-servicetoken-role-binding-k8s1-6.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-redis.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-config.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-config-service-clusterip.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-config-route.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-service-clusterip.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-app-route.yaml
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/clair-service-clusterip.yaml
```
## Setup Quay with REST API call
```bash
QUAY_CONFIG_ROUTE=$(oc get route quay-enterprise-config --template='{{ .spec.host }}')
QUAY_ROUTE=$(oc get route quay-enterprise --template='{{ .spec.host }}')
QUAY_CONFIG_USER=quayconfig
QUAY_CONFIG_PASS=secret

curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/registrystatus
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/configapp/initialization
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/validate/database -d '{"config":{"DB_URI":"postgresql://quayuser:quaypass@postgresql/quay"}}'
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X PUT https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config -d '{"config":{"DB_URI":"postgresql://quayuser:quaypass@postgresql/quay"}}'
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -X GET https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/setupdb
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/createsuperuser -d '{"username":"quayadmin","email":"quayadmin@redhat.com","password":"redhat123","repeatPassword":"redhat123"}'
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -X GET https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/keys
```
## Create Security Scanner Key, capture the KID and Private Key
```bash
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/keys -d '{"name":"security_scanner Service Key","service":"security_scanner","expiration":null,"notes":"Created during setup for service `security_scanner`"}' > security_scanner.json
```
## Create the Quay config
```bash
cat <<EOF >quay.json
{"config":{"SECURITY_SCANNER_ENDPOINT":"http://clair-clusterip.quay-enterprise.svc:6060","BUILDLOGS_REDIS":{"host": "quay-enterprise-redis", "port": 6379},"USER_EVENTS_REDIS":{"host": "quay-enterprise-redis","port": 6379},"PREFERRED_URL_SCHEME":"https","SERVER_HOSTNAME":"$(oc get route quay-enterprise --template='{{ .spec.host }}')","EXTERNAL_TLS_TERMINATION":true,"SUPER_USERS": ["quayadmin"],"GITHUB_TRIGGER_CONFIG":{},"DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS": [],"GITHUB_LOGIN_CONFIG": {},"GITLAB_TRIGGER_KIND": {},"SETUP_COMPLETE":true,"FEATURE_SECURITY_SCANNER":true,"DB_URI":"postgresql://quayuser:quaypass@postgresql/quay"},"exists": true}
EOF
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X PUT https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config -d @quay.json
```
## Save the Quay config and verify with the config file
```bash
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config > config.json

curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/validate/redis -d @config.json
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/validate/time-machine -d @config.json
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/validate/access -d @config.json
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/validate/ssl -d @config.json
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/superuser/config/validate/security-scanner -d @config.json
```
...
## Complete the setup
```bash
curl -k --user ${QUAY_CONFIG_USER}:${QUAY_CONFIG_PASS} -H "Content-Type: application/json" -X POST https://${QUAY_CONFIG_ROUTE}/api/v1/kubernetes/config -d '{}'
```
## Deploy Quay
```bash
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/quay-enterprise-app-rc.yaml
```
## Create Clair Config
```bash
cat <<EOF >clair-config-cm.yaml
clair:
  database:
    type: pgsql
    options:
      source: postgresql://quayuser:quaypass@postgresql:5432/clair?sslmode=disable
      cachesize: 16384
  api:
    healthport: 6061
    port: 6062
    timeout: 900s
    paginationkey: "XxoPtCUzrUv4JV5dS+yQ+MdW7yLEJnRMwigVY/bpgtQ="
  updater:
    interval: 6h
    notifier:
      attempts: 3
      renotifyinterval: 1h
      http:
        endpoint: https://$(oc get route quay-enterprise --template='{{ .spec.host }}')/secscan/notify
        proxy: http://localhost:6063
jwtproxy:
  signer_proxy:
    enabled: true
    listen_addr: :6063
    ca_key_file: /certificates/mitm.key # Generated internally, do not change.
    ca_crt_file: /certificates/mitm.crt # Generated internally, do not change.
    signer:
      issuer: security_scanner
      expiration_time: 5m
      max_skew: 1m
      nonce_length: 32
      private_key:
        type: preshared
        options:
          key_id: $(cat security_scanner.json | jq .kid | cut -d '"' -f2)
          private_key_path: /clair/config/security_scanner.pem
  verifier_proxies:
  - enabled: true
    listen_addr: :6060
    verifier:
      audience: http://clair-clusterip.quay-enterprise.svc:6060
      upstream: http://localhost:6062
      key_server:
        type: keyregistry
        options:
          registry: https://$(oc get route quay-enterprise --template='{{ .spec.host }}')/keys/
EOF
oc create secret generic clair-config-secret --from-file=config.yaml=clair-config-cm.yaml
```
## Create security_scanner.pem secret
```bash
printf -- "$(cat security_scanner.json | jq .private_key | cut -d '"' -f2)" > security_scanner.pem
oc create secret generic security-scanner-key-secret --from-file=security_scanner.pem=security_scanner.pem
```
## Create Clair trust CA
```bash
echo quit | openssl s_client -connect $(oc get route quay-enterprise |grep quay |awk '{print $2}'):443 -showcerts | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > clair-trust-ca.crt
oc create secret generic clair-trust-ca-secret --from-file=ca.crt=clair-trust-ca.crt
```
## Deploy Clair
```bash
oc create -f https://raw.githubusercontent.com/theodor2311/quay_installation/3.0.4/clair-rc.yaml
```
## Testing (This setup will use quayadmin:redhat123 for the credential)
```bash
skopeo copy --dest-tls-verify=false --dest-creds=quayadmin:redhat123 docker://<TESTING_IMAGE_SOURCE> docker://$(oc get route quay-enterprise --template='{{ .spec.host }}')/quayadmin/<TESTING_IMAGE_DEST>
```
## Cleanup
```bash
oc delete project quay-enterprise
```
