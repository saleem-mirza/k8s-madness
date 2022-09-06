# RBAC misconfiguration and exploits

## Scenario

Investigate a possible permission escape in Namespace `restricted`. The context authenticates as user restricted which has only limited permissions and shouldn't be able to read Secret values.

Try to find the password-key values of the Secrets secret1, secret2 and secret3 in Namespace `restricted` and print decoded values.


## Assumption

To proceed, you must have the following:

- An existing AWS account with administrator permissions
- An existing and working EKS cluster with Kubernetes version 1.21 to 1.24
- The latest versions of the following utilities installed and configured on the workspace that you will use to interact with AWS and the Amazon EKS cluster
  - aws cli
  - eksctl
  - kubectl
  - jq
  - git
- Linux bash like environment


## Setup

Deploy sample application. 

```bash
# create namespace `restricted`
kubectl apply -f ./namespace.yaml

# create secrets
kubectl apply -f ./secrets.yaml

# create role and rolebinding
kubectl apply -f ./roles.yaml
kubectl apply -f ./rolebindings.yaml

# create service account
kubectl apply -f ./sa.yaml

# deploy sample pods
kubectl apply -f ./deploy.yaml
```

## Create AWS IAM user

```bash
aws iam create-user --user-name restricted
USER_ARN=$(aws iam get-user --user-name restricted | jq .User.Arn)
echo "User Arn: ${USER_ARN}"
```

## Update `aws-auth` ConfigMap

EKS stores IAM/RBAC mapping information in a ConfigMap called aws-auth in kube-system namespace. Append IAM user mapping into ConfigMap

```bash
kubectl edit cm aws-auth -n kube-system
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::<AWS Account ID>:user/restricted
      username: restricted
```
## Create AccessKey and Secret Keys

Create keys for user restricted to access resources programatically. Save keys into environment variables

```bash
RESULT=$(aws iam create-access-key --user-name restricted)
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey <<< $RESULT)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId <<< $RESULT)
```

## Get IAM user details based on exported secret and access keys

```bash
aws sts get-caller-identity
```

A successful execution of command will print

```text
{
    "Account": <AWS Account ID>,
    "UserId": <AWS User ID>,
    "Arn": "arn:aws:iam::<AWS Account ID>:user/restricted"
}

```

Now, lets make a request to list secrets in context of `restricted` user to see the effect.

```bash
kubectl get secrets -n restrcited
```
