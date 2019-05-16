CloudFormation is an infrastructure as a service provided by AWS. Although it has wide range of functionalities, it is yet to support many features which are natively available from AWS console or Command Line Interface. One such feature is the instance scale-in protection option provided by the AWS Auto Scaling Group. [1] 

Now, usually developers do not mind enabling scale-in protection directly from the console after the resource has been created as it does affect 99% of the times( although this is an out of band change i.e CloudFormation is unaware of this ). But a scenario where the autoscaling group resource needs to be replaced ( i.e if you change the AutoScaling group name etc.. ) then the instances launched as a part of new autoscaling group will not have scale-in protection enabled. Additionally going and enabling scale-in protection from AWS console or CLI is a manual task and it is good to automate wherever possible to avoid human errors. ( lets say there was a production architecture being deployed and at the end the developer forgot to enable this option by manually can cause a serious loss of data and money in the time to come )

For this purpose we can make use of UserData. It is a powerful option available in the “AWS::AutoScaling::LaunchConfiguration” resource along with a few others, which allows us to run custom commands at launch time on the EC2 instances. So leveraging this, we will run the AWS CLI command “set-instance-protection”[2] as a part of our user data which will enable scale-in protection on the instances as soon as they are launched.
The below is the CloudFormation snippet :
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  "UserData": {
          "Fn::Base64": {
            "Fn::Join": ["", [
              "#!/bin/bash -xe\n",
              "yum install -y aws-cfn-bootstrap\n",
              "instanceID=$(curl 169.254.169.254/latest/meta-data/instance-id)\n",
              "aws autoscaling set-instance-protection --instance-ids $instanceID --auto-scaling-group-name", " ",{
                "Ref": "AutoScalingGroup"
              }," ", "--protected-from-scale-in --region", " ",{
                "Ref": "AWS::Region"
              }, "\n"
            ]]
          }
        }
      }
    },
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For windows based EC2 instances, we can achieve it using the below command:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$instanceId = (New-Object System.Net.WebClient).DownloadString("http://169.254.169.254/latest/meta-data/instance-id”)
"aws autoscaling set-instance-protection --instance-ids $instanceID --auto-scaling-group-name", " ",{
                "Ref": "AutoScalingGroup"
              }," ", "--protected-from-scale-in --region", " ",{
                "Ref": "AWS::Region"
              }, "\n"

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Reference Links:

[1]Scale-in Protection : https://aws.amazon.com/blogs/aws/new-instance-protection-for-auto-scaling/ 

[2]set-instance-protection: https://docs.aws.amazon.com/cli/latest/reference/autoscaling/set-instance-protection.html



