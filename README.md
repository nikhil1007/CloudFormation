CloudFormation is an infrastructure as a service provided by AWS. Although it has wide range of functionalities, it is yet to support many features which are natively available from AWS console or Command Line Interface. One such feature is the instance scale-in protection option provided by the AWS Auto Scaling Group. [1] 

Now, usually developers do not mind enabling scale-in protection directly from the console after the resource has been created as it does affect 99% of the times( although this is an out of band change i.e CloudFormation is unaware of this ). But a scenario where the autoscaling group resource needs to be replaced ( i.e if you change the AutoScaling group name etc.. ) then the instances launched as a part of new autoscaling group will not have scale-in protection enabled. Additionally going and enabling scale-in protection from AWS console or CLI is a manual task and it is good to automate wherever possible to avoid human errors. ( lets say there was a production architecture being deployed and at the end the developer forgot to enable this option by manually can cause a serious loss of data and money in the time to come )

For this purpose we can make use of UserData. It is a powerful option available in the “AWS::AutoScaling::LaunchConfiguration” resource along with a few others, which allows us to run custom commands at launch time on the EC2 instances. So leveraging this, we will run the AWS CLI command “set-instance-protection”[2] as a part of our user data which will enable scale-in protection on the instances as soon as they are launched.
The below is the CloudFormation snippet :
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash -xe
            yum install -y aws-cfn-bootstrap
            instanceID=$(curl 169.254.169.254/latest/meta-data/instance-id)
            sleep 10
            aws autoscaling set-instance-protection --instance-ids $instanceID --auto-scaling-group-name ${AutoScalingGroup} --region ${AWS::Region} --protected-from-scale-in
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For windows based EC2 instances, we can achieve it using the below command:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Fn::Base64: !Sub |
          <powershell>
          iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))
          choco install python -y
          choco install awscli -y
          $ChocoProfileValue = @'
          $ChocolateyProfile = "$env:ChocolateyInstall\helpers\chocolateyProfile.psm1"
          if (Test-Path($ChocolateyProfile)) {
            Import-Module "$ChocolateyProfile"
          }
          '@
          Set-Content -Path "$profile" -Value $ChocoProfileValue -Force
          . $profile
          refreshenv
          Start-Sleep -Seconds 15
          $instanceId = (New-Object System.Net.WebClient).DownloadString("http://169.254.169.254/latest/meta-data/instance-id”)
          aws autoscaling set-instance-protection --instance-ids $instanceID --auto-scaling-group-name ${AutoScalingGroup} --region ${AWS::Region} --protected-from-scale-in
          </powershell>
          <persist>true</persist>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


One of the first work-arounds the developers think about in such scenarios is to use of Custom Resource [2]. We can use update_auto_scaling_group() method listed in boto3 documentation [3] to enable Scale-in protection to the AutoScaling Group resource. There are pros and cons of using this method :

Pros : The stack Deletion is graceful, i.e it causes no issues to CloudFormation stack operations

Cons: The main intention to enable scale-in protection is to prevent instances from terminating due to human errors or from other services who’s behaviour we are unaware of . Here if there is a delete call on the CloudFormation stack by mistake, the instances part of the autoscaling group even though they are a protected from scale-in are terminated. 

The pros and cons of using this method:

Pros: The instance do not Delete unless they are manually deleted from console or fail to pass instance health checks ( even when there is a false delete call on the CloudFormation stack, the instance do not terminate )
    
Cons: The CloudFormation stack does not Delete gracefully. 

The CloudFormation stack fails to delete gracefully because of the following reasons :

Before the AutoScaling Group resource is deleted, its min and desired is set to 0. i.e once all the instances launched by the AutoScaling Group is terminated, CloudFormation proceeds to delete the AutoScaling Group resource.
Since we have enabled Scale-in protection the instances, it fails to terminate and stack goes into DELETE_FAILED state. We need use the skip-resources feature and proceed to delete the stack. Further navigate to the AutoScaling Group console and proceed to delete the resources.

Reference Links:

[1]Scale-in Protection : https://aws.amazon.com/blogs/aws/new-instance-protection-for-auto-scaling/ 

[2]set-instance-protection: https://docs.aws.amazon.com/cli/latest/reference/autoscaling/set-instance-protection.html



