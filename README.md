# Aqua Container Security Platform (CSP) for Amazon EKS

This github repo retains the helm charts for Aqua Security's AWS EKS Marketplace offering. This readme includes reference documention regarding installation and removals while operating within AWS EKS.

Installation is simple, as Cloud Native apps should be! There are minimal pre-requsites to attend to in order to deploy Aqua CSP on AWS EKS as defined below.

>**A word about Helm**
>
>This is a basic deployment guide and not intended to cover all installation variations.
>While Helm is a very easy tool to use, *Helm is not required.* Many other installation variations are available to customers with an `https://my.aquasec.com` account. Cloud Market specific documentation is located at `https://cloud-market-docs.aquasec.com`

## Prerequisites

* A current installation of [Helm](https://helm.sh/)
* EKS role binding appropriate for Helm's use
* Database options
* Extend EKS with a StorageClass that supports EBS
* AWS Marketplace Subscription to the [Aqua CSP EKS offer.](https://aws.amazon.com/marketplace/pp/B07KCNBW7B)

### Aquiring the charts

The Aqua console components are non-FOSS, therefore this chart is not available in the Helm package repository.  However, you may simply clone this repository and install via Helm from this collection.

```shell
git clone https://github.com/aquasecurity/aws-marketplace-eks-byol.git
```

### Helm EKS Role Binding

#### Using helm with EKS requires providing a service account for use by tiller

* For Helm 3.x, Tiller is not required for the Helm installation.

* For Helm 2.x
Run the following commands to create the requisite SA for Tiller and give it appropriate permissions:

```bash
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

### Database Options

This helm chart includes an Aqua provided PostgreSQL database container for small environments and/or testing scenerios. For production deployments Aqua recommends implementing a dedicated database such as Amazon RDS. 

#### RDS requirements
A production-grade Aqua CSP deployment requires a managed Postgres database installation. Following are the requirements:
```bash
1. Engine type: PostgreSQL
2. Version: 9.6.9
3. DB instance size: Allowed values[db.t2.micro, db.t2.small, db.t2.medium, db.t2.large, db.t2.xlarge, db.t2.2xlarge,
                                   db.m4.large, db.m4.xlarge, db.m4.2xlarge, db.m4.4xlarge, db.m4.10xlarge, db.m4.16xlarge,
                                   db.r4.large, db.r4.xlarge, db.r4.2xlarge, db.r4.4xlarge, db.r4.8xlarge, db.r4.16xlarge,
                                   db.r3.large, db.r3.2xlarge, db.r3.4xlarge, db.r3.8xlarge]
4. Storage type: General Purpose or Provisioned IOPS based on the environment
5. Allocated storage: 40GB (minimum)
6. Multi-AZ deployment: enabled/disabled based on the environment
7. Connectivity: For multi-cluster deployments, make RDS publicly accessible else deploy it in the same VPC
```

The helm chart may be modified to utilize such an external instance by modifying the file *aws-marketplace-eks-byol/aqua/values.yaml*, section *dbExternalServiceHost* as in the example below.

```shell
dbExternalServiceHost:"<myserver>.CB2XKFSFFMY7.US-WEST-2.RDS.AMAZONAWS.COM"
```

### Extend EKS with an EBS supported StorageClass

If you are using an external PostgreSQL provider such as RDS this step is unnecessary. If you are deploying EKS clusters with Kubernetes version above 1.11, this step is unnecessary. 

Per [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html)
EKS does not ship with any StorageClasses for clusters that were created prior to Kubernetes version 1.11. Included in the git repo is the file *aws-marketplace-eks-byol/gp2-storage-class.yaml*. Apply this file to add support for EBS volumes and set the gp2 StorageClass as default for the cluster. Alternatively, edit the database chart to utilize your own StorageClass.

```shell
kubectl create -f gp2-storage-class.yaml
```

## Secrets and Service Account

Please ignore this section if you are deploying to EKS from the AWS Marketplace (AWS MP) directly. The following section describing the dockerImagePull secrets is unnecessary as AWS MP authorizes the image pull from the AWS MP ECR.

Only if you want to use a privately hosted repository for the Aqua images, refer to this section. The Aqua console components are hosted on a private repository: `registry.aquasec.com`. As such a service account and associated dockerImagePull secret are required to be created. The Helm chart does this for you. Edit the *aws-marketplace-eks-byol/aqua/values.yaml* to include the credentials that were granted permission to download from Aqua Security's private repository. Many customers also utilize privatly hosted registries. If this is your scenerio, change the `registry:` variable to match as well.

```shell
  imageCredentials:
    registry: "registry.aquasec.com"
    username: "example@aquasec.com"
    password: "k8s4allis@sh0rtP@ssword"
```

## Installing the Helm chart

### Create aqua namespace
```shell
kubectl create ns aqua
```

### Install Helm chart

* For Helm 2.x
```
helm install --namespace aqua --name csp ./aqua
```

* For Helm 3.x
```
helm install --namespace aqua csp ./aqua
```

## Complete Initial Deployment

Helm will deploy the Aqua Command Center and accompanying Aqua Enforcers set to audit mode. This process takes approximatly five minutes. The time-consuming part of the deployment is the ELB recognizing the containers are available after the deployment. Watching the ELB status in AWS EC2 console is possilbe in the AWS EC2 console. The AWS cli counterpart may be used to poll status as well. Replace the `releaseName (csp)` and `namespace (aqua)` with your variables if you so choose.

>Note: The `helm install` command will output similar commands with these variables pre-populated.

```shell
EKSELB=$(kubectl get svc csp-aqua-console --namespace aqua -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"|sed 's/-.*//')
  
aws elb describe-instance-health --load-balancer-name $EKSELB
```

You will see something similar to the below output while waiting for the system to come online.

```shell
{
    "InstanceStates": [
        {
            "InstanceId": "i-0f90dcf0238b343fb",
            "State": "OutOfService",
            "ReasonCode": "ELB",
            "Description": "Instance registration is still in progress."
        }
    ]
}
```

When the Instance State is operational, the below output will match.

```shell
{
    "InstanceStates": [
        {
            "InstanceId": "i-0f90dcf0238b343fb",
            "State": "InService",
            "ReasonCode": "ELB",
            "Description": "Instance registration is still in progress."
        }
    ]
}
```

## The following four basic steps are necessary to complete deployment

## 1. Backup Auto-Generated Secrets

There are four secrets generated: `admin password, database password, enforcer token` and `registry auth` which is used by the docker pull service account.

By default the Aqua PostgreSQL container utilizes a persistant volume (PVC). When removing the application, this PVC is not deleted along with the other componants in order to save your data.
In the case of a re-deploy, reloading these secrets will be necessary to access the db files on the reused PVC. It is **very important** to back up the database password secrets for this purpose.
Please back them up ***now***. See the [ReDeploying Aqua CSP](#ReDeploying-Aqua-CSP) section for redeployment instructions.

```shell
kubectl get secrets -l secretType=aquaSecurity \
--namespace aqua -o json > aquaSecrets.json
```
## 2. Obtain the Aqua Command Center administrator password

The default username is `administrator`. Use `kubectl` to extract the generated password from the secret.

```shell
kubectl get secret csp-admin-password --namespace aqua -o json | jq -r .data.password | base64 -D
```



## 3. Obtain the Aqua Command Center portal information and login

A user may run the following command:

```shell
AQUA_CONSOLE=$(kubectl get svc csp-aqua-console --namespace aqua -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
  
ECHO "http://$AQUA_CONSOLE:8080"
```

## 4. Enter the license to enable the product

Users that previously registered for a license token for use with AWS Container Marketplace (PAYG) deployments should enter it to enable the hourly billing of Enforcers. If you do not have a license token, you may request one by filling out the form linked on the Aqua Command Center startup portal.

>*A note about Aqua CSP for AWS Marketplace licenses*
>
>The license issued is specific to the environement. As of this writing an Enterprise license will not enable a deplyment via AWS Container Marketplace or vice versa without changing startup variables. A complete list of startup variables are documented at `https://docs.aquasec.com`

## View logs of the Aqua Command Center

Sometimes an admin just needs the read some logs. While these are accessible in the Aqua Console under Settings > Logs, should you need to access logs of the Aqua server pod via CLI, use the below command.

```shell
CONSOLEPOD=$(kubectl get pods -l app=csp-aqua-console -n aqua --no-headers -o=custom-columns=NAME:.metadata.name)
kubectl logs -f ${CONSOLEPOD} --namespace=aqua
```

## ReDeploying Aqua CSP
### External PostgreSQL container in use

Redeploying when utilizing an external PostgreSQL is detailed below. One must either edit or replace the secrets that allow the componants to communicate.

1. Backup then Delete your existing secrets and services

```shell
kubectl get secrets -l secretType=aquaSecurity \
--namespace aqua -o json > aquaSecrets.json

cat aquaSecrets.json (or use your favorite editor to validate content)

kubectl delete -f aquaSecrets.json
kubectl delete sa -n aqua csp-sa
```
2. Run the helm installer with the `exact same release name`

```shell
helm install --namespace aqua --name csp ./aqua
 ```

3. Wait 15 seconds, then re-delete and reapply the secrets from your backup file.

```bash
kubectl delete -f aquaSecrets.json
kubectl apply -f aquaSecrets.json
```
 
4. Check the console as in the above installation section [Complete Initial Deployment](#Complete-Initial-Deployment)


### Aqua PostgreSQL container in use

The Aqua provided PostgreSQL container uses a Persistant Volume Claim (PVC) set to `retain` upon `helm delete` in order to safe-guard inadvertant database loss. A [PVC](https://kubernetes.io/docs/concepts/storage/volumes/#creating-an-ebs-volume) is a mechanisim within kubernetes that allows an application to mount a physical volume (PV) as a kubernetes volume. This grants the PV reusability, among other capabilities.

To redeploy Aqua CSP and reattach the previously utilized PV, one may utilize the same cluster, namespace and Helm release name. Doing so will cause kubernetes to attempt to reattach the matching PV. This presents a challenge however due to the *pv.claimRef.uid* that links the PVC to the PV. The PV for security purposes will only allow a *specific* PVC UID to make a claim on itself. The helm chart will also regenerate the necessary secrets. Worse yet, reapplying the wrong backup can cause the database connection from the Aqua console and database containers to fail. To allieviate this particular issue, stage the commands from step number four below in the a shell and run it 20 seconds after redpoluing via `helm install`. Doing so will replace the secrets with the backup values, and allow the console and gateway pods to reconnect to the database.

1. Backup then Delete your existing secrets and services

```shell
kubectl get secrets -l secretType=aquaSecurity \
--namespace aqua -o json > aquaSecrets.json

cat aquaSecrets.json (or use your favorite editor to validate content)

kubectl delete -f aquaSecrets.json
kubectl delete sa -n aqua csp-sa
```

2. Obtain the PVC information and Helm release name to reuse

```shell
kubectl get pvc -n aqua

NAME               STATUS   VOLUME
csp-database-pvc   Bound    pvc-df93aca6-d6fa-11e8-a39b-0a14904e5754
```

3. Delete the PVC

```shell
kubectl delete pvc -n aqua csp-database-pvc
```

4. Obtain the PV name

```shell
kubectl get pv -n aqua

NAME                                     CAPACITY ACCESS MODES RECLAIM POLICY STATUS  
pvc-df93aca6-d6fa-11e8-a39b-0a14904e5754 50Gi     RWO          Retain         Released
```
>Note: If the status is anything other then `Released` at this point stop and retrace your steps as it'll be difficult to procede.

4. Edit the PV to `unlock` the dynamically generated PVC UID that is specified.

```shell

kubectl edit pv -n aqua pvc-df93aca6-d6fa-11e8-a39b-0a14904e5754

"""Dynamic Example"""
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: csp-database-pvc
    namespace: aqua
    resourceVersion: "931154"
    uid: df93aca6-d6fa-11e8-a39b-0a14904e5754

"""Reused, existing PV example"""
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: csp-database-pvc
    namespace: aqua
    resourceVersion: "931154"
    uid:
```

5. Check that the status of the PV has changed to `Available`

```shell
NAME                                     CAPACITY ACCESS MODES RECLAIM POLICY STATUS  
pvc-df93aca6-d6fa-11e8-a39b-0a14904e5754 50Gi     RWO          Available         Released
```

6. Run the helm installer with the `exact same release name`

```shell
helm install --namespace aqua --name csp ./aqua
 ```

7. Wait 15 seconds, then reapply the secrets from your backup file.

```bash
kubectl apply -f aquaSecrets.json
```
 
8. Check the console as in the above installation section [Complete Initial Deployment](#Complete-Initial-Deployment)


## Uninstalling Aqua CSP

Uninstalling the Aqua CSP and all componants may be performed by the following functions:

Delete the helm release

```shell
Helm delete csp
```

Delete the namespace

```shell
kubectl delete ns aqua
```

Remove any EBS volumes via the AWS CLI or Console
