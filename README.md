# Hashicorp Vault / Cert Manager workshop

## Credits

Loosely based on: https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-cert-manager

## DO

- Use Vault as PKI infrastructure
- Have cert manager in place to order certificates from vault
- Secure our ingress for wordpress with certificate issued by vault, and managed by cert-manager

## DON'T 

**DO NOT** use this in production, unless you're looking for a new assignment very VERY soon. ;-)

But serious: This workshop shows the concepts, but is (very) **INSECURE** by design. Don't use outside a lab environment.

## Prerequisites 

- Minikube
- Docker
- Helm
- jq

## Vault

### Install vault through helm

```
kubectl create namespace vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm -n vault install vault hashicorp/vault --set "injector.enabled=false"
```

Check for running pods:

```
kubectl -n vault get pods

NAME      READY   STATUS    RESTARTS   AGE
vault-0   0/1     Running   0          33s
```

Running, but no pods ready? CORRECT! 

Our vault is currently in both the uninitialized as well as the sealed state. An unitialized sealed vault has no access to the vault storage backend, rendering it quite useless for now. Use the following command to initialize the vault with 1 unseal key:

```
kubectl -n vault exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > init-keys.json
``` 

Now, vault is initialized, but still sealed.

```
When a Vault server is started, it starts in a sealed state. In this state, Vault is configured to know where 
and how to access the physical storage, but doesn't know how to decrypt any of it.

Unsealing is the process of obtaining the plaintext root key necessary to read the decryption key to decrypt the data, 
allowing access to the Vault.

Prior to unsealing, almost no operations are possible with Vault. For example authentication, managing the mount tables, 
etc. are all not possible. The only possible operations are to unseal the Vault and check the status of the seal.
```

Next, unseal our vault so changes are possible:

```
VAULT_UNSEAL_KEY=$(cat init-keys.json | jq -r ".unseal_keys_b64[]")
kubectl -n vault exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.13.1
Build Date      2023-03-23T12:51:35Z
Storage Type    file
Cluster Name    vault-cluster-e89b269d
Cluster ID      47df1b2e-966b-4cda-9ab6-69a6fc0e294f
HA Enabled      false
```

We should see a running and ready pod now:

```
kubectl -n vault get pods
NAME      READY   STATUS    RESTARTS   AGE
vault-0   1/1     Running   0          14m
```

Try to login to vault using the root token:

```
VAULT_ROOT_TOKEN=$(cat init-keys.json | jq -r ".root_token")
kubectl -n vault exec vault-0 -- vault login $VAULT_ROOT_TOKEN

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.YdrVhsc7r4qbX6MR7PDT337i
token_accessor       ftIlswViOSIBc2h2O2WuEGOU
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

We want Vault to act as an PKI. Therefor, we should enable the secrets PKI engine:

Start an interactive shell on the vault-0 pod:

```
kubectl -n vault exec --stdin=true --tty=true vault-0 -- /bin/sh
```

Next, enable the PKI engine:

```
vault secrets enable pki

Success! Enabled the pki secrets engine at: pki/
```

Defauklt TTL is 30 days, tune the max TTL of the pki engine, mainly for our Root CA certificate:

```
vault secrets tune -max-lease-ttl=8760h pki

Success! Tuned the secrets engine at: pki/
```

Generate a self-signed Root CA certificate, valid for 8670 hours (1 year):

```
vault write pki/root/generate/internal \
   common_name=sue-workshop.nl \
   ttl=8760h

WARNING! The following warnings were returned from Vault:

  * This mount hasn't configured any authority information access (AIA)
  fields; this may make it harder for systems to find missing certificates
  in the chain or to validate revocation status of certificates. Consider
  updating /config/urls or the newly generated issuer with this information.

