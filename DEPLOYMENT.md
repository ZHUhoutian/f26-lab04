# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

<!-- The ServiceUrl and InstanceId outputs. Paste both here every time
describe-stacks prints them, for the healthy deploy and for scenario 2. Both
change on every recreate, and you will need them for curls and sessions. -->

### Healthy deploy (milestone 1)

```
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-07637e3b604c089f7                                    |
|  ServiceUrl|  http://ec2-3-94-195-247.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+
```

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-3-94-195-247.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

Three or four sentences, your own words. What compute, what network access, and what
glue made the service start.

For compute, the template created a single `t3.micro` EC2 instance whose AMI is not
hardcoded — the `AmiId` parameter is a public SSM parameter that resolves to the
current Amazon Linux 2023 image — and attached the `LabInstanceProfile` IAM profile
(which carries the SSM permissions for opening a session without a key pair) plus the
`vockey` key pair as an SSH fallback. For network access, it created a security group
that opens exactly two inbound ports to `0.0.0.0/0`: TCP 8080, the `ServicePort` my
external health check hits, and TCP 22 for the SSH fallback; outbound is left at the
EC2 default of all traffic allowed, which is what lets the instance install Docker and
pull the image. The glue is the `UserData` shell script, which runs once at first boot
to install Docker, enable it, add `ec2-user` to the docker group, set a `shutdown -h
+240` guard so an abandoned instance stops burning credits, and finally `docker run`
the course image with `--restart unless-stopped` and `-p 8080:8080`. Because that
script runs after CloudFormation already reports CREATE_COMPLETE, there is a window of
a minute or two where the stack exists but a curl still fails to connect.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

```

```

**The log line that told you what was wrong:**

```

```

**What was wrong, and the fix you applied:**

<!-- One or two sentences. Say what you changed and where you changed it. -->

**The healthy curl after the fix:**

```

```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

```

```
