## Setup the Lab Env

You need a Kubernetes Cluster to run the crossplane. Below steps will help you create and manage a kubernetes cluster on an EC2 instance using Docker and K3D.

1) Create an AWS Instance running a LINUX OS (Eg: Ubuntu Server LTS)

2) Install Docker by following the documentation => https://docs.docker.com/engine/install/ubuntu/

3) Allow ubuntu user to run docker commands

    ```
    sudo adduser ubuntu docker
    ```

4) Install K3D by following => https://k3d.io/

5) Create a Kubernetes cluster with k3d by running below command:

    ```k3d cluster create --agents=2```

6) Install Kubectl by following => https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/


## Install Crossplane 

#### 1) Set up Prerequisites: 
* Lab setup should be done following above instructions
* Helm must be installed on your laptop by following the instructions at https://helm.sh/docs/intro/install/

#### 2) Install Crossplane
You can install crossplane by following the instructions at https://docs.crossplane.io/latest/software/install/



## Install AWS Provider

You can search in the upbound marketplace or Google to get the list of providers. 

Some examples are

https://marketplace.upbound.io/providers/upbound/provider-aws/v0.37.0
https://github.com/crossplane-contrib/provider-aws
https://marketplace.upbound.io/providers/crossplane-contrib/provider-helm/v0.15.0

To install a provider, create Customer resource "provider" provided the cross-plane. 

```
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-eks
spec:
  package: xpkg.upbound.io/upbound/provider-aws-eks:v0.37.0
```

Save above content on a YAML and create the provider with kubectl apply -f <FILENAME>


After you installed the provider, run below command to verify the provider. 

```kubectl get providers # shows the Status of the prover 
   kubectl get pods -n crossplane-system  # See the status of the provider pod
```

Make sure that status=True in the above command before proceeding to next step. 


Now your provider is installed, its time to create a providerConfig. 

    1) Create a secret that looks like an aws credentials file inside Kubernetes.
        ```
        [default]
        aws_access_key_id=
        aws_secret_access_key=
        region: 
        ```
        
        Save above content to a text file "test.txt" on your machine, and create a kubernetes secret called "aws-secret" under the key "creds".

        ```kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=test.txt```

        Verify the secret is created by running 

        ```kubectl get secrets -n crossplane-system```

        Create a Provider Configuration.

        You can create a provider configuration by applying the below manifest to the cluster.  

        ```
        apiVersion: aws.upbound.io/v1beta1
        kind: ProviderConfig
        metadata:
          name: default
        spec:
          credentials:
            source: Secret
            secretRef:
              namespace: crossplane-system
              name: aws-secret
              key: creds
        ```

        Save above file to a file called "awsConfig.yaml" and modify the namespace, name, key.

        Congrats with the Provider configuration.! You can now start creating your managed resources. 



## Create an S3 bucket

To create an S3 bucket, make sure that the provider is installed. Look for the provider configuration for upbound by running below command.

```kubectl get providerconfig```

The READY should be True. Mention the name of the ProviderConfig in the last line of below YAML that reads "providerConfigRef"

Now you can create your first managed resource (S3 Bucket). You can save the below YAML to a file and use kubectl apply. 

```
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/external-name: hfdghjfsjkfhgksjdhgkjdfkgjffedgf
  name: basil-12345670987-dgfhf
  labels:
    managed-by: crossplane
    project: my-interal-app
spec:
  forProvider:
    region: ap-northeast-1
    tags:
      managed-by: crossplane
      project: my-interal-app
      billingContact: basil
  providerConfigRef:
    name: default
```

Congrats! You created your first managed resource. 


## Create all resources

I have created a bunch of managed resources inside the folder "managed" in this repository. You can clone this repository and use them as you like. 


```
git clone https://github.com/basil1987/crossplane-springpeople.git
```

After your ran above, you will find a folder "crossplane-springpeople". Go to that folder and go to managed.

```cd crossplane-springpeople/managed```

NOTE: All these managed resources in the "managed" folder use the provider https://marketplace.upbound.io/providers/upbound/provider-aws/v0.37.0 . So make sure that your provider and provider configurations are created already.


## Create an RDS Instance

I want to create an multi-tier infrastructure at AWS to run my application and the database. Below Diagram Demonstrate what I want.

![rds-architecture](https://github.com/basil1987/crossplane-springpeople/assets/6800901/6e4f6cb1-36d1-473a-8d16-e1e0294f6462)


I have kept all the managed resources you want to possibly created under the folder "managed/rds". Clone the repository and go to the folder.

```
git clone https://github.com/basil1987/crossplane-springpeople.git # Clone the code if you did not clone it already.
git pull # If you cloned, take the latest update.
cd crossplane-springpeople/managed/rds # Go to the folder that consist of the resources.

# Create the resources:

kubectl apply -f VPC.yaml  # Creates the VPC
kubectl apply -f subnets.yaml # Creates Subnets
kubectl apply -f internetgateway.yaml # Creates Internet Gateway
kubectl apply -f routeTablePrivate.yaml
kubectl apply -f routeTablePublic.yaml
kubectl apply -f routes.yaml
kubectl apply -f routeTableAssociationPrivate.yaml
kubectl apply -f routeTableAssociationPublic.yaml
kubectl apply -f dbSubnetGroup.yaml
kubectl apply -f securityGroupDB.yaml
kubectl apply -f securityGroupRuleDB.yaml
kubectl apply -f postgreSQLAurora.yaml
```