Key              Value
---              -----
certificate      -----BEGIN CERTIFICATE-----
MIIDQTCCAimgAwIBAgIUBe7Vw5wF/PG2nTFvfknG9feHf9owDQYJKoZIhvcNAQEL
BQAwGjEYMBYGA1UEAxMPc3VlLXdvcmtzaG9wLm5sMB4XDTIzMDUxNzE4MDczMVoX
DTI0MDUxNjE4MDgwMVowGjEYMBYGA1UEAxMPc3VlLXdvcmtzaG9wLm5sMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5h4bYFZ3T5jzNky546nCh3o+Mmi3
FRi3jUSkMxeRzhc5ZJxCEV3I1ze0oQST6X/xQDA/7dn20UbzuJJegI6TmU+hE2Zk
l0SHCKYvgskY9i3se/+utcktJFmPWIntPv0zleDOMSYrPE70enywJGlWkMsa93b0
zR5l3YWQc4cbhpnHyJoTWRJWVPmM4Xx5Y+dZ2kELEZ220E/dqhng6XEZdUg490L0
o00KuCe8pAsDScnjGKyvC2PjnI2crxs2fnQGvOS0o0XDBHjCgpYcanTAqc0V0h/B
Dst2yv2ZInFKC84roAzzZDcqkcITNIqMoDOqlZLQ+mwmAuuKGu5KZQBnDQIDAQAB
o38wfTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQU
bjHRhZscgkxmdINVCyr3sAur+bYwHwYDVR0jBBgwFoAUbjHRhZscgkxmdINVCyr3
sAur+bYwGgYDVR0RBBMwEYIPc3VlLXdvcmtzaG9wLm5sMA0GCSqGSIb3DQEBCwUA
A4IBAQBi20cDn23zPYXHmJaDS7Ohi5wOXbWztLqCuSwPt3Ef0la9NJj27O8VbDvl
rAnZCKam5aaMeMOw6rFC6XUegy4SklIgmOlDpY94LAn2/wA8cw6f+JH8PZNylaD2
ia/mbMOxUvfF+mUuurbK3lAq+/moHhFYORT8a/FCfVzTya407mZOE7DrX61G7R0i
ahoA16K6Nus6vZFhx1m7hVGRqVPjgh5E+M8JjIYzJm2yMyEFrt6Okn0anm/RmYpa
iffXuzDW6INInXKrS0fWGVEQUNyZdNf6q7k7Gech329toNRbYsZJx/XLlvmS5tib
jHARUU5wJ0tRFm8KfngtkUe8F9YV
-----END CERTIFICATE-----
expiration       1715882881
issuer_id        97d2a6b6-701b-9368-c216-668140c7fbc3
issuer_name      n/a
issuing_ca       -----BEGIN CERTIFICATE-----
MIIDQTCCAimgAwIBAgIUBe7Vw5wF/PG2nTFvfknG9feHf9owDQYJKoZIhvcNAQEL
BQAwGjEYMBYGA1UEAxMPc3VlLXdvcmtzaG9wLm5sMB4XDTIzMDUxNzE4MDczMVoX
DTI0MDUxNjE4MDgwMVowGjEYMBYGA1UEAxMPc3VlLXdvcmtzaG9wLm5sMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5h4bYFZ3T5jzNky546nCh3o+Mmi3
FRi3jUSkMxeRzhc5ZJxCEV3I1ze0oQST6X/xQDA/7dn20UbzuJJegI6TmU+hE2Zk
l0SHCKYvgskY9i3se/+utcktJFmPWIntPv0zleDOMSYrPE70enywJGlWkMsa93b0
zR5l3YWQc4cbhpnHyJoTWRJWVPmM4Xx5Y+dZ2kELEZ220E/dqhng6XEZdUg490L0
o00KuCe8pAsDScnjGKyvC2PjnI2crxs2fnQGvOS0o0XDBHjCgpYcanTAqc0V0h/B
Dst2yv2ZInFKC84roAzzZDcqkcITNIqMoDOqlZLQ+mwmAuuKGu5KZQBnDQIDAQAB
o38wfTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQU
bjHRhZscgkxmdINVCyr3sAur+bYwHwYDVR0jBBgwFoAUbjHRhZscgkxmdINVCyr3
sAur+bYwGgYDVR0RBBMwEYIPc3VlLXdvcmtzaG9wLm5sMA0GCSqGSIb3DQEBCwUA
A4IBAQBi20cDn23zPYXHmJaDS7Ohi5wOXbWztLqCuSwPt3Ef0la9NJj27O8VbDvl
rAnZCKam5aaMeMOw6rFC6XUegy4SklIgmOlDpY94LAn2/wA8cw6f+JH8PZNylaD2
ia/mbMOxUvfF+mUuurbK3lAq+/moHhFYORT8a/FCfVzTya407mZOE7DrX61G7R0i
ahoA16K6Nus6vZFhx1m7hVGRqVPjgh5E+M8JjIYzJm2yMyEFrt6Okn0anm/RmYpa
iffXuzDW6INInXKrS0fWGVEQUNyZdNf6q7k7Gech329toNRbYsZJx/XLlvmS5tib
jHARUU5wJ0tRFm8KfngtkUe8F9YV
-----END CERTIFICATE-----
key_id           1b03ef3c-784e-358b-f7b8-75092689a169
key_name         n/a
serial_number    05:ee:d5:c3:9c:05:fc:f1:b6:9d:31:6f:7e:49:c6:f5:f7:87:7f:da
```

Configure Issuer / CRL endpoints for vault service in vault namespace:

```
vault write pki/config/urls \
  issuing_certificates="http://vault.vault:8200/v1/pki/ca" \
  crl_distribution_points="http://vault.vault:8200/v1/pki/crl"

