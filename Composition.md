#### Composite Resources(XR). 

An XR is a resource that you wants to create to get the opinionated resource you need for doing something. 

Below example shows an XR which when you submit, Crossplane will use its machinery to do whatever is necessary get the resource created and ready


```
apiVersion: springexample.io/v1alpha1
kind: XObjectStorage
metadata:
  name: test-bucket-awsblueprint-123456789
spec:
  resourceConfig:
    region: us-east-1
```

An XR schema is defined in another resource called XRD. It basically provider a custom API Group, Kind, and the schema of the XR. 


#### Composite Resources Definition (XRD)

XRD kind of looks like a CRD. A Sample reference is given below which defines schema for above XR. 


```
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0

apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xobjectstorages.springexample.io
spec:
  claimNames:
    kind: ObjectStorage
    plural: objectstorages
  group: springexample.io
  names:
    kind: XObjectStorage
    plural: xobjectstorages
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          properties:
            spec:
              description: ObjectStorageSpec defines the desired state of ObjectStorage
              properties:
                resourceConfig:
                  description: ResourceConfig defines general properties of this AWS
                    resource.
                  properties:
                    name:
                      description: Set the name of this resource in AWS to the value
                        provided by this field.
                      type: string
                    region:
                      type: string
                    tags:
                      items:
                        properties:
                          key:
                            type: string
                          value:
                            type: string
                        required:
                        - key
                        - value
                        type: object
                      type: array
                  required:
                  - region
                  type: object
              required:
              - resourceConfig
              type: object
            status:
              description: ObjectStorageStatus defines the observed state of ObjectStorage
              properties:
                bucketName:
                  type: string
                arn:
                  type: string
              type: object
          type: object
```


#### Composition 

The composition tells what to do when an XR is created. This may mean creating a number of managed resources (MR),  patching some fields, setting up default values. Composition refer MR. 

The sample reference is below. 


```
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0

apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3bucket.springexample.io
  labels:
    springexample.io/provider: aws
    springexample.io/environment: dev
    s3.springexample.io/configuration: standard
spec:
  compositeTypeRef:
    apiVersion: springexample.io/v1alpha1
    kind: XObjectStorage
  resources:
    - name: s3-bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          deletionPolicy: Delete
          forProvider:
            region: ap-south-1
            forceDestroy: true # be careful with this
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.resourceConfig.region
          toFieldPath: spec.forProvider.region
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.id
          toFieldPath: status.bucketName
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.arn
          toFieldPath: status.arn
```



#### Creating an XR 

The folder composite/s3-simple/composition consist of the XRD and Composition objects. You can create them in Kubernetes with kubectl apply. 

```
git clone https://github.com/basil1987/crossplane-springpeople.git
cd crossplane-springpeople/composite/s3-simple/composition
kubectl apply -f definition.yaml
kubectl apply -f composition.yaml
```

Make sure that they are ready. Below commands should return READY=true

```
kubectl get xrds 
kubectl get compositions
```


Now you can create the XR by applying the file composite/s3-simple/claim/s3.yaml

```
cd
cd composite/s3-simple/claim
kubectl apply -f s3.yaml
```


#### Verify everything. 

1) Make sure that your XR is retuning the status READY=true 

```
kubectl get composite
```

2) The Bucket was created in the right region mentioned in the claim. For that, you can check the MR

```
kubectl describe buckets.s3.aws.upbound.io 
```

You can look into the status field which contains region information, ARN, ID

3) Your XR copied the status updates such as ARN and ID.

```
kubectl get composite
kubectl describe composite COMPOSITE_NAME_FROM_PREVIOUS_COMMAND
```

You will see ARN and BucketName updated in the status field. 


## Another use case

I want a bucket and the bucket will be used for storing some sensitive data by my applications. So I want a bucket to be created and managed by crossplane. 

1) I want new bucket to be created if it does not exists.
2) I want the bucket to be enabled with server side encryption
3) I want the bucket to be protected from public
4) I want the bucket name to be written inside a kubernetes configmap



#### Create the XRD and composition

The project is located in the folder "crossplane-springpeople/composite/s3" 

"composition" folder has both XRD and composition definitions. Apply them with kubectl

```
git clone https://github.com/basil1987/crossplane-springpeople.git
cd crossplane-springpeople/composite/s3/composition
kubectl apply -f definition.yaml
kubectl apply -f composition.yaml
```

"Claim" folder has s3.yaml, apply with kubectl 

```
cd
cd crossplane-springpeople/composite/s3/claim
kubectl apply -f s3.yaml
```

