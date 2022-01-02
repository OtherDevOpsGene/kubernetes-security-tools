# kubernetes-security-tools

Notes for doing the demos for "Keeping your Kubernetes Cluster Secure".

## Set up a target cluster

### Update eksctl

```console
$ eksctl version
0.67.0
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin
$ eksctl version
0.77.0
```

### User with sufficient permissions

Per the `eksctl` [minimum IAM policies](https://eksctl.io/usage/minimum-iam-policies/),
the AWS user needs:

* AmazonEC2FullAccess
* AWSCloudFormationFullAccess
* `eksctl-iam-policy.json` (replace `054858005475` with the account number)

Just having those caused an error:

```
Error: unable to determine AMI to use: error getting AMI from SSM Parameter Store: AccessDeniedException: User: arn:aws:iam::054858005475:user/gene.gotimer-eks is not authorized to perform: ssm:GetParameter on resource: arn:aws:ssm:us-east-2::parameter/aws/service/eks/optimized-ami/1.20/amazon-linux-2/recommended/image_id
        status code: 400, request id: 2a4a86a1-0053-4ef1-bcb7-16b8b256e8d0. please verify that AMI Family is supported
```

so I added `AmazonSSMFullAccess`, even though the [IAM Policy Simulator](https://policysim.aws.amazon.com/home/index.jsp?#)
said I shouldn't need it.

```console
$ aws sts get-caller-identity
{
    "UserId": "AIDAQZROKXPRRQ27Y67ME",
    "Account": "054858005475",
    "Arn": "arn:aws:iam::054858005475:user/gene.gotimer-eks"
}
```

### Stand up the cluster and a nodegroup

The cluster and nodegroup take about 15 minutes and 3 minutes, respectively.

```console
$ eksctl create cluster -f cluster.yaml
2022-01-02 16:53:42 [ℹ]  eksctl version 0.77.0
2022-01-02 16:53:42 [ℹ]  using region us-east-2
...
2022-01-02 17:07:42 [✔]  all EKS cluster resources for "codemash" have been created
2022-01-02 17:07:46 [ℹ]  kubectl command should work with "/home/ggotimer/.kube/config", try 'kubectl get nodes'
2022-01-02 17:07:46 [✔]  EKS cluster "codemash" in "us-east-2" region is ready
$ eksctl create nodegroup --cluster=codemash --nodes-min=2 --nodes-max=5 --instance-types=m5.xlarge codemash-ng-1
2022-01-02 17:11:50 [ℹ]  eksctl version 0.77.0
2022-01-02 17:11:50 [ℹ]  using region us-east-2
...
2022-01-02 17:14:54 [ℹ]  nodegroup "codemash-ng-1" has 2 node(s)
2022-01-02 17:14:54 [ℹ]  node "ip-192-168-25-109.us-east-2.compute.internal" is ready
2022-01-02 17:14:54 [ℹ]  node "ip-192-168-38-129.us-east-2.compute.internal" is ready
2022-01-02 17:14:54 [✔]  created 1 managed nodegroup(s) in cluster "codemash"
2022-01-02 17:14:54 [ℹ]  checking security group configuration for all nodegroups
2022-01-02 17:14:54 [ℹ]  all nodegroups have up-to-date cloudformation templates
$ kubectl get nodes
NAME                                           STATUS   ROLES    AGE     VERSION
ip-192-168-25-109.us-east-2.compute.internal   Ready    <none>   8m45s   v1.21.5-eks-bc4871b
ip-192-168-38-129.us-east-2.compute.internal   Ready    <none>   8m44s   v1.21.5-eks-bc4871b
```

### Add a sample workload with helm

```console
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
$ helm version
version.BuildInfo{Version:"v3.7.2", GitCommit:"663a896f4a815053445eec4153677ddc24a0a361", GitTreeState:"clean", GoVersion:"go1.16.10"}
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update bitnami
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
$ helm search repo bitnami
NAME                                            CHART VERSION   APP VERSION     DESCRIPTION
bitnami/bitnami-common                          0.0.9           0.0.9           DEPRECATED Chart with custom templates used in ...
...
bitnami/mongodb                                 10.30.11        4.4.11          NoSQL document-oriented database that stores JS...
...
bitnami/tomcat                                  10.0.0          10.0.14         Chart for Apache Tomcat
...
$ helm install bitnami/tomcat --generate-name
NAME: tomcat-1641163641
LAST DEPLOYED: Sun Jan  2 17:47:24 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: tomcat
CHART VERSION: 10.0.0
APP VERSION: 10.0.14

** Please be patient while the chart is being deployed **

1. Get the Tomcat URL by running:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w tomcat-1641163641'

  export SERVICE_IP=$(kubectl get svc --namespace default tomcat-1641163641 --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "Tomcat URL:            http://$SERVICE_IP:/"
  echo "Tomcat Management URL: http://$SERVICE_IP:/manager"

2. Login with the following credentials

  echo Username: user
  echo Password: $(kubectl get secret --namespace default tomcat-1641163641 -o jsonpath="{.data.tomcat-password}" | base64 --decode)
$ helm install bitnami/mongodb --generate-name
NAME: mongodb-1641163726
LAST DEPLOYED: Sun Jan  2 17:48:50 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mongodb
CHART VERSION: 10.30.11
APP VERSION: 4.4.11

** Please be patient while the chart is being deployed **

MongoDB&reg; can be accessed on the following DNS name(s) and ports from within your cluster:

    mongodb-1641163726.default.svc.cluster.local

To get the root password run:

    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default mongodb-1641163726 -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

To connect to your database, create a MongoDB&reg; client container:

    kubectl run --namespace default mongodb-1641163726-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:4.4.11-debian-10-r0 --command -- bash

Then, run the following command:
    mongo admin --host "mongodb-1641163726" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/mongodb-1641163726 27017:27017 &
    mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD
$ helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
mongodb-1641163726      default         1               2022-01-02 17:48:50.0025475 -0500 EST   deployed        mongodb-10.30.11        4.4.11
tomcat-1641163641       default         1               2022-01-02 17:47:24.1903283 -0500 EST   deployed        tomcat-10.0.0           10.0.14
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
mongodb-1641163726-5f47b864d5-2db5k   1/1     Running   0          2m8s
tomcat-1641163641-74f8bd4d86-flb42    1/1     Running   0          3m34s
```

