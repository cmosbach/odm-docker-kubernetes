# Deploying IBM Operational Decision Manager on Google GKE

This project demonstrates how to deploy an IBM® Operational Decision Manager (ODM) clustered topology with the [container-native load balancer of GKE](https://cloud.google.com/blog/products/containers-kubernetes/container-native-load-balancing-on-gke-now-generally-available).
The ODM services are exposed through the Ingress provided by the ODM on Kubernetes Helm chart.
This deployment implements Kubernetes and Docker technologies.
Here is the home page of Google Cloud: https://cloud.google.com

<img width="1000" height="560" src='./images/architecture.png'/>

The ODM Docker material is available in Passport Advantage. It includes Docker container images and Helm chart descriptors.

## Included components

The project comes with the following components:

- [IBM Operational Decision Manager](https://www.ibm.com/docs/en/odm/8.11.0)
- [Google Cloud SQL for PostgreSQL](https://cloud.google.com/sql)
- [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine)
- [IBM License Service](https://github.com/IBM/ibm-licensing-operator)

## Tested environment

The commands and tools have been tested on macOS and Linux.

## Prerequisites

First, install the following software on your machine:

- [gcloud CLI](https://cloud.google.com/sdk/gcloud)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm v3](https://helm.sh/docs/intro/install/)
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)

Then, perform the following tasks:

1. [Create a Google Cloud account](https://cloud.google.com/apigee/docs/hybrid/v1.6/precog-gcpaccount)

2. [Create a Google Cloud project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)

3. [Manage the associated billing](https://cloud.google.com/billing/docs/how-to/modify-project#confirm_billing_is_enabled_on_a_project)

Without the relevant billing level, some Google Cloud resources will not be created.

> Note:  Prerequisites and software supported by ODM 8.11 are listed on [the Detailed System Requirements page](https://www.ibm.com/software/reports/compatibility/clarity-reports/report/html/softwareReqsForProduct?deliverableId=2D28A510507B11EBBBEA1195F7E6DF31&osPlatforms=AIX%7CLinux%7CMac%20OS%7CWindows&duComponentIds=D002%7CS003%7CS006%7CS005%7CC006&mandatoryCapIds=30%7C1%7C13%7C25%7C26&optionalCapIds=341%7C47%7C9%7C1%7C15).

## Steps to deploy ODM on Kubernetes from Google GKE

<!-- TOC titleSize:2 tabSpaces:2 depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 skip:0 title:0 charForUnorderedList:* -->
* [Deploying IBM Operational Decision Manager on Google GKE](#deploying-ibm-operational-decision-manager-on-google-gke)
  * [Included components](#included-components)
  * [Tested environment](#tested-environment)
  * [Prerequisites](#prerequisites)
  * [Steps to deploy ODM on Kubernetes from Google GKE](#steps-to-deploy-odm-on-kubernetes-from-google-gke)
  * [Prepare your GKE instance (30 min)](#prepare-your-gke-instance-30-min)
    * [Log into Google Cloud](#log-into-google-cloud)
    * [Create a GKE cluster](#create-a-gke-cluster)
    * [Set up your environment to this cluster](#set-up-your-environment-to-this-cluster)
  * [Create the Google Cloud SQL PostgreSQL instance (10 min)](#create-the-google-cloud-sql-postgresql-instance-10-min)
  * [Prepare your environment for the ODM installation](#prepare-your-environment-for-the-odm-installation)
    * [Using the IBM Entitled registry with your IBMid (10 min)](#using-the-ibm-entitled-registry-with-your-ibmid-10-min)
    * [Create the datasource secrets for Google Cloud SQL PostgreSQL](#create-the-datasource-secrets-for-google-cloud-sql-postgresql)
    * [Manage a digital certificate (2 min)](#manage-a-digital-certificate-2-min)
  * [Install an ODM Helm release using the GKE loadbalancer (10 min)](#install-an-odm-helm-release-using-the-gke-loadbalancer-10-min)
    * [Manage a PV containing the JDBC driver](#manage-a-pv-containing-the-jdbc-driver)
    * [Install the ODM release](#install-the-odm-release)
    * [Check the topology](#check-the-topology)
    * [Check the Ingress and GKE LoadBalancer](#check-the-ingress-and-gke-loadbalancer)
    * [Create a Backend Configuration for the Decision Center Service](#create-a-backend-configuration-for-the-decision-center-service)
    * [Access ODM services](#access-odm-services)
  * [Install the IBM License Service and retrieve license usage](#install-the-ibm-license-service-and-retrieve-license-usage)
    * [Create a NGINX Ingress controller](#create-a-nginx-ingress-controller)
    * [Install the IBM License Service](#install-the-ibm-license-service)
    * [Create the Licensing instance](#create-the-licensing-instance)
    * [Retrieving license usage](#retrieving-license-usage)
  * [Optional steps](#optional-steps)
  * [Troubleshooting](#troubleshooting)
* [License](#license)
<!-- /TOC -->

## Prepare your GKE instance (30 min)

Source: https://cloud.google.com/kubernetes-engine/docs/quickstart

### Log into Google Cloud

After installing the gcloud tool, use the following command line:

```
gcloud auth login [ACCOUNT]
```
https://cloud.google.com/sdk/gcloud/reference/auth/login

### Create a GKE cluster

There are several [types of cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/types-of-clusters).
In this article, you create a [regional cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster).

Set the project (associated to a billing account):

```
gcloud config set project [PROJECT_NAME]
```

Set the zone:

```
gcloud config set compute/zone [ZONE (ex: europe-west1-b)]
```

Set the region:

```
gcloud config set compute/region [REGION (ex: europe-west1-b)]
```

Create a cluster and enable autoscaling. Here, you start with 4 nodes (with 16 max.):

```
gcloud container clusters create [CLUSTER_NAME] --num-nodes 4 --enable-autoscaling --min-nodes 1 --max-nodes 16
```

You can also create your cluster from the Google Cloud Platform by clicking the Create button to display the Kubernetes Engine Clusters panel.

<img width="1000" height="300" src='./images/create_cluster.png'/>

### Set up your environment on this cluster

To configure kubectl, use the following command line:
```
gcloud container clusters get-credentials [CLUSTER_NAME]
```

You can also retrieve the command line to configure kubectl from the Google Cloud Platform, in the Kubernetes Engine>Cluster panel, by selecting "Connect" for the dedicated cluster.

<img width="1000" height="300" src='./images/connection.png'/>

Now, you can check that kubectl works fine:

```
kubectl get pods
```

## Create the Google Cloud SQL PostgreSQL instance (10 min)

Use the Google Cloud Platform to create this instance:

- Go to the [SQL context](https://console.cloud.google.com/sql) and click "CREATE INSTANCE".
- Choose PostgreSQL.
  - Take "PostgreSQL 13" database version.
  - Choose a region similar to the cluster, so that the communication is optimal between the database and the ODM instance.
  - Keep "Multiple zones" for Zonal availability to be the highest.
  - Expand "Customize your instance", and then expand "Connections".
  - As Public IP is selected by default, click "ADD NETWORK", enter a name and add "0.0.0.0/0" for Network, then click "DONE".

After the instance is created, you can drill on the SQL instance overview to retrieve needed information to connect to this instance, like the IP adress and the connection name:

<img width="1000" height="630" src='./images/database_overview.png'/>

A default "postgres" database is created with a default "postgres" user. You can change the password of the postgres user in the Users panel by clicking "Change password".

<img width="1000" height="360" src='./images/database_changepassword.png'/>

## Prepare your environment for the ODM installation

To get access to the ODM material, you must have an IBM entitlement registry key to pull the images from the IBM Entitled Registry.

### Using the IBM Entitled Registry with your IBMid (10 min)

Log in to [MyIBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) with the IBMid and password that are associated with the entitled software.

In the Container software library tile, verify your entitlement on the View library page, then go to Get entitlement key to retrieve the key.

Create a pull secret by running a kubectl create secret command.

```
$ kubectl create secret docker-registry <registrysecret> --docker-server=cp.icr.io \
                                                         --docker-username=cp \
                                                         --docker-password="<entitlementkey>" \
                                                         --docker-email=<email>
```

where:

* \<registrysecret\> is the secret name.
* \<entitlementkey\> is the entitlement key from the previous step. Make sure you enclose the key in double-quotes.
* \<email\> is the email address associated with your IBMid.

> Note:  The cp.icr.io value for the docker-server parameter is the only registry domain name that contains the images. You must set the docker-username to cp to use the entitlement key as docker-password.

Make a note of the secret name so that you can set it for the image.pullSecrets parameter when you run a helm install of your containers.  The image.repository parameter will later be set to cp.icr.io/cp/cp4a/odm.

Add the public IBM Helm charts repository:

```
helm repo add ibmcharts https://raw.githubusercontent.com/IBM/charts/master/repo/entitled
helm repo update
```

Check that you can access the ODM charts:

```
helm search repo ibm-odm-prod
NAME                  	CHART VERSION	APP VERSION	DESCRIPTION                     
ibmcharts/ibm-odm-prod	21.3.0       	8.11.0.0   	IBM Operational Decision Manager
```

You can now proceed to the [creation of the datasource secret](#create-the-datasource-secrets-for-azure-postgresql).

### Create the datasource secrets for Google Cloud SQL PostgreSQL

The Google Cloud SQL PostgreSQL connection is established through [Cloud SQL Connector for Java](https://github.com/GoogleCloudPlatform/cloud-sql-jdbc-socket-factory#cloud-sql-connector-for-java).

If you do not want to build the driver, you can get the last [driver](https://storage.googleapis.com/cloud-sql-java-connector/) named postgres-socket-factory-X.X.X-jar-with-driver-and-dependencies.jar.

The test was performed with the driver version [postgres-socket-factory-1.4.2-jar-with-driver-and-dependencies.jar](https://storage.googleapis.com/cloud-sql-java-connector/v1.4.2/postgres-socket-factory-1.4.2-jar-with-driver-and-dependencies.jar).

Copy the files [datasource-dc.xml.template](datasource-dc.xml.template) and [datasource-ds.xml.template](datasource-ds.xml.template) to your local machine and rename them `datasource-dc.xml` and `datasource-ds.xml`.

Replace the following placeholders:

- DRIVER_VERSION - The Cloud SQL Connector for Java Version (e.g. 1.4.2)
- IP - The public IP address
- CONNECTION_NAME - The database connection name
- DBNAME - The database name (default is postgres)
- USERNAME - The database username (default is postgres)
- PASSWORD - The database password

The code should look like this:

```xml
...
 <library id="postgresql-library">
            <fileset id="postgresql-fileset"  dir="/drivers" includes="postgres-socket-factory-<DRIVER_VERSION>-jar-with-driver-and-dependencies.jar" />
 </library>
...
        <properties URL="jdbc:postgresql://<IP>/<DBNAME>?cloudSqlInstance=<CONNECTION_NAME>;socketFactory=com.google.cloud.sql.postgres.SocketFactory"
                    user="<USERNAME>"
                    password="<PASSWORD>"/>
...
```

Create a secret with the two modified files:

```
kubectl create secret generic <customdatasourcesecret> \
        --from-file datasource-ds.xml --from-file datasource-dc.xml
```

### Manage a digital certificate (2 min)

1. Generate a self-signed certificate.

If you do not have a trusted certificate, you can use OpenSSL and other cryptography and certificate management libraries to generate a certificate file and a private key, to define the domain name, and to set the expiration date. The following command creates a self-signed certificate (.crt file) and a private key (.key file) that accepts the domain name *mycompany.com*. The expiration is set to 1000 days:

```
openssl req -x509 -nodes -days 1000 -newkey rsa:2048 -keyout mycompany.key \
        -out mycompany.crt -subj "/CN=mycompany.com/OU=it/O=mycompany/L=Paris/C=FR"
```

2. Create a TLS secret with these keys.

```
kubectl create secret tls mycompany-crt-secret --key mycompany.key --cert mycompany.crt
```

The certificate must be the same as the one you used to enable TLS connections in your ODM release. For more information, see [Server certificates](https://www.ibm.com/docs/en/odm/8.11.0?topic=servers-server-certificates) and [Working with certificates and SSL](https://docs.oracle.com/cd/E19830-01/819-4712/ablqw/index.html).

## Install an ODM Helm release using the GKE loadbalancer (10 min)

### Manage a PV containing the JDBC driver

To be able to use the PostgreSQL database that you created, you need a Persistent Volume (PV) to provide the JDBC driver to the ODM container.
Unfortunately, GKE PV does not support the [ReadWriteMany access mode](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes).
If you try a PV in ReadWriteMany access mode, only one ODM pod starts, the other pods fail with the following error:

```
Warning  FailedAttachVolume  ... : Googleapi: Error 400: RESOURCE_IN_USE_BY_ANOTHER_RESOURCE - The disk resource '...' is already being used by '...'
```

To work around this issue, use a ReadWriteOnce PV on an NGINX pod that has the root permission to copy the driver.
Then, change the PV permission to ReadOnlyMany before you launch the ODM release to be able to install ODM on many nodes.

1. [Enable the SCI FileStore Driver](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/filestore-csi-driver#console_1).

2. Create the [filestore-example](filestore-example.yaml) storageClass.

```
kubectl apply -f filestore-example.yaml
```

3. Create the [customdatasource-pvc](customdatasource-pvc.yaml) PVC using the filestore-example StorageClass in ReadWriteOnce access mode to copy the driver to the PV.

```
kubectl apply -f customdatasource-pvc.yaml
```

4. Create a [nginx](nginx.yaml) pod that uses this PVC. It serves only to copy the PostgresSQL driver because this container is accessible as root.

```
kubectl apply -f nginx.yaml
```

5. Copy the Google Cloud PostgresSQL driver to the nginx pod.

```
export NGINX_COPY_POD=$(kubectl get pod | grep nginx-driver-copy)
kubectl cp postgres-socket-factory-<X.X.X>-jar-with-driver-and-dependencies.jar $(NGINX_COPY_POD):/usr/share/nginx/html
```

6. Change the PV access mode to ReadOnlyMany. This way, all ODM containers will be able to access the PV in readonly mode and be scheduled on several nodes.

```
export PV_NAME=$(kubectl get pvc customdatasource-pvc -o jsonpath={.spec.volumeName})
kubectl patch pv $PV_NAME -p '{"spec":{"accessModes":["ReadOnlyMany"]}}'
```

### Install the ODM release

You can now install the ODM product.

The ODM instance uses the externalCustomDatabase parameters to import the PostgreSQL datasource and driver.
The ODM services are exposed with an Ingress that uses the previously created mycompany certificate.
The Ingress automatically creates an HTTPS GKE loadbalancer. Therefore, you can disable the ODM internal TLS as it is not needed.
The helm install command executes a kustomize as post-rendering step to change the Decision Server readiness, because the GKE loadbalancer uses it to create a service healthCheck that requires 200 as response code (ODM default is 301).

```
helm install <release> ibmcharts/ibm-odm-prod \
        --set image.repository=cp.icr.io/cp/cp4a/odm --set image.pullSecrets=<registrysecret> \
        --set externalCustomDatabase.datasourceRef=<customdatasourcesecret> --set externalCustomDatabase.driverPvc=customdatasource-pvc \
        --set service.enableTLS=false --set service.ingress.tlsSecretRef=mycompany-crt-secret \
        --set service.ingress.enabled=true --set service.ingress.host=mycompany.com --set service.ingress.tlsHosts={"mycompany.com"} \
        --post-renderer ./kustomize
```

### Check the topology

Run the following command to check the status of the pods that are created:

```
kubectl get pods
NAME                                                   READY   STATUS    RESTARTS   AGE
<release>-odm-decisioncenter-***                       1/1     Running   0          20m
<release>-odm-decisionrunner-***                       1/1     Running   0          20m
<release>-odm-decisionserverconsole-***                1/1     Running   0          20m
<release>-odm-decisionserverruntime-***                1/1     Running   0          20m
```

### Check the Ingress and GKE LoadBalancer

To get a status of the current deployment, go to the [Kubernetes Engine/Services & Ingress Panel](https://console.cloud.google.com/kubernetes/ingresses) in the Google Cloud Platform.
The Ingress remains in the "ingress creating" state for several minutes until the pods are up and ready, and the backend  reaches a healthy state.

<img width="1000" height="308" src='./images/ingress_creating.png'/>

You can also check the [load balancer status](https://console.cloud.google.com/net-services/loadbalancing/list/loadBalancers).
It provides information about the backend through the service health check.

<img width="1000" height="352" src='./images/loadbalancer.png'/>

In the Ingress details, you should get a HEALTHY state on all backends.
This panel also provides logs on the loadbalancer activity.
When the Ingress shows an OK status, all ODM services are available.

<img width="1000" height="517" src='./images/ingress_details.png'/>

### Create a Backend Configuration for the Decision Center Service

A sticky session is needed for Decision Center. The browser contains a user session cookie that is linked to a unique container. 
The ODM on Kubernetes Helm chart has a [clientIP](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-ipvs) for the Decision Center session affinity, but GKE does not use it automatically.
However, you will not encounter any issue until you scale up the Decision Center deployment.

A [BackendConfig](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-features#session_affinity) configuration is needed at the loadbalancer level.

1. Create the [Decision Center Backend Config](decisioncenter-backendconfig.yaml).

```
kubectl apply -f decisioncenter-backendconfig.yaml
```

2. Annotate the Decision Center Service with the GKE Backend Config.

```
kubectl annotate service <release>-odm-decisioncenter cloud.google.com/backend-config="{\"ports\": {\"9453\":\"dc-backendconfig\"}}"
```

As soon as GKE manages the Decision Center session affinity at the loadbalancer level, you can check the ClientIP availability under the Decision Center Network Endpoint Group configuration in the Load Balancer details panel of the Google Cloud Platform.

<img width="1000" height="353" src='./images/dc_sessionaffinity.png'/>

### Access ODM services

In a real enterprise use-case, to access the mycompany.com domain name, you have to deal with [Google Managed Certificate](https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs) and [Google Cloud DNS](https://cloud.google.com/dns).

In this trial, you use a self-signed certificate. So, there is no extra charge like certificate and domain purchase.
You just need to manage a configuration to simulate the mycompany.com access.

To get the EXTERNAL-IP, use the command line:

```
kubectl get ingress <release>-odm-ingress -o json | jq -r '.status.loadBalancer.ingress[].ip'
```

To access ODM services, you must edit your /etc/hosts file and add the following entry:

```
<EXTERNAL-IP> mycompany.com
```

Now, you can access all ODM services with the following URLs:

| SERVICE NAME | URL | USERNAME/PASSWORD
| --- | --- | ---
| Decision Server Console | https://mycompany.com/res | odmAdmin/odmAdmin
| Decision Center | https://mycompany.com/decisioncenter | odmAdmin/odmAdmin
| Decision Center REST-API | https://mycompany.com/decisioncenter-api | odmAdmin/odmAdmin
| Decision Server Runtime | https://mycompany.com/DecisionService | odmAdmin/odmAdmin
| Decision Runner | https://mycompany.com/DecisionRunner | odmAdmin/odmAdmin

You can also click on the Ingress routes that are accessible from the Google Cloud Platform below the [Kubernetes Engine/Services & Ingress Details Panel](https://console.cloud.google.com/kubernetes/ingresses):

<img width="1000" height="532" src='./images/ingress_routes.png'/>

## Install the IBM License Service and retrieve license usage

This section explains how to track ODM usage with the IBM License Service.

### Create an NGINX Ingress controller

1. Add the official stable repository.

    ```
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    ```

2. Use Helm to deploy an NGINX Ingress controller.

    ```
    helm install nginx-ingress ingress-nginx/ingress-nginx \
      --set controller.replicaCount=2 \
      --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
      --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
    ```

3. Get the Ingress controller external IP address.

    ```
    kubectl get service -l app.kubernetes.io/name=ingress-nginx
    NAME                                               TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
    nginx-ingress-ingress-nginx-controller             LoadBalancer   10.0.191.246   <externalip>   80:30222/TCP,443:31103/TCP   3m8s
    nginx-ingress-ingress-nginx-controller-admission   ClusterIP      10.0.214.250   <none>         443/TCP                      3m8s
    ```

### Install the IBM License Service

Follow the **Installation** section of the [Manual installation without the Operator Lifecycle Manager (OLM)](https://github.com/IBM/ibm-licensing-operator/blob/latest/docs/Content/Install_without_OLM.md). Do not follow the instantiation part!

### Create the Licensing instance

Run the following command:

```
kubectl create -f licensing-instance.yml
```

More information and use cases on [this page](https://github.com/IBM/ibm-licensing-operator/blob/latest/docs/Content/Configuration.md#configuring-ingress).

### Retrieving license usage

After a couple of minutes, the NGINX load balancer reflects the Ingress configuration and you are able to access the IBM License Service by retrieving the URL with the following command:

```
export LICENSING_URL=$(kubectl get ingress ibm-licensing-service-instance -n ibm-common-services |awk '{print $4}' |tail -1)/ibm-licensing-service-instance
export TOKEN=$(oc get secret ibm-licensing-token -o jsonpath={.data.token} -n ibm-common-services |base64 -d)
```

You can access the `http://${LICENSING_URL}/status?token=${TOKEN}` URL to view the licensing usage or retrieve the licensing report .zip file by running:

```
curl -v "http://${LICENSING_URL}/snapshot?token=${TOKEN}" --output report.zip
```

If your IBM License Service instance is not running properly, refer to this [troubleshooting page](https://github.com/IBM/ibm-licensing-operator/blob/latest/docs/Content/Troubleshooting.md).

## Optional steps

You might prefer to access ODM services through the NGINX Ingress controller instead of directly from the IP addresses.  If so, follow [these instructions](README_NGINX.md).

## Troubleshooting

If your ODM instances are not running properly, refer to [our dedicated troubleshooting page](https://www.ibm.com/docs/en/odm/8.11.0?topic=8110-troubleshooting-support).

# License

[Apache 2.0](/LICENSE)