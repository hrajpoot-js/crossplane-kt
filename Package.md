## Packaging and Distributing your compositions.

Crossplane has "Configurations" that allows you to package your compositions, xrds and the metadata to into OCI compatable images. Below diagram demonstrates a typical workflow. 



## package the application 

1) To package, you need a machine with crossplane cli installed. 

Follow the link to install Crossplane CLI on the AWS machine => https://gist.github.com/jonashackt/2da4dd06798fa391eda1a3fc299490c3

```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
```

You can verify the installation by running the command "kubectl crossplane".


2) Package your configurations into an OCI-compatible image. The folder "packages/my-dynamo-bucket" consist of a sample configuration ready to be packaged.

```
git clone https://github.com/basil1987/crossplane-springpeople.git
git pull # if you already cloned.
cd packages/my-dynamo-bucket
ls
kubectl crossplane build configuration # Build the package
```

Once the package is ready you will be able to see it (file that ends with .xpkg) in your current folder. 

```
ubuntu@ip-172-31-35-216:~/crossplane-springpeople/packages/my-dynamo-bucket$ ls
composition.yaml  crossplane.yaml  definition.yaml  my-dynamo-bucket-e7750a56771c.xpkg
```

3) Push to the oci image to a registry (Docker Hub / ECR) to share and distribute. 

```
kubectl crossplane push configuration YOUR_DOCKERID/my-dynamo-bucket:latest
```

NOTE: You must login to the registry by using "docker login" and providing your credentials at Docker Hub. If you are using ECR, go to the ECR page and find the commands there to login. 


## Install the Package a Kubernetes Cluster that has crossplane installed. 

You can insatll the package on any Kubernetes (k3d, eks) by running below command. Make sure that crossplane cli is installed first.

```
kubectl crossplane install configuration basilvarghese/my-dynamo-bucket:latest 
```

Replace "basilvarghese/my-dynamo-bucket:latest" with your own oci image ID that you pushed in the last step. 

Alternatively you can also create a cnfiguration by defining it as a configuration object of pkg.crossplane.io/v1 . Create a YAML file with below content and apply it with kubectl.


```
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: my-org-infra
spec:
  package: basilvarghese/my-dynamo-bucket:latest
  packagePullPolicy: IfNotPresent
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
  PackagePullSecrets: YOUR-SECRET
```


NOTE: If you are using a private registry to store your packages, your kubernetes cluster wont be able to pull the packages unless a "PackagePullSecrets" field is mentioned. 



## Verify the installation 

You can verify that configuration got created using below command. 

```
kubectl get configurations
```

Wait until status becomes READY=true .

make sure that compositions and xrd exists in the cluster. 

```
kubectl get compositions
kubectl get xrd 
```

The above commands should display the composition and xrd that you packaged. 



## Make a claim 

You can make a claim by applying below YAML using kubectl apply.

Please note that Claims are namespaced. So create the claim in the namespace of your chouce (eg: dev) when you apply.


```
apiVersion: custom-api.example.org/v1alpha1
kind: custom-database
metadata:
  name: claimed-eu-database-basil
  namespace: dev
spec:
  region: "AP"
```

Copy above content to a file called Claim.yaml and run 

```
kubectl create namespace dev
kubectl apply -f Claim.yaml --namespace dev
```

After you created the claim, verify that it got created by running. 

```
kubectl get custom-database.custom-api.example.org
```

Verify the XR got created by running

```
kubectl get composite
```

make sure that it has a status of Synced=True and READY=true.


If READY is False, run the describe command and look under the "spec.Resource Refs". You should see all the managed resources reference here. Make sure that MRs are present there. 


Now check the status of the MRs by running below command. 

```
kubectl get managed
```

You will see an S3 Bucket as well as a Dynamo table. Both resources must have READY=true.

If READY=False, run the kubectl describe command against each managed resource which false status. 



## SOMETHING WENT WRONG 

For some reason, the table was created but bucket was NOT. While executing describe comamnd againts the XR, I noticed that there is a warning that talks about including sme field spec.atProvider.region into the composite resource. 

I also recently got few requests that some Dynamo tables should have accidental protection enabled. 

So I am going to create a new package that will have following bug fixed.

1) Bucket Issues

And Following feature enhancement. 

1) Ability for the developers to protect their tables from accidental protetion. Its purely an optional choice.


#### Implementing above changes, 

I need to modify both xrd (defenition.yaml) and composition (composition.yaml) to fix this.

1) (definition.yaml). 

```
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: databases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: database
    plural: databases
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string
                oneOf:
                  - pattern: '^EU$'
                  - pattern: '^US$'
                  - pattern: '^AP$'
              enableAccidentalTermination:     ### added  enableAccidentalTermination to the schema
                type: boolean                  ###
            required:
              - region
  claimNames:
    kind: custom-database
    plural: custom-databases
```

2) (composition.yaml)

```
    - name: dynamoDB
      base:
        apiVersion: dynamodb.aws.upbound.io/v1beta1
        kind: Table
        metadata:
          name: crossplane-quickstart-database
        spec:
          forProvider:
            deletionProtectionEnabled: False                                   ## This becomes true if the developer set enableAccidentalTermination
            writeCapacity: 1
            readCapacity: 1
            attribute:
              - name: S3ID
                type: S
            hashKey: S3ID
      patches:
        - fromFieldPath: "spec.enableAccidentalTermination"                     ## Patch the deletionProtectionEnabled with developer provded value
          toFieldPath: "spec.forProvider.deletionProtectionEnabled"             ## 
```



#### Packaging and Pushing the changes.


Repeat the same instructions we used earlier to create the package. You have to go to the folder "packages/my-dynamo-bucket" and run below command


```
cd packages/my-dynamo-bucket
rm *.xpkg  ## to delete old files
kubectl crossplane build configuration ## To build new package
kubectl crossplane push configuration basilvarghese/my-dynamo-bucket:1.0.1
```


#### Installing the new package in Crossplane 

When you install a new version of the configuration package, the crossplane will not delete the old version. It created new revision and makes the latest revision "Active". The old one become "InActive" . This behaviour can be modified by setting "revisionActivationPolicy" field. 

To install the package 1.0.1, use below YAML file and apply it with kubectl apply .


```
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: basilvarghese-my-dynamo-bucket
spec:
  package: basilvarghese/my-dynamo-bucket:1.0.1
  packagePullPolicy: IfNotPresent
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

Replace the package basilvarghese/my-dynamo-bucket:1.0.1 with your own package. 



#### Verify the new installation 

Run the command below to verify that READY=true 

```
kubectl get configurations
```

Now, check which one is the Active revision. The new one should be active by default.

```
kubectl get configurationrevisions
```

When the new new revision become active, the current XRs will start using the new revision. Which also means, the error should be gone, and bucket should be created. 

Verify that your XR is showing READY=true

```
kubectl get composite
```

TADA!! Bucket is now created. 


Also, my developer now will have the ability to protect their tables. To verify, modify your claim which look like below.


```
apiVersion: custom-api.example.org/v1alpha1
kind: custom-database
metadata:
  name: claimed-eu-database-basil
  namespace: dev
spec:
  region: "AP"
  enableAccidentalTermination: True     ## This field enable protection for the developer table
```

Save above file and run kubectl apply to make the changes. 

After a couple of seconds, check the managed resource curresponding to the bucket. Look for the field status.atProvider.deletionProtectionEnabled. It should be True. 

```
kubectl get tables.dynamodb.aws.upbound.io
kubectl describe <YOUR_TABLE_PREVIOUS_COMMAND>
```

