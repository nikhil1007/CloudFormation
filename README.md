The CFN template uses aws cli command "set-instance-protection" to enable scale-in protection on EC2 instances (linux)lauched by AutoScaling group.
This feature is yet to be supported natively from CloudFormation, but is achievable via Autoscaling console or CLI. But to minimize the manual intervention 
during any architectural provisoning etc.. we can automate it using the userdata section.


