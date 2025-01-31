![](https://codebuild.us-west-2.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoiVW85THhEZXRnd3FxUmM5QnRaS1cyYjNFUktvVkFJSkVONlpEZGV2WHg5ZzRiNjdqRUIrVFBmWWdacXdZYll3TzU5N3NxR1h3MDRPZ0YwajhUT3gzbnI0PSIsIml2UGFyYW1ldGVyU3BlYyI6ImpqN0w4MTc1TmozVmI4c24iLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=master)
[![Build Status](https://travis-ci.org/pahud/eks-kubectl-docker.svg?branch=master)](https://travis-ci.org/pahud/eks-kubectl-docker)
[![GitHub release](https://img.shields.io/github/release/pahud/eks-kubectl-docker.svg?style=plastic)](https://github.com/pahud/eks-kubectl-docker/releases)
![docker image size](https://shields.beevelop.com/docker/image/image-size/pahud/eks-kubectl-docker/latest.svg?style=plastic)
![image layers](https://shields.beevelop.com/docker/image/layers/pahud/eks-kubectl-docker/latest.svg?style=plastic)
![image pulls](https://shields.beevelop.com/docker/pulls/pahud/eks-kubectl-docker.svg?style=plastic)


# eks-kubectl-docker
**eks-kubectl-docker** is a docker image with `kubectl` CLI and Amazon EKS authentication support that helps you directly interact with Amazon EKS control plane in a Docker image. You may run it locally or in any managed container services such as `AWS Fargate` or `AWS CodeBuild`.

# Usage

```bash
# get pod list from eksdemo1 cluster
$ docker run -v $HOME/.aws:/home/kubectl/.aws \
-e AWS_REGION=us-west-2 \
-e CLUSTER_NAME=eksdemo1 \
-ti pahud/eks-kubectl-docker:latest \
kubectl get po 

[INFO] region=us-west-2
Added new context arn:aws:eks:us-west-2:{AWS_ID}:cluster/eksdemo1 to /home/kubectl/.kube/kubeconfig
NAME                        READY     STATUS    RESTARTS   AGE
greeting-58cb8c7dfc-j5dqq   1/1       Running   0          3d
greeting-58cb8c7dfc-nvfcd   1/1       Running   0          3d
nginx-7d86684c7c-jwqrt      1/1       Running   0          3d
nginx-7d86684c7c-z99sk      1/1       Running   0          3d

# or just make get-nodes to get the node list
# edit the Makefile before you run the make command
# you may optionally specify a --role-arn in the Makefile
$ make get-nodes
[INFO] region=us-west-2
[INFO] got EKS_ROLE_ARN=arn:aws:iam::{AWS_ID}:role/AmazonEKSAdminRole, updating kubeconfig with this role
Added new context arn:aws:eks:us-west-2:{AWS_ID}:cluster/eksdemo3 to /home/kubectl/.kube/kubeconfig
NAME                                           STATUS   ROLES    AGE   VERSION
ip-100-64-126-218.us-west-2.compute.internal   Ready    <none>   19h   v1.12.7
ip-100-64-173-2.us-west-2.compute.internal     Ready    <none>   19h   v1.12.7
```



# CodeBuild support

You can use `pahud/eks-kubectl-docker` as the custom image for your CodeBuild environment.



## kubectl get pod

Create an IAM Role for CodeBuild with a custom policies

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "arn:aws:eks:*:*:cluster/*"
        }
    ]
}
```



edit `aws-auth` ConfigMap and add the created role Arn in the `system:masters` group

```
kubectl edit -n kube-system cm/aws-auth
```

![](images/01.png)



In CodeBuild, create a project and specify `pahud/eks-kubectl-docker` as the custom image.

![](images/02.png)

Specify the IAM Service role you just created in the previous step

![](images/03.png)



Create `buildspec.yaml` or just specify `/root/bin/entrypoint.sh kubectl get po` as the build command



**Insert build commands** and click **Switch to editor**, enter the build commands as below:

![](images/04.png)



Specify the required environment variables. In this case, we specify our Amazon EKS cluster name as `eksdemo`. Set your correct Amazon EKS cluster name here.

![](images/05.png)



start the build and see the build logs. Check the output of your `kubectl get po` commands.

![](images/06.png)



## kubectl apply or create

You may also let CodeBuild work with the `buildspec.yml` in your github repository. Check [buildspec.yml](./buildspec.yml) in the root repository for your reference. In this sample, we `kubectl apply` a nginx deployment as well as service within CodeBuild environment.

![](images/08.png)

And you'll see the deployment, replicaset, service as well as the pods are all created and running.

![](images/09.png)

## Docker in Docker support

**pahud/eks-kubectl-docker** has docker in docker support, which means you can docker build|pull|tag|push in the docker container and optionally push to Amazon ECR or other git repository. Behind the scene, when you bring up **pahud/eks-kubectl-docker** with **CODEBUILD_BUILD_ID** environment variable available, which is by default available in AWS CodeBuild, it will start the dockerd for you.



Make sure you flag **Privileged** in your CodeBuild Environment setting and specify **pahud/eks-kubectl-docker:latest** as your custom image.

![](images/07.png)

# AWS Fargate Support
TBD

# AWS Fargate with CloudWatch Event scheduled events
TBD

# Automated Amazon EKS provisioning and custom validation with AWS CodeBuild
You can automate the Amazon EKS workload provisioning and service state validation by CodeBuild or triggered by Cloudwatch Events periodically or triggered by Github push/PR. Check this sample [buildspec.yml](https://github.com/pahud/eks-kubectl-docker/blob/master/samples/codebuild/service-validation/buildspec.yml) that creates a fresh new temporary Amazon EKS cluster to validate the `aws-vpc-cni` driver version and eventually delete the cluster.

# FAQ

Q: Do I need to specify **REGION**  in my CodeBuild environment variables?

A: No. It will determine the running region from the built-in CodeBuild environment varialble **CODEBUILD_AGENT_ENV_CODEBUILD_REGION**, however, if you specify **REGION**, it will override **CODEBUILD_AGENT_ENV_CODEBUILD_REGION**.

