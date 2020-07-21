

# start new cluster
kind delete cluster --name keycloak
kind create cluster --config configs/kind-cluster-config.yaml --name keycloak

# deploy contour and cert-manager
kubectl apply -f https://raw.githubusercontent.com/projectcontour/contour/v1.4.0/examples/render/contour.yaml
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.yaml


# create certs with cert-manager
kubectl apply -f manifests/certificates.yaml

kubectl get secret keycloakcert -o jsonpath="{..tls\.crt}" | base64 -d | openssl x509 -text -noout
kubectl get secret keycloakcert -o jsonpath="{..keystore\.p12}" | base64 -d | openssl pkcs12 -password pass:secret -nodes


kubectl create configmap openldap-config --dry-run -o yaml --from-file=templates/database.ldif --from-file=templates/users-and-groups.ldif | kubectl apply -f -
kubectl create configmap keycloak-config --dry-run -o yaml --from-file=configs/master-realm.json | kubectl apply -f -


docker build docker/keycloak/ -t localhost/keycloak:latest
kind load docker-image --name keycloak localhost/keycloak:latest

docker build docker/openldap/ -t localhost/openldap:latest
kind load docker-image --name keycloak localhost/openldap:latest


kubectl apply -f manifests


https://keycloak.127-0-0-121.nip.io/auth/



# reference
helm repo add codecentric https://codecentric.github.io/helm-charts
cd helm
helm fetch --untar codecentric/keycloak






kubectl exec -it keycloak-0 -- /opt/jboss/keycloak/bin/jboss-cli.sh --connect


# download and unpack dependencies
mkdir -p downloads
curl -L https://downloads.jboss.org/keycloak/10.0.0/keycloak-10.0.0.tar.gz > downloads/keycloak.tar.gz
curl -L https://repo1.maven.org/maven2/org/postgresql/postgresql/42.2.5/postgresql-42.2.5.jar > downloads/postgres-jdbc.jar

rm -rf docker/keycloak/files
mkdir -p docker/keycloak/files/opt/jboss
tar xf downloads/keycloak.tar.gz -Cdocker/keycloak/files/opt/jboss
mv docker/keycloak/files/opt/jboss/keycloak* docker/keycloak/files/opt/jboss/keycloak

mkdir -p docker/keycloak/files/opt/jboss/keycloak/modules/system/layers/base/org/postgresql/jdbc/main
cp -a downloads/postgres-jdbc.jar docker/keycloak/files/opt/jboss/keycloak/modules/system/layers/base/org/postgresql/jdbc/main
cp docker/keycloak/tools/databases/postgres/module.xml docker/keycloak/files/opt/jboss/keycloak/modules/system/layers/base/org/postgresql/jdbc/main

# build keycloak container
docker build docker/keycloak/ -t localhost/keycloak:latest
kind load docker-image --name keycloak localhost/keycloak:latest



# create distribution tarball from source
mvn -Pdistribution -pl distribution/server-dist -am -Dmaven.test.skip clean install
distribution/server-dist/target/keycloak-*.tar.gz







sudo nsenter --target $(pidof slapd) --net wireshark -f  "port 389" -k
sudo nsenter --target $(pgrep -f org.jboss.as.standalone | sed -n 1p) --net wireshark -k
sudo nsenter --target $(pgrep -f org.jboss.as.standalone | sed -n 2p) --net wireshark -k


kubectl get secret keycloakcert -o jsonpath="{..ca\.crt}" | base64 -d > ca.crt
keytool -importcert -storetype PKCS12 -keystore truststore.p12 -storepass secret -noprompt -alias ca -file ca.crt
keytool -importcert -storetype PKCS12 -keystore truststore-new.p12 -storepass secret -noprompt -alias ca -file ca.crt
kubectl create secret generic cacert --dry-run -o yaml --from-file=truststore-new.p12 | kubectl apply -f -
