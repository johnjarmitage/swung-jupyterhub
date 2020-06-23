# Creating a k8s cluster on AWS to host jupyter-lab

I am going to use `eksctl` to create the cluster using my AWS credentials, and `kubectl` to manage the cluster once it is made. The details are [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html). The firs step is to get get the `awscli` and give it my credentials with the `aws configure`command.

Once this is done create a micro cluster with the command:
```commandline
eksctl create cluster -f create-cluster/micro-cluster.yml
```
This will create a CloudFormation stack where first the VPC is created and then the resources, in this case a micro instance is created. You can go watch its progress in the CloudFormation tab of the AWS web interface, or you can go have a coffee. Once the cluster is created I like to check in EC2 Dashboard just to see that the right EC2 is there and verify that I am using the resources I wanted.

The cluster can be removed with:
```commandline
eksctl delete cluster -f create-cluster/micro-cluster.yml
```

The YAML file for creating the k8s cluster is below:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-micro
  region: eu-west-1

nodeGroups:
  - name: micro-k8s
    instanceType: t2.micro
    desiredCapacity: 2
```
There are more options, but this is the basics. I have selected that the cluster is created in Ireland (`eu-west-1`) where it is cheap. The EC2 instance type is a t2 micro, and there are 2 nodes (see [eksctl.io](https://eksctl.io/usage/creating-and-managing-clusters/) for more options especialy auto-scaling).

Once `eksctl` exits check the cluster is created with:
```commandline
kubectl get svc
```
If all is correct it will return:
```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   11m
```

The cluster can now be modified with `kubectl`. It is good to check that this is case with a command such as `kubectl get nodes` to get information on the nodes. 

# Creating and uploading the docker image onto ECR

The first step is to create a repository within the Amazon Elastic Container Registry service. Navigate to the ECR and click create a repository. I will call this repository `notebook-library` in line with the Docker image I will push to the ECR.

For now I will use the Dockerfile within [this](https://github.com/johnjarmitage/notebook-library) repository. The first step is to build the image:
```commandline
docker build -t notebook-library:1.0 .
```
Now I will tag it so it can be exported to the AWS ECR service:
```commandline
docker tag notebook-library:1.0 ****.dkr.ecr.eu-west-1.amazonaws.com/notebook-library:1.0
```
(**** is your AWS account number *not the alias*)

Docker however does not yet know about AWS. I will use the `awscli` to get the credentials of the ECR to give to Docker. This is done by passing the output of command below into the terminal.
```commandline
aws ecr get-login --region eu-west-1 --no-include-email
```
This means we can now put the Docker image into the ECR:
```commandline
docker push ****.dkr.ecr.eu-west-1.amazonaws.com/notebook-library:1.0
```
For me this took quite some time given the size of the Docker image. It will be interesting to see if I can fit it on a k8s cluster running on a micro instance.

# Deploying the k8s cluster

I will use kubectl to deploy the cluster. The deployment is within a YAML file. The Jupyter-lab or notebook will require a port to be exposed on the k8s cluster. This would normally be port 8888. (I will also need the access key to the notebook, but this is a problem for later). First lets just see if the Docker image works.

The first task is to create a namespace *library* for the pod.
```commandline
kubectl create ns library
```

The Jupyter-lab or notebook will need to talk to the outside world, so I will create a load balancer to give it access to the outside via AWS. This is created with the YAML file `deploy-cluster/loadbalancer-notebook-library.yml`. The pod containing the container with the notebook is deployed with this YAML: `deploy-cluster/deploy-notebook-library.yml`.
First the deployment:
```commandline
kubectl -n library apply -f deploy-cluster/deploy-notebook-library.yml
```

```commandline
kubectl -n library apply -f deploy-cluster/loadbalancer-notebook-library.yml
```

To access the notebook I need to get the key, so first I get the pod name,
```commandline
kubectl -n library get pods
```
and then look a the log of the pod to get the key,
```commandline
kubectl -n library logs <name of pod>
```
To navigate to the notebook I use the address of loadbalancer, the port specified followed by the key from the log file. This is not ideal for running as a hub for other users, but as a proof of concept it looks OK.

*Unfortunately (and not surprisingly) a micro instance is too small. But as a proof of concept this works.*

# Using `helm` to deploy a jupyter-hub

I am going to follow [this](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub/setup-jupyterhub.html) guide mostly to set up the deployment using `helm` to orchestrate `kubectl`. The advantage here is that those instructions set up a key to get into the Jupyter lab or notebook without looking at the log file of the pod. 

First I deploy a larger cluster `t3.large`:
```commandline
eksctl create cluster -f create-cluster/t3large-cluster.yml
```
Then as described I want to create a hex key, so I follow the [guide](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub/setup-jupyterhub.html) to create the key. I also want to use my docker image and have the Jupyter-lab interface as default. So I create the `config.yml` file as described below (see [here](https://zero-to-jupyterhub.readthedocs.io/en/latest/customizing/user-environment.html#choose-and-use-an-existing-docker-image)):
```yaml
# some choices to customize the user environment

proxy:
  secretToken: "pasted from `openssl rand -hex 32`"
singleuser:
  defaultUrl: "/lab"
singleuser:
  image:
    name: ****.dkr.ecr.eu-west-1.amazonaws.com/notebook-library
    tag: 1.0
```
where `****` is the AWS account ID. Helm uses these values to then deploy a k8s cluster. 

Check the cluster exists:
```commandline
kubectl get svc
```
Create the namespace for the deployment:
```commandline
kubectl create ns library
```
Run the `helm` deployment:
```commandline
helm upgrade --install jhub jupyterhub/jupyterhub --namespace library \
  --version=0.9.0 --values config.yml
```
where `jhub` is the release name for the deployment within the namespace `library`. To list this deployment you need to specify the namespace:
```commandline
helm list --namespace library
```
To remove the deployment you likewise need to specify the namespace:
```commandline
helm delete jhub --namespace library
```