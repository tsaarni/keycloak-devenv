
# Environment to develop, test and experiment with Keycloak and LDAP federation


![](diagram.png)


## Preparation

Create certificates and keys for LDAP server and clients

```bash
mkdir -p certs
cfssl genkey -initca config/cfssl-csr-ca.json | cfssljson -bare certs/ca
cfssl gencert -ca certs/ca.pem -ca-key certs/ca-key.pem config/cfssl-csr-server.json | cfssljson -bare certs/server
cfssl gencert -ca certs/ca.pem -ca-key certs/ca-key.pem config/cfssl-csr-user.json | cfssljson -bare certs/user
cfssl gencert -ca certs/ca.pem -ca-key certs/ca-key.pem config/cfssl-csr-ldap-admin.json | cfssljson -bare certs/ldap-admin
cfssl gencert -ca certs/ca.pem -ca-key certs/ca-key.pem config/cfssl-csr-ldap-client.json | cfssljson -bare certs/ldap-client
```

Create truststore and keystore for Keycloak

```bash
keytool -importcert -storetype PKCS12 -keystore truststore.p12 -storepass password -noprompt -alias ca -file certs/ca.pem
openssl pkcs12 -export -passout pass:password -noiter -nomaciter -in certs/ldap-admin.pem -inkey certs/ldap-admin-key.pem -out keystore.p12
```



## Build and run test services

Run LDAP server (OpenLDAP) and LDAP client (SSH and SSSD)

```bash
docker-compose up
```


Test that LDAP is up and running

```bash
# dump configuration
docker exec ldap-sasl-external_openldap_1 ldapsearch -H ldapi:/// -Y EXTERNAL -b cn=config

# dump users and groups
docker exec ldap-sasl-external_openldap_1 slapcat -F /data/config

# dump user and groups by using `ldap-admin`
ldapsearch -D cn=ldap-admin,ou=users,o=example -w ldap-admin -b ou=users,o=example

# test bind (by changing password)
ldappasswd -ZZ -D cn=user,ou=users,o=example -w user -s user
```

The client configuration is read from `ldaprc` in home directory by default.
For parameters seem https://www.openldap.org/software/man.cgi?query=ldap.conf



## Running Keycloak

Clone keycloak repository, build and install to local maven repo

```bash
mvn install -DskipTests
```

After editing part of the code, build and install only single module in
multi-module maven project (to save time)

```bash
mvn install -DskipTests -pl federation/ldap/
```

Run Keycloak with embedded undertow server

```bash
export WORKDIR=/home/tsaarni/work/keycloak-devenv/ldap-sasl-external

mvn -f testsuite/utils/pom.xml exec:java -Pkeycloak-server -Dkeycloak.migration.action=import -Dkeycloak.migration.provider=dir -Dkeycloak.migration.dir=$WORKDIR/keycloak/ -Djavax.net.ssl.trustStore=$WORKDIR/truststore.p12 -Djavax.net.ssl.trustStorePassword=password -Djavax.net.ssl.javax.net.ssl.trustStoreType="PKCS12" -Djavax.net.ssl.keyStore=$WORKDIR/keystore.p12 -Djavax.net.ssl.keyStorePassword=password -Djavax.net.ssl.javax.net.ssl.keyStoreType="PKCS12"
```

To login to keycloak using command linem use `kcinit` from keycloak

```bash
./testsuite/integration-arquillian/tests/base/target/kcinit login --config $WORKDIR/
```


When starting keycloak above, the realm configuration is imported from
$WORKDIR/keycloak
(see [here](https://github.com/keycloak/keycloak-documentation/blob/master/server_admin/topics/export-import.adoc))

To re-export realm after doing configuration changes:

1. Choose Export
    - Export groups and roles: on
    - Export clients: on
2. In `realm-export.json` find `bindCredential` and fill in LDAP bind password


## Capturing LDAP traffic

To debug LDAP traffic, first get the interface name from LDAP server container
and then run wireshark.

```bash
IFNAME=$(ip -j link show | jq ".[] | select(.ifindex==$(docker exec ldap-sasl-external_openldap_1 cat /sys/class/net/eth0/iflink)) | .ifname")

wireshark -i $IFNAME -f "port 389 or port 636" -k -o tls.keylog_file:output/wireshark-keys.log
```

The LDAP server container uses openssl wrapper ([see here](docker/openldap/sslkeylog/))
that dumps TLS pre-master secrets to `output/wireshark-keys.log`.
This enables debugging LDAP over TLS.



## Using LDAP client

Run follownig to login to LDAP client container using LDAP user account

```bash
sshpass -p user ssh user@localhost -p 2222 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no "echo Hello world!"
```
