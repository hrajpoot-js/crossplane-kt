## Final Project


In our company, we use Amazon Aurora database and we want to build a platform for the developers to provision and manage databases in the cloud themselves. 

We have

1) QA people
2) Developer who want to do some development and preview their application
3) Performance testing team who validates the database transactions againts to see how databases are performining
4) Platform engineer who manages pre-production and production databases.

I want to build and reuse a platform API which all above team / people can consule. 

In any case except production, I need team to get a dedicated uninterrupted infrastructire whenever they want. 

The user will have the ability to chose 
1) Size of the database in GB.
2) Request the RAM in GBs. If requested any size between 2 to 4, I want to give them a t2.db.medium instance. Its 5 to 8, t2.db.large. If its 9 to 15, I want to give them a t2.db.xlarge instance.
3) Ability to specify the location where they want the database to be physically from a set of pre-defined values (Mumbai, Tokyo).
4) Kind of Disk the need can be selected from 3 options - slow, moderate, high.



### Work to do:

1) Create XRD. XRD defines possible fields my developers can choose.

2) Create a composition, which will have a VPC, 3 private Subnets, 1 public subject (where application runs), a subnetgroup, db security group with permission given to the public IP.

3) I want to build it as a package and store in AWS ECR (private / public).

4) Crossplane clusters can download and install the packages.


