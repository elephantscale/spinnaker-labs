# Configuring AWS

## Install AWS tools v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```


## Configure AWS
```bash
aws configure  # enter the Keys we will get
```

```console

aws-cli 1.15.58 from Amazon Web Services (aws✓) installed
ubuntu@ip-172-16-0-64:~$
ubuntu@ip-172-16-0-64:~$ aws sts get-caller-identity

{
    "Arn": "arn:aws:sts::092413168457:assumed-role/Spinnaker-Managing-Role/i-0bf3f22b08ccdd16e",
    "UserId": "AROARLBCBBNE3N7YM2ADI:i-0bf3f22b08ccdd16e",
    "Account": "092413168457"
}
ubuntu@ip-172-16-0-64:~$ aws sts assume-role --role-arn arn:aws:iam::092413168457:role/Spinnaker-Managed-Role --role-session-name   test
{
    "Credentials": {
        "SessionToken": "FwoGZXIvYXdzEM7//////////wEaDFMdS1kpdCbo9c/DRCKoAaVI9adJoMX8JWNqQFiAZLougeSObEFPIn2TLZXeksfIQ/XOHDQX+BDEjV4UFxYORve4Fj49FSdpfcbK370tyeI1BHDMh++ivLhWSbmZCCz0/iKaZDyvmJ1gQRp5WJebBrWb6qNW5oYoQjFkixYcFtqm3heBCFDfNyqiiE0vqoYWGu2AkMWC0bc+qEMDXl7c7zSCqLtx7STOY4PLNyE2a32YTJGbYV0QyyjkhrCKBjItM/KmVYWpNE4aKyC5mMVi2CKB8XvzntY6BS2GfRJ4jg0j11Tgb5w2xdHqbwKm",
        "AccessKeyId": "ASIARLBCBBNERQ6GQK54",
        "Expiration": "2021-09-23T05:32:36Z",
        "SecretAccessKey": "a+kyU0LBZG3CuIsFHAZl2KFHc6nvis7CnwjMON5R"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROARLBCBBNESOM3YMEEJ:test",
        "Arn": "arn:aws:sts::092413168457:assumed-role/Spinnaker-Managed-Role/test"
    }
}
```



## Confirm that your VM has IAM role to access AWS

```bash
aws sts assume-role --role-arn arn:aws:iam::092413168457:role/Spinnaker-Managed-Role --role-session-name   test
export AWS_ACCOUNT_NAME=elephantscale export ACCOUNT_ID=092413168457 export ROLE_NAME=role/Spinnaker-Managed-Role
export AWS_ACCOUNT_NAME=elephantscale export ACCOUNT_ID=092413168457 export ROLE_NAME=role/Spinnaker-Managed-Role
```


## Configure AWS
```bash
aws configure  # enter the Keys we will get
```

##  Use Halyard to set up aws account
```bash
export AWS_ACCOUNT_NAME=elephantscale \
export ACCOUNT_ID=092413168457 \
export ROLE_NAME=role/Spinnaker-Managed-Role
hal config provider aws account add ${AWS_ACCOUNT_NAME}   --account-id ${ACCOUNT_ID}   --assume-role ${ROLE_NAME}   --regions us-east-1
hal config provider aws enable
hal deploy apply
hal deploy connect

##  Install ekscli 

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Create an EKS Cluster

```bash
ssh-keygen -o
cd spinnaker-labs/cloud
vi cluster.yaml
eksctl create cluster -f cluster.yaml
eksctl utils write-kubeconfig --cluster=tim-cluster3
```


## Install Spinnaker Tools

```bash
curl -L https://github.com/armory/spinnaker-tools/releases/download/0.0.7/spinnaker-tools-linux -o spinnaker-tools
chmod +x spinnaker-tools
./spinnaker-tools --version
```

## Create Service account
```bash
./spinnaker-tools create-service-account
```

