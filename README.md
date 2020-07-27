# Instruction

## Abstract
- In the scope of this testing, we are going to deploy a minimal testing enviroment for Edge/IoT computing. There will be a simulated environment as a DataCenter where all the information of Edge Computing return, a simulated environment for Edge/IoT Data Server where IoT sensors running.

- At simulated DataCenter, there will be a GUI to display information sent from IoT sensors running on Edge. All the changes at the edge will be displayed on this GUI.

- The simulated DataCenter is also used for Development/Testing Environment which includes CI/CD.

- [TODO] Exercise 3rd and 4th phases for more use cases. i.e running AI/ML at the edge


## Prerequisites
- There needs to have two Red Hat Openshift Cluster:
  * Cluster 1: Factory Datacenter, CI/CD and Testing env.
  * Cluster 2: Edge/IoT Data Server

## Cloning necessary repos in both of environment
- Clone the repo: https://github.com/vietstacker/manuela-dev.git
- Clone the repo: https://github.com/vietstacker/manuela-gitops.git
- Clone the repo: https://github.com/vietstacker/manuela.git


## Use Case 1: Testing Edge Computing information sent back to Central DC and displayed there.
### On the Cluster 1 (Factory Datacenter/ CICD-Testing Env)
- Change the cluster hostname. Grep the existing hostname from github (as an example) and replace it by your cluster name. Run the below command line and replace you cluster name in all the files.
```bash
cd ~/manuela-gitops
grep -r "sing-1a06" .
```

- Subscribe to neccessary operators.
``` bash
cd ~/manuela
oc apply -k namespaces_and_operator_subscriptions/openshift-pipelines
oc apply -k namespaces_and_operator_subscriptions/manuela-ci
oc apply -k namespaces_and_operator_subscriptions/argocd
oc apply -k namespaces_and_operator_subscriptions/manuela-temp-amq
```

- Wait for ArgoCD operator to be subscribed and available
```bash
oc get pods -n argocd

NAME                                             READY   STATUS              RESTARTS   AGE
argocd-operator-6d5d7bf9c5-kppd6                 1/1     Running             0          4d18h
```
- Create an ArgoCD instance and assign the cluster-admin role to its serviceaccount

```bash
oc apply -k infrastructure/argocd
oc adm policy add-cluster-role-to-user cluster-admin -n argocd -z argocd-application-controller
```

- Wait for the argocd secret to be created
```bash
oc get secret argocd-secret -n argocd

NAME            TYPE     DATA   AGE
argocd-secret   Opaque   2      30s
```

- Check the ArgoCD up and running

```bash
oc get pods -n argocd

NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller                    1/1     Running   0          1m
argocd-dex-server                                1/1     Running   0          1m
argocd-redis                                     1/1     Running   0          1m
argocd-repo-server                               1/1     Running   0          1m
argocd-server                                    1/1     Running   0          1m
```

- Deploy the Factory Data Center and CI/CD-Testing Env.

```bash
oc create -n argocd -f ~/manuela-gitops/meta/argocd-ocp3.yaml
```
Note: In this scope, Red Hat OpenDataHub (ODH) is not used. You can see the .orig files in the ~/manuela-gitops/deployment/execenv-ocp3

- Remove manuela-temp-amq namespace

This namespace was created to kickstart the ArgoCD deployment of manuela-tst-all by making the AMQ Broker CRD known to the cluster. It can now be removed:
```bash
oc delete -k namespaces_and_operator_subscriptions/manuela-temp-amq
```

### On the Cluster 2 (Edge/IoT Data Server)

- Change the cluster hostname. Grep the existing hostname from github (as an example) and replace it by your cluster name. Run the below command line and replace you cluster name in all the files.
```bash
cd ~/manuela-gitops
grep -r "sing-1a06" .
```

- Deploy ArgoCd
```bash
cd ~/manuela
oc apply -k namespaces_and_operator_subscriptions/argocd
oc apply -k infrastructure/argocd
```

- Deploy sensor services
```bash
oc apply -n argocd -f ~/manuela-gitops/meta/argocd-ocp4.yaml
```

- Remove manuela-temp-amq namespace
```bash
oc delete -k namespaces_and_operator_subscriptions/manuela-temp-amq
```

## Use Case 2: Testing CI/CD flow on the CI/CD Environment for Central DC applications development and deploy them.
### On the Cluster 1 (Factory Datacenter/ CICD-Testing Env)

- Make sure that Cluster 1 is deployed, up and running as in use case 1

- Adjust Tekton secrets to match your environments as below:

- Github Secret. Search for how to get Github personal access token if you do not know:

```bash
cd ~/manuela-dev
export GITHUB_PERSONAL_ACCESS_TOKEN=<ChangeMe>
sed "s/token: cmVwbGFjZW1l/token: $(echo -n $GITHUB_PERSONAL_ACCESS_TOKEN|base64)/" tekton/secrets/github-example.yaml >tekton/secrets/github.yaml
```

```bash
cd ~/manuela-dev
export GITHUB_USER=<ChangeMe>
sed -i "s/user: cmVwbGFjZW1l/user: $(echo -n $GITHUB_USER|base64)/" tekton/secrets/github.yaml
```

- ArgoCD Secret:

```bash
sed "s/ARGOCD_PASSWORD:.*/ARGOCD_PASSWORD: $(oc get secret argocd-cluster -n argocd -o jsonpath='{.data.*}')/" tekton/secrets/argocd-env-secret-example.yaml >tekton/secrets/argocd-env-secret.yaml
```

- Quay Build Secret:
```bash
export QUAY_BUILD_SECRET=ewogICJhdXRocyI6IHsKICAgICJxdWF5LmlvIjogewogICAgICAiYXV0aCI6ICJiV0Z1ZFdWc1lTdGlkV2xzWkRwSFUwczBRVGMzVXpjM1ZFRlpUMVpGVGxWVU9GUTNWRWRVUlZOYU0wSlZSRk5NUVU5VVNWWlhVVlZNUkU1TVNFSTVOVlpLTmpsQk1WTlZPVlpSTVVKTyIsCiAgICAgICJlbWFpbCI6ICIiCiAgICB9CiAgfQp9
sed "s/\.dockerconfigjson:.*/.dockerconfigjson: $QUAY_BUILD_SECRET/" tekton/secrets/quay-build-secret-example.yaml >tekton/secrets/quay-build-secret.yaml
```

- Adjust Tekton Config Map:
Tekton environment config map should have to reflect the environments used for CI/CD. You need to change the values which begin with "GIT_" or end with "REMOTE_IMAGE".
The path of tektok config map is "~/manuela-dev/tekton/configmaps/environment"

- Instantiate pipelines:

```bash
cd ~/manuela-dev
oc apply -k tekton/secrets -n manuela-ci
oc apply -k tekton -n manuela-ci
```