Key                        Value
---                        -----
crl_distribution_points    [http://vault.vault:8200/v1/pki/crl]
enable_templating          false
issuing_certificates       [http://vault.vault:8200/v1/pki/ca]
ocsp_servers               []
```

Configure a role named sue-workshop-nl that enables the creation of certificates for the sue-workshop.nl domain with any subdomains:

```
vault write pki/roles/sue-workshop-nl \
    allowed_domains=sue-workshop.nl \
    allow_subdomains=true \
    max_ttl=72h

Key                                   Value
---                                   -----
allow_any_name                        false
allow_bare_domains                    false
allow_glob_domains                    false
allow_ip_sans                         true
allow_localhost                       true
allow_subdomains                      true
allow_token_displayname               false
allow_wildcard_certificates           true
allowed_domains                       [sue-workshop.nl]
allowed_domains_template              false
allowed_other_sans                    []
allowed_serial_numbers                []
allowed_uri_sans                      []
allowed_uri_sans_template             false
allowed_user_ids                      []
basic_constraints_valid_for_non_ca    false
client_flag                           true
cn_validations                        [email hostname]
code_signing_flag                     false
country                               []
email_protection_flag                 false
enforce_hostnames                     true
ext_key_usage                         []
ext_key_usage_oids                    []
generate_lease                        false
issuer_ref                            default
key_bits                              2048
key_type                              rsa
key_usage                             [DigitalSignature KeyAgreement KeyEncipherment]
locality                              []
max_ttl                               72h
no_store                              false
not_after                             n/a
not_before_duration                   30s
organization                          []
ou                                    []
policy_identifiers                    []
postal_code                           []
province                              []
require_cn                            true
server_flag                           true
signature_bits                        256
street_address                        []
ttl                                   0s
use_csr_common_name                   true
use_csr_sans                          true
use_pss                               false
```

The role, sue-workshop-nl, is a logical name that maps to a policy used to generate credentials. This generates a number of endpoints that are used by the Kubernetes service account to issue and sign these certificates. A policy must be created that enables these paths.

Create a policy named pki that enables read access to the PKI secrets engine paths:

```
vault policy write pki - <<EOF
path "pki*"                        { capabilities = ["read", "list"] }
path "pki/sign/sue-workshop-nl"    { capabilities = ["create", "update"] }
path "pki/issue/sue-workshop-nl"   { capabilities = ["create"] }
EOF

Success! Uploaded policy: pki
```

Enable the kubernetes authentication method in vault:

```
vault auth enable kubernetes

Success! Enabled kubernetes auth method at: kubernetes/
```

Configure the Kubernetes authentication method to use location of the Kubernetes API:

```
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

Success! Data written to: auth/kubernetes/config
```

Finally, create a Kubernetes authentication role named sue-workshop-issuer that binds the pki policy with a Kubernetes service account named sue-workshop-issuer:

```
vault write auth/kubernetes/role/sue-workshop-issuer \
    bound_service_account_names=sue-workshop-issuer \
    bound_service_account_namespaces=wordpress \
    policies=pki \
    ttl=20m

Success! Data written to: auth/kubernetes/role/sue-workshop-issuer
```

IMPORTANT:

The bound_service_account_names and bound_service_account_namespaces option allows us to specify which serviceaccounts from which namespaces are allowed to order certificates.

Exit the vault-0 pod:

```
exit
```

## Cert Manager

Create the cert-manager namespace:

```
kubectl create namespace cert-manager
```

Add the jetstack helm repository:

```
helm repo add jetstack https://charts.jetstack.io

"jetstack" has been added to your repositories
```

Update the helm repositories:

```
helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Install cert-manager in the cert-manager namespace (this may take a while):

```
helm install cert-manager \
    --namespace cert-manager \
    --version v1.11.2 \
    --set installCRDs=true \
   jetstack/cert-manager

NAME: cert-manager
LAST DEPLOYED: Wed May 17 20:48:28 2023
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 4
TEST SUITE: None
NOTES:
cert-manager v1.11.2 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```

Verify all pods are running:

```
kubectl -n cert-manager get pods

NAME                                       READY   STATUS      RESTARTS   AGE
cert-manager-8598b54d89-hvvx5              1/1     Running     0          7m54s
cert-manager-cainjector-66885bc449-fd9dr   1/1     Running     0          7m54s
cert-manager-startupapicheck-8g7ng         0/1     Completed   3          7m53s
cert-manager-webhook-884c4477c-2vzd2       1/1     Running     0          7m54s
```

Next, we create the necesary items to order a certificate from a separate namespace, wordpress, outside the vault namespace:

Create the wordpress namespace:

```
kubectl create namespace wordpress
```

Create service account sue-workshop-issuer (we created the pki role within vault for this serviceaccount):

```
kubectl -n wordpress create serviceaccount sue-workshop-issuer

serviceaccount/sue-workshop-issuer created
```

Next, we create the secret containing the token used to authenticate with the service account:

```
cat > issuer-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: issuer-token-lmzpj
  namespace: wordpress
  annotations:
    kubernetes.io/service-account.name: sue-workshop-issuer
type: kubernetes.io/service-account-token
EOF
```

```
kubectl -n wordpress apply -f issuer-secret.yaml

secret/issuer-token-lmzpj created
```

Get all the secrets in the wordpress namespace:

```
kubectl -n wordpress get secrets

NAME                          TYPE                                  DATA   AGE
issuer-token-lmzpj            kubernetes.io/service-account-token   3      10m
```

Define an Issuer, named vault-issuer, that uses the vault service URL as a certificate issuer.

```
cat > vault-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: vault-issuer
  namespace: wordpress
spec:
  vault:
    server: http://vault.vault:8200
    path: pki/sign/sue-workshop-nl
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: sue-workshop-issuer
        secretRef:
          name: issuer-token-lmzpj
          key: token
EOF
```

```
kubectl -n wordpress apply -f vault-issuer.yaml

issuer.cert-manager.io/vault-issuer created
```

**About Issuers**

```
An Issuer is a namespaced resource, and it is not possible to issue certificates from an Issuer 
in a different namespace. This means you will need to create an Issuer in each namespace you wish 
to obtain Certificates in.

If you want to create a single Issuer that can be consumed in multiple namespaces, you should consider 
creating a ClusterIssuer resource. This is almost identical to the Issuer resource, however is non-namespaced 
so it can be used to issue Certificates across all namespaces.
```

Define a certificate named wordpress-sue-workshop-nl:

```
cat > wordpress-sue-workshop-nl.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wordpress-sue-workshop-nl
  namespace: wordpress
spec:
  secretName: wordpress-sue-workshop-tls
  issuerRef:
    name: vault-issuer
  commonName: wordpress.sue-workshop.nl
  dnsNames:
  - wordpress.sue-workshop.nl
EOF
```

Request the certificate through vault-issuer:

```
kubectl -n wordpress apply -f wordpress-sue-workshop-nl.yaml

certificate.cert-manager.io/wordpress-sue-workshop-nl created

```

Check if the certifcate was issued succesfully:

```
kubectl -n wordpress get certificates

NAME                        READY   SECRET                       AGE
wordpress-sue-workshop-nl   True    wordpress-sue-workshop-tls   2m26s
```

## Wordpress

Next, we will install wordpress, to showcase that we can use our newly issued certificate using Ingress:

First, add the bitnami helm repository:

```
helm repo add bitnami https://charts.bitnami.com/bitnami

"bitnami" has been added to your repositories
```

Update the helm repositories:

```
helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Install the wordpress helm template:

```
helm install -n wordpress wordpress-sue-workshop bitnami/wordpress
```

## Ingress

Next, enable the ingress addon in minikube:

```
minikube addons enable ingress
```

Create Ingress for wordpress:

```
cat > ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
spec:
  tls:
  - hosts:
      - wordpress.sue-workshop.nl
    secretName: wordpress-sue-workshop-tls
  rules:
    - host: wordpress.sue-workshop.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wordpress-sue-workshop
                port:
                  number: 80
EOF

kubectl -n wordpress apply -f ingress.yaml

ingress.networking.k8s.io/wordpress-ingress created
```

**=================== IMPORTANT IMPORTANT IMPORTANT ===================**

**WHEN ON A MAC USING THE MINIKUBE DOCKER DRIVER**

When on a Mac using the docker driver for Minikube Ingress is BROKEN. This is a known bug for 2 (!) years now.

Ugly fix to circumvent this:

Open a NEW TERMINAL and execute the following commands:

Find the minikube API ssh port:

```
docker port minikube | grep 22

22/tcp -> 127.0.0.1:54443
```

Open a ssh connection to the minikube API and tunnel traffic over SSH for HTTP/HTTPS:

```
sudo ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -N docker@127.0.0.1 -p <MINIKUBE_SSH_API_PORT> -i /Users/<YOUR_USERNAME>/.minikube/machines/minikube/id_rsa -L 80:127.0.0.1:80 -L 443:127.0.0.1:443
```

Leave this command running, switch to another terminal, and add an entry in /etc/hosts for wordpress.sue-workshop.nl:

```
sudo echo "127.0.0.1 wordpress.sue-workshop.nl" >> /etc/hosts
```

When not on a mac (or using another driver than docker), the following commands should suffice:

```
minikube ip

<SOME_IP_ADDRESS>

echo "<SOME_IP_ADDRESS> wordpress.sue-workshop.nl" >> /etc/hosts
```

Open https://wordpress.sue-workshop.nl in your favourite webbrowser, and you should see our newly 
issued certificate (with the obvious security warning due to an self signed CA) securing the connection.

## That's it! 

Thanks for your attention!
