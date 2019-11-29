## Fix security group rules for the task
```
### Drop the allow all rule
aws ec2 revoke-security-group-ingress --group-id ${SECURITY_GROUP} --cidr 0.0.0.0/0 --protocol all
### Add the rule to all HTTP access to the task
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP} --protocol tcp --port 80 --cidr 0.0.0.0/0
```
Repeat [step 6](demo-running-task-stage-1.md). You should still be able to access the IPv4 address via port 80 (on your browser). But, you will not be able to connect to the task using SSH since we have blocked it.

