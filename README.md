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

I needed to create and use an ECR repo, so I created a policy with `ecr:CreateRepository` and `ecr:PutImage` for `arn:aws:ecr:us-east-2:054858005475:repository/*` and `ecr:GetAuthorizationToken` for `*` called `ECR-create` and added it to the user. 

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

## kube-bench

Instructions at (https://github.com/aquasecurity/kube-bench/blob/main/docs/running.md#running-in-an-eks-cluster).

I needed to create and use an ECR repo, so I created a policy with `ecr:CreateRepository` for `arn:aws:ecr:us-east-2:054858005475:repository/*` and `ecr:GetAuthorizationToken` for `*` called `ECR-create` and added it to the user. I also added `AmazonEC2ContainerRegistryPowerUser` so the user can push images

```console
$ aws ecr create-repository --repository-name k8s/kube-bench --image-tag-mutability MUTABLE
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-2:054858005475:repository/k8s/kube-bench",
        "registryId": "054858005475",
        "repositoryName": "k8s/kube-bench",
        "repositoryUri": "054858005475.dkr.ecr.us-east-2.amazonaws.com/k8s/kube-bench",
        "createdAt": "2022-01-02T18:05:49-05:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
$ cd ..
$ rm -rf kube-bench
$ git clone https://github.com/aquasecurity/kube-bench.git
$ cd kube-bench
$ aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 054858005475.dkr.ecr.us-east-2.amazonaws.com
Login Succeeded
$ docker build -t k8s/kube-bench .
[+] Building 68.6s (29/29) FINISHED
...
$ docker tag k8s/kube-bench:latest 054858005475.dkr.ecr.us-east-2.amazonaws.com/k8s/kube-bench:latest
$ docker push 054858005475.dkr.ecr.us-east-2.amazonaws.com/k8s/kube-bench:latest
The push refers to repository [054858005475.dkr.ecr.us-east-2.amazonaws.com/k8s/kube-bench]
...
latest: digest: sha256:18777462bf3d8d85b7d8acb15cf5e65877f00c6ed946e9f098caa5cf20cbe057 size: 2832
```

In `job-eks.yaml`, add `054858005475.dkr.ecr.us-east-2.amazonaws.com/k8s/kube-bench:latest` as `image` in line 14.

```console
$ kubectl apply -f job-eks.yaml
job.batch/kube-bench created
$ kubectl get pods
NAME                                  READY   STATUS      RESTARTS   AGE
kube-bench-x7vgh                      0/1     Completed   0          7s
mongodb-1641163726-5f47b864d5-2db5k   1/1     Running     0          40m
tomcat-1641163641-74f8bd4d86-flb42    1/1     Running     0          41m
$ kubectl logs kube-bench-x7vgh
[INFO] 3 Worker Node Security Configuration
[INFO] 3.1 Worker Node Configuration Files
[PASS] 3.1.1 Ensure that the kubeconfig file permissions are set to 644 or more restrictive (Manual)
[PASS] 3.1.2 Ensure that the kubelet kubeconfig file ownership is set to root:root (Manual)
[PASS] 3.1.3 Ensure that the kubelet configuration file has permissions set to 644 or more restrictive (Manual)
[PASS] 3.1.4 Ensure that the kubelet configuration file ownership is set to root:root (Manual)
[INFO] 3.2 Kubelet
[PASS] 3.2.1 Ensure that the --anonymous-auth argument is set to false (Automated)
[PASS] 3.2.2 Ensure that the --authorization-mode argument is not set to AlwaysAllow (Automated)
[PASS] 3.2.3 Ensure that the --client-ca-file argument is set as appropriate (Manual)
[PASS] 3.2.4 Ensure that the --read-only-port argument is set to 0 (Manual)
[PASS] 3.2.5 Ensure that the --streaming-connection-idle-timeout argument is not set to 0 (Manual)
[PASS] 3.2.6 Ensure that the --protect-kernel-defaults argument is set to true (Automated)
[PASS] 3.2.7 Ensure that the --make-iptables-util-chains argument is set to true (Automated)
[PASS] 3.2.8 Ensure that the --hostname-override argument is not set (Manual)
[WARN] 3.2.9 Ensure that the --eventRecordQPS argument is set to 0 or a level which ensures appropriate event capture (Automated)
[PASS] 3.2.10 Ensure that the --rotate-certificates argument is not set to false (Manual)
[PASS] 3.2.11 Ensure that the RotateKubeletServerCertificate argument is set to true (Manual)

== Remediations node ==
3.2.9 If using a Kubelet config file, edit the file to set eventRecordQPS: to an appropriate level.
If using command line arguments, edit the kubelet service file
/etc/systemd/system/kubelet.service on each worker node and
set the below parameter in KUBELET_SYSTEM_PODS_ARGS variable.
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service


== Summary node ==
14 checks PASS
0 checks FAIL
1 checks WARN
0 checks INFO

== Summary total ==
14 checks PASS
0 checks FAIL
1 checks WARN
0 checks INFO
```

## kube-hunter

Sign up at (https://kube-hunter.aquasec.com/).

Copy the `docker run` command and run it.

```console
$ docker run -it --rm --network host aquasec/kube-hunter:aqua --token ...
Choose one of the options below:
1. Remote scanning      (scans one or more specific IPs or DNS names)
2. Interface scanning   (scans subnets on all local network interfaces)
3. IP range scanning    (scans a given IP range)
Remotes (separated by a ','): 192.168.25.109,192.168.38.129
2022-01-02 23:37:05,304 INFO kube_hunter.modules.report.collector Started hunting

Report will be available at:
...
```

Since we are behind a Wireguard VPN, it can find the cluster at all. That is good.


## Checkov

```console
$ pip3 install -U checkov
$ cd ..
$ git clone https://github.com/bitnami/bitnami-docker-mongodb.git
$ cd bitnami-docker-mongodb/
$ checkov -d 5.0/debian-10/
$ cd ../deptrack/
$ checkov -d .
```

## Trivy

Install with: 

```console
$ curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.22.0
```

The vulnerability data is downloaded/updated on scan, if needed.

```console
$ trivy image docker.io/bitnami/mongodb:4.4.11-debian-10-r0
$ trivy image docker.io/bitnami/tomcat:10.0.14-debian-10-r21
$ trivy image webgoat/webgoat-7.1
2022-01-02T18:49:53.617-0500    INFO    Need to update DB
2022-01-02T18:49:53.617-0500    INFO    Downloading DB...
25.32 MiB / 25.32 MiB [------------------------------------------------------------------------------------------------------------------------] 100.00% 10.82 MiB p/s 3s
2022-01-02T18:50:05.347-0500    INFO    Detected OS: debian
2022-01-02T18:50:05.347-0500    INFO    Detecting Debian vulnerabilities...
2022-01-02T18:50:05.357-0500    INFO    Number of language-specific files: 1
2022-01-02T18:50:05.357-0500    INFO    Detecting jar vulnerabilities...
2022-01-02T18:50:05.360-0500    WARN    This OS version is no longer supported by the distribution: debian 8.6
2022-01-02T18:50:05.360-0500    WARN    The vulnerability detection may be insufficient because security updates are not provided

webgoat/webgoat-7.1 (debian 8.6)
================================
Total: 890 (UNKNOWN: 23, LOW: 264, MEDIUM: 241, HIGH: 256, CRITICAL: 106)
...
```

Pulled (https://github.com/aquasecurity/trivy/blob/main/contrib/html.tpl) as `trivy-html.tpl`.

```console
$ trivy image --format template --template "@trivy-html.tpl" -o webgoat-trivy-report.html webgoat/webgoat-8.0
2022-01-02T18:57:38.022-0500    INFO    Detected OS: debian
2022-01-02T18:57:38.022-0500    INFO    Detecting Debian vulnerabilities...
2022-01-02T18:57:38.031-0500    INFO    Number of language-specific files: 1
2022-01-02T18:57:38.031-0500    INFO    Detecting jar vulnerabilities...
```

Results in `webgoat-trivy-report.html`. Search for `CVE-2021-44228` == log4shell.

## Polaris

### Dashboard

Instructions at (https://polaris.docs.fairwinds.com/dashboard/#installation).

```console
$ kubectl apply -f https://github.com/fairwindsops/polaris/releases/latest/download/dashboard.yaml
namespace/polaris created
serviceaccount/polaris created
clusterrole.rbac.authorization.k8s.io/polaris created
clusterrolebinding.rbac.authorization.k8s.io/polaris-view created
clusterrolebinding.rbac.authorization.k8s.io/polaris created
service/polaris-dashboard created
deployment.apps/polaris-dashboard created
$ kubectl port-forward --namespace polaris svc/polaris-dashboard 8555:80
```

Report will be at (https://localhost:8555). Note the memory/CPU requests/limits (foreshadowing).


### Admission controller

Instructions at (https://polaris.docs.fairwinds.com/admission-controller/#installation).

Install cert-manager (https://cert-manager.io/docs/installation/).

```console
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

Download the `webhook.yaml` and edit the `apiVersion`s:
```diff
176c176
< apiVersion: cert-manager.io/v1alpha2
---
> apiVersion: cert-manager.io/v1
196c196
< apiVersion: cert-manager.io/v1alpha2
---
> apiVersion: cert-manager.io/v1
205c205
< apiVersion: admissionregistration.k8s.io/v1beta1
---
> apiVersion: admissionregistration.k8s.io/v1
213c213
<   - v1beta1
---
>   - v1
```

Add and delete WebGoat to show it works:

```console
$ kubectl run webgoat --image webgoat/webgoat-7.1
pod/webgoat created
$ kubectl delete pod webgoat
pod "webgoat" deleted
```

Install the admission controller:

```
$ kubectl apply -f webhook.yaml
namespace/polaris created
serviceaccount/polaris created
clusterrole.rbac.authorization.k8s.io/polaris created
clusterrolebinding.rbac.authorization.k8s.io/polaris-view created
clusterrolebinding.rbac.authorization.k8s.io/polaris created
service/polaris-webhook created
deployment.apps/polaris-webhook created
certificate.cert-manager.io/polaris-cert created
issuer.cert-manager.io/polaris-selfsigned created
validatingwebhookconfiguration.admissionregistration.k8s.io/polaris-webhook created
```

After 10 seconds or so, try to add WebGoat again:

```console
$ kubectl run webgoat --image webgoat/webgoat-7.1
Error from server (
Polaris prevented this deployment due to configuration problems:
- Container webgoat: Privilege escalation should not be allowed
- Container webgoat: Image tag should be specified
): admission webhook "polaris.fairwinds.com" denied the request:
Polaris prevented this deployment due to configuration problems:
- Container webgoat: Privilege escalation should not be allowed
- Container webgoat: Image tag should be specified
```

### Static analysis

Instructions at (https://polaris.docs.fairwinds.com/infrastructure-as-code/#install-the-cli).

Download (https://github.com/FairwindsOps/polaris/releases/download/4.2.0/polaris_linux_amd64.tar.gz), unpack, and copy to `/usr/local/bin`.

```console
$ wget https://github.com/FairwindsOps/polaris/releases/download/4.2.0/polaris_linux_amd64.tar.gz
...
2022-01-02 21:37:39 (11.0 MB/s) - ‘polaris_linux_amd64.tar.gz’ saved [11541484/11541484]

$ gzip -dc polaris_linux_amd64.tar.gz | tar xvf - polaris
polaris
$ sudo mv polaris /usr/local/bin/
$ polaris version
Polaris version:4.2.0
```

Scan `deptrack` again:

```console
$ cd ../deptrack/
$ polaris audit --audit-path .
...
$ polaris audit --audit-path . --format=pretty
...
$ polaris audit --audit-path . --format=pretty --only-show-failed-tests true
...
```

## Goldilocks

Instructions at (https://goldilocks.docs.fairwinds.com/installation/#requirements). The Polaris admission controller must not be installed before installing metrics-server since it will block for privilege escalation.

Install [metrics-server](https://github.com/kubernetes-sigs/metrics-server).

```console
$ kubectl delete -f webhook.yaml
...
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
...
```

Install VPA and Goldilocks:

```console
$ helm repo add fairwinds-stable https://charts.fairwinds.com/stable
"fairwinds-stable" already exists with the same configuration, skipping
$ helm install vpa fairwinds-stable/vpa --namespace vpa --create-namespace
NAME: vpa
LAST DEPLOYED: Mon Jan  3 09:06:07 2022
NAMESPACE: vpa
STATUS: deployed
REVISION: 1
NOTES:
Congratulations on installing the Vertical Pod Autoscaler!

Components Installed:
  - recommender
  - updater

To verify functionality, you can try running 'helm -n vpa test vpa'
$ helm -n vpa test vpa
NAME: vpa
LAST DEPLOYED: Mon Jan  3 09:06:07 2022
NAMESPACE: vpa
STATUS: deployed
REVISION: 1
...
Congratulations on installing the Vertical Pod Autoscaler!

Components Installed:
  - recommender
  - updater

To verify functionality, you can try running 'helm -n vpa test vpa'
$ helm install goldilocks --namespace goldilocks fairwinds-stable/goldilocks --create-namespace
NAME: goldilocks
LAST DEPLOYED: Mon Jan  3 09:06:54 2022
NAMESPACE: goldilocks
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace goldilocks -l "app.kubernetes.io/name=goldilocks,app.kubernetes.io/instance=goldilocks,app.kubernetes.io/component=dashboard" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

Enable Goldilocks in each namespace you want to monitor.

```console
$ kubectl label pods ns goldilocks goldilocks.fairwinds.com/enabled=true
$ kubectl label pods ns default goldilocks.fairwinds.com/enabled=true
$ kubectl -n goldilocks port-forward svc/goldilocks-dashboard 8444:80
```

The Goldilocks dashboard will be on (http://localhost:8444/).
