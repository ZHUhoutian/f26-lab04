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

### Scenario 2 deploy (milestone 2, broken)

```
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
----------------------------------------------------------------------
|                           DescribeStacks                           |
+------------+-------------------------------------------------------+
|  InstanceId|  i-0f2f53b5ff736b54e                                  |
|  ServiceUrl|  http://ec2-3-95-11-72.compute-1.amazonaws.com:8080   |
+------------+-------------------------------------------------------+
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
`vockey` key pair as an SSH fallback. 

For network access, it created a security group
that opens exactly two inbound ports to `0.0.0.0/0`: TCP 8080, the `ServicePort` my
external health check hits, and TCP 22 for the SSH fallback; outbound is left at the
EC2 default of all traffic allowed, which is what lets the instance install Docker and
pull the image.

 The glue is the `UserData` shell script, which runs once at first boot
to install Docker, enable it, add `ec2-user` to the docker group, set a `shutdown -h
+240` guard so an abandoned instance stops burning credits, and finally `docker run`
the course image with `--restart unless-stopped` and `-p 8080:8080`. Because that
script runs after CloudFormation already reports CREATE_COMPLETE, there is a window of
a minute or two where the stack exists but a curl still fails to connect.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

Repeated over several minutes, well past the install-and-pull warm-up window, so the
failure is persistent rather than a too-early curl:

```
$ curl http://ec2-3-95-11-72.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-3-95-11-72.compute-1.amazonaws.com port 8080 after 95 ms: Couldn't connect to server
$ curl http://ec2-3-95-11-72.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-3-95-11-72.compute-1.amazonaws.com port 8080 after 582 ms: Couldn't connect to server
$ curl http://ec2-3-95-11-72.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-3-95-11-72.compute-1.amazonaws.com port 8080 after 27 ms: Couldn't connect to server
```

**The log line that told you what was wrong:**

Taken from an SSM session on the scenario 2 instance (`aws ssm start-session --target
i-0f2f53b5ff736b54e`). The container is up and the host mapping is fine, so the two
commands have to be read side by side:

```
sh-5.2$ sudo docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
877610fd3974   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   6 minutes ago   Up 6 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service
sh-5.2$ sudo docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

<!-- One or two sentences. Say what you changed and where you changed it. -->
docker ps shows the host forwarding 8080->8080, but docker logs shows the process inside bound 9090, so nothing was listening on the container's 8080 and the connection had nowhere to land. params-scenario2.json sets PortOverride to 9090, which the template feeds into the container's PORT; I fixed it by deleting the stack and recreating it with params-healthy.json, where PortOverride is empty so the container binds 8080.

**The healthy curl after the fix:**

The broken stack was deleted and recreated with `params-healthy.json`, which produced a
third instance (`i-04e775ecfbef03264`) with a new public DNS name. Curling immediately
after CREATE_COMPLETE still fails, because the UserData script is only then installing
Docker and pulling the image; once that finishes, the same URL answers:

```
$ curl http://ec2-100-53-185-27.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

```

```
