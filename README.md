# Kubernetes cluster on AWS
Each of the steps will cover:

Step 1: **Configure** (this document)

 - Download the kube-aws CloudFormation generator
 - Define account and cluster settings

Step 2: **Render**

 - Compile a re-usable CloudFormation template for the cluster
 -    Optionally adjust template configuration Validate the rendered
 -    CloudFormation stack

Step 3: **Launch**

 - Create the CloudFormation stack and start our EC2 machines 
 - Set up CLI   access to the new cluster

## Download kube-aws
```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key FC8A365E
$ gpg2 --fingerprint FC8A365E
$ PLATFORM=linux-amd64

$ wget https://github.com/coreos/coreos-kubernetes/releases/download/v0.8.1/kube-aws-linux-amd64.tar.gz

$ tar zxvf kube-aws-${PLATFORM}.tar.gz
$ sudo mv ${PLATFORM}/kube-aws /usr/local/bin
```
## Configure AWS credentials

```bash
$ aws configure
AWS Access Key ID [None]: AKID1234567890
AWS Secret Access Key [None]: MY-SECRET-KEY
Default region name [None]: us-west-2
Default output format [None]: text
```

## Configure your Kubernetes cluster on AWS

**KMS key**
Amazon KMS keys are used to encrypt and decrypt cluster TLS assets. If you already have a KMS Key that you would like to use, you can skip creating a new key and provide the Arn string for your existing key.

You can create a KMS key in the AWS console, or with the aws command line tool:

```bash
$ aws kms --region=us-west-2 create-key --description="kube-aws assets"
```

**output:**

```text
{
    "KeyMetadata": {
        "CreationDate": 1458235139.724,
        "KeyState": "Enabled",
        "Arn": "arn:aws:kms:us-west-1:xxxxxxxxx:key/xxxxxxxxxxxxxxxxxxx",
        "AWSAccountId": "xxxxxxxxxxxxx",
        "Enabled": true,
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyId": "xxxxxxxxx",
        "Description": "kube-aws assets"
    }
}
```
You will use the KeyMetadata.Arn string to identify your KMS key in the init step.


**Initialize an asset directory**
```bash
$ mkdir cluster-asset
$ cd cluster-asset
```

**Initialize the cluster CloudFormation stack with the KMS Arn, key pair name, and DNS name from the previous step:**

```bash
$ kube-aws init \
--cluster-name=weddingday-cluster \
--external-dns-name=cluster.weddingday.com \
--region=us-west-2 \
--availability-zone=us-west-2a \
--key-name=staging_pg \
--kms-key-arn="arn:aws:kms:us-west-1:xxxxxxxxxx:key/xxxxxxxxxxxxxxxxxxx"
```	

**Render contents of the asset directory**
Next, generate a set of cluster assets in the current directory, based on the settings in your cluster.yaml file.

```bash
$ kube-aws render
```
Create the instances defined in the CloudFormation template

Now for the exciting part, creating your cluster:
```bash
$ kube-aws up
```
**NOTE:** It can take some time after kube-aws up completes before the cluster is available. When the cluster is first being launched, it must download all container images for the cluster components.

**Install kubectl**

```bash
$ curl -O https://storage.googleapis.com/kubernetes-release/release/v1.3.4/bin/linux/amd64/kubectl
```
After downloading the binary, ensure it is executable and move it into your PATH:
```bash
$ sudo chmod +x kubectl
$ sudo mv kubectl /usr/local/bin/kubectl
```

**Once the API server is running, you should see:**
```bash
$ kubectl --kubeconfig=kubeconfig get nodes

NAME                                       STATUS                     AGE
ip-10-0-0-xxx.us-west-2.compute.internal   Ready                      5m
ip-10-0-0-xxx.us-west-2.compute.internal   Ready                      5m
ip-10-0-0-xx.us-west-2.compute.internal    Ready,SchedulingDisabled   5m
```