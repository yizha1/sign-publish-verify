# Hands-on sign, publish and verify container images

## Why

Ensuring the integrity and authenticity of container images

## The End-to-end scenario

Signing:
- Notary Project tooling Notation CLI

Publishing:
- ORAS

Verification:
- OPA Gatekeeper
- Ratify
- Helm

CI/CD:
- Docker Desktop (Simulation)


## Build a container image

```shell
export IMAGE=docker.io/yizha1/net-monitor:v1

docker buildx build . -f Dockerfile -o type=oci,dest=net-monitor.tar -t $IMAGE

mkdir net-monitor

tar -xf net-monitor.tar -C net-monitor
```

## Sign a container image

### Key Management

```shell
notation cert generate-test 2.23.io --default
notation cert ls
notation key ls
```

### List signatures

```shell
export NOTATION_EXPERIMENTAL=1
notation ls --oci-layout net-monitor:v1
```

### Sign the image

```shell
notation sign --oci-layout ./net-monitor:v1
```

### List signatures

```shell
notation list --oci-layout ./net-monitor:v1
```

## Publishing both the container image and signature


```shell
oras cp -r ./net-monitor:v1 --from-oci-layout docker.io/yizha1/net-monitor:v1
```

### View from docker hub

Switch to the docker hub to view the image


## Verify a container image in CI/CD pipelines.

Note: Use Notation CLI to simulate CI/CD pipelines

```json
{
 "version": "1.0",
 "trustPolicies": [
    {
         "name": "mypolicy",
         "registryScopes": [ "docker.io/yizha1/net-monitor" ],
         "signatureVerification": {
             "level" : "strict"
         },
         "trustStores": [ "ca:mycert" ],
         "trustedIdentities": [
             "x509.subject: CN=2.23.io,O=Notary,L=Seattle,ST=WA,C=US"
         ]
     }
 ]
}
```

```shell
notation policy import mypolicy.json
notation policy show
notation verify $IMAGE
```

## Verify a container image before pulling to K8s

### Install Gatekeeper

```shell
helm repo add gatekeeper  https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace --set enableExternalData=true --set validatingWebhookTimeoutSeconds=5 --set mutatingWebhookTimeoutSeconds=2 --set externaldataProviderResponseCacheTTL=10s
```

### Install Ratify

```shell
helm repo add ratify https://ratify-project.github.io/ratify
helm install ratify ratify/ratify --atomic --namespace gatekeeper-system --set-file notationCerts[0]="2.23.io.crt" --set featureFlags.RATIFY_CERT_ROTATION=true --set notation.trustPolicies[0].registryScopes[0]="docker.io/yizha1/net-monitor" --set notation.trustPolicies[0].trustStores[0]=ca:notationCerts[0] --set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: CN=2.23.io\,O=Notary\,L=Seattle\,ST=WA\,C=US"
```



### Set up policies

```shell
kubectl apply -f https://ratify-project.github.io/ratify/library/default/template.yaml
kubectl apply -f https://ratify-project.github.io/ratify/library/default/samples/constraint.yaml
```

## Check the results

### Verify an unsigned image

```shell
export UNSIGNED=docker.io/yizha1/unsigned:v1
kubectl run demo-unsigned --image=$UNSIGNED
```

### Verify a signed image

```shell
kubectl run demo-signed --image=$IMAGE
```

