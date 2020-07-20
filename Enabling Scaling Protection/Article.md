Developers do not mind enabling scale-in protection directly from the console after the Autoscaling Group resource has been created. This does not affect the normal stack operations 99% of the times( although this is an out of band change i.e CloudFormation is unaware of this ). In scenarios where the AutoScaling Group resource needs to be replaced ( i.e if you change the AutoScaling group name ) then the instances launched as a part of new AutoScaling Group will not have scale-in protection enabled. Additionally enabling scale-in protection from AWS console or CLI is a manual task and it is good to automate wherever possible to avoid human errors. ( let us assume there was a production architecture being deployed and at the end the developer forgot to enable this option manually can cause a serious loss of data and money in the time to come )

Why do we need Scale-in Protection enabled on the instance ?
Usually we enable scale-in protection on those instances which might be handling a long-running work task, perhaps pulled from an SQS queue. Protecting the instance from termination will avoid wasted work. Second, the instance might serve a special purpose within the group. It could be the master node of a Hadoop cluster, or a “canary” that flags the entire group of instances as up and running."
One of the first work-arounds the developers think about in such scenarios is to use of Custom Resource [2]. We can use update_auto_scaling_group() method listed in boto3 documentation [3] to enable Scale-in protection to the AutoScaling Group resource. There are pros and cons of using this method :
Pros : The stack Deletion is graceful, i.e it causes no issues to CloudFormation stack operations
Cons: The main intention to enable scale-in protection is to prevent instances from terminating due to human errors or from other services who’s behaviour we are unaware of . Here if there is a delete call on the CloudFormation stack by mistake, the instances part of the autoscaling group even though they are a protected from scale-in are terminated.
AWS::AutoScaling::LaunchConfiguration” resource along with a few others allows us to run custom commands at launch time on the EC2 instances. So leveraging this, we will run the AWS CLI command “set-instance-protection”[4] as a part of our user data which will enable scale-in protection on the instances as soon as they are launched.For this purpose we can make use of UserData. It is a powerful option available in the “

Here we set scale-in protection at the instance level rather than the autoscaling group level ( as set by Custom Resource ). It is to be noted that it is not feasible to set scale-in protection at instance level using custom resource as they run during CloudFormation Create/Update/Delete operations and do not take effect if the AutoScaling Group scales normally. More information on this can be seen in the Link [5] 

The below is the CloudFormation snippet : (Amazon-linux)
```
 UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash -xe
            yum install -y aws-cfn-bootstrap
            instanceID=$(curl 169.254.169.254/latest/meta-data/instance-id)
            sleep 10
            aws autoscaling set-instance-protection --instance-ids $instanceID --auto-scaling-group-name ${AutoScalingGroup} --region ${AWS::Region} --protected-from-scale-in
```

Here, we curl the instance metadata to retrieve the instance ID
Use the instance ID in the set-instance-protection AWS CLI command as shown above.
Similarly for windows ami below is the user-data used:

```
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
```

* We install python and awscli using chocolatey ( feel free to install it as per convenience (for e.g. it can be done done using msi installer as well ) )
* Further for the installed packages to take effect we need to restart the shell (i.e powershell here ). So I have manually edited the powershell profile.
* We get the instance id and then run the set-instance-protection AWS CLI command to set scale-in protection for the instance.

Note: I have added sleep commands in the respective user-data as the script was intermittently failing sometimes due to package installation issues. The value provided to the sleep commands have been decided after a few trial and error attempts at my end.

Below are the steps we can follow to test the requirement :

* Depending on the underlying OS (i.e windows or amazon linux) choose the attached CloudFormation template and proceed to deploy it.
* Specify the parameters accordingly ( Description provided for each parameter should be of help here )
* Launch the stack

Expected results :

* On successful completion of the CloudFormation stack, navigate to the AutoScaling Group console ( Please feel free to use the resources section to navigate directly )
* Select the concerned Autoscaling Group and click on the ‘instances’ tab
* We should be able to see the instances under the AutoScaling Group are protected from Scaling. ( Please refer the screen shot attached )

Note: In case of windows instances please wait for the instances to pass the status health checks for the scale-in protection to reflect on the console.

The pros and cons of using this method:

* Pros: The instance do not Delete unless they are manually deleted from console or fail to pass instance health checks [6] ( even when there is a false delete call on the CloudFormation stack, the instance do not terminate )
* Cons: The CloudFormation stack does not Delete gracefully.

The CloudFormation stack fails to delete gracefully because of the following reasons :

* Before the AutoScaling Group resource is deleted, its min and desired is set to 0. i.e once all the instances launched by the AutoScaling Group is terminated, CloudFormation proceeds to delete the AutoScaling Group resource.
* Since we have enabled Scale-in protection the instances, it fails to terminate and stack goes into DELETE_FAILED state. We need use the skip-resources feature and proceed to delete the stack. Further navigate to the AutoScaling Group console and proceed to delete the resources.
