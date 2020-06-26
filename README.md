# Cloud Service Broker on k8s

this repo walks through installing the CSB on a k8s cluster in a VMC environment. this will not cover the details of setting up TKG from scratch on VMC for that you can go here. This will cover installing the CSB for AWS and using the k8s service catalog to interact with it.

this guide can likely be used for any k8s cluster

## pre-reqs


## Building the CSB Container
 
 As of writing this, there is not a published CSB container image so we will go through the process of making one. if you choose to skip this I have one prebuilt on dockerhub that will be used by default `warroyo90/csb:0.1.0.rc18`

1. clone the CSB repo and checkout the latest released tag, as of the time of writing this the below is the latest tag for AWS. 

```bash
git clone https://github.com/pivotal/cloud-service-broker
git checkout sb-0.1.0-rc.18-aws-0.0.1-rc.92
```

2. build the code , you will need golang 1.14 installed.

```bash
cd cloud-service-broker
make build
```

3. build the cloud broker-paks. 

```bash
make build-brokerpak-aws
make build-brokerpak-gcp
make build-brokerpak-azure
```

4. package up the container and publish to your registry

```bash
docker build -t <your-registry>/csb:0.1.0.rc18 .
docker push <your-registry>/csb:0.1.0.rc18
```

5. update `manifests/kustomization.yml` to use your image instead of  mine.

## install the k8s service catalog

the service catalog is a way to integrate a service broker with k8s. docs are here to read me about it.

1. install via helm3

```bash
helm repo add svc-cat https://svc-catalog-charts.storage.googleapis.com
helm install catalog svc-cat/catalog --namespace catalog
```

## Deploying CSB on K8s

this assumes you have logged into your k8s cluster.

1. make sure there is  a storage class , the mysl db for the csb will need to create PVs. if not create one, this will be specific to the cloud you are running in.

2. generate the broker config. export the below environment variables and run the script for the cloud you want to use.  this will be used to generate a config map for the csb app. 

### For AWS:
Required environment variables
| Variable | Value |
|----------|-------|
| AWS_ACCESS_KEY_ID | access key id |
| AWS_SECRET_ACCESS_KEY | secret access key |

```bash
./build-config.sh aws
```

### For Azure:
Required environment variables
| Variable | Value |
|----------|-------|
|ARM_TENANT_ID|ID for tenant that resources will be created in|
|ARM_SUBSCRIPTION_ID|ID for subscription that resources will be created in|
|ARM_CLIENT_ID|service principal client ID|
|ARM_CLIENT_SECRET|service principal secret|

```bash
./build-config.sh azure
```

### For GCP:
Required environment variables
| Variable | Value |
|----------|-------|
|GOOGLE_CREDENTIALS| gcp service account json |
|GOOGLE_PROJECT| gcp project |


```bash
scripts/build-config.sh gcp
```

validate config was updated

```bash
cat manifests/config-files/broker-config.yaml
```


3. deploy the CSB along with the `ClusterServiceBroker` to wire it into the service catalog

```bash
kubectl apply -k ./manifests
```



