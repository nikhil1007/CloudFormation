
# Upgrading Aurora DB Cluster's Engine Version via CloudFormation without Replacing the Existing DB Cluster

Minor versions patch security vulnerabilities, fix bugs, and generally do not add new functionality. Minor releases never change the internal storage format and are always compatible with earlier and later minor releases of the same major version number. But even such small upgrade needs resource replacement if done via CloudFormation. This could obviously be frustrating as replacement of a DB cluster could take ample time and that directly increases the downtime.

## How to upgrade the Aurora DB cluster's Engine Version without replacement ? ##

Precaution before we proceed:
```
1. Stop connections to the DB cluster so that there's no activity on the DB cluster.
2. Create a manual snapshot of the DB cluster.
```

**Step 1**: Add Deletion policy attribute with value "Retain" and update the CloudFormation stack.
We need to add the deletion policy attribute to AWS::RDS::DBCluster, AWS::RDS::DBInstance and all other resources in which reference (aka !Ref) the DB cluster or DB instance in their definition if any.

Note: Please see the sample template initial.yaml to know how my sample template looks after adding deletion policy.

**Step 2**: Remove all the resources to which deletion policy was added and Update the Cloudformation stack.
Since the value of Deletion policy is "Retain", the resources will be taken out of the CloudFormation lifecycle in the update process, but it will be retained in the AWS account as an independent resource.

Note: Please see the sample template ResourceRemoved.yaml to know how my sample template looks after completing step.

**Step 3**: Perform manual DB Engine version upgrade from console


**Step 4**. Add back all the removed resource definitions (in step 2) back to the CloudFormation template. 
Make sure to have the upgraded Engine version here ( i.e if you have upgraded the engine version from 10.7 to 10.11 in step 3, then have the template also reflect the Engine version as 10.11)

**Step 5**: Importing all the resources into CloudFormation's lifecycle
Navigate to the CloudFormation console, select your stack > click on Stack Actions > "Import Resources into your stack" > Upload your template. 
Further you will be asked to input your DB cluster identifier, instance identifier etc. which you can retrieved from their respective Service console and proceed further to import.

At the end of the 5th step you should have your Aurora DB cluster and instances back under the CloudFormation's Lifecycle with upgraded Engine Version. 
