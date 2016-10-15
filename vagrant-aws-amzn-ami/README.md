Vagrantfile for Amazon EC2
==========================

## Environment

* Vagrant 1.7.2
* vagrant-aws 0.6.0

## Usage

* Install vagrant-aws plugin

```bash
$ vagrant plugin install vagrant-aws
```

* Set environment values

```bash
# require
export AWS_ACCESS_KEY_ID=YOUR_VALUE
export AWS_SECRET_ACCESS_KEY=YOUR_VALUE
export AWS_KEYPAIR_NAME=YOUR_VALUE
export AWS_PRIVATE_KEY_FILE=YOUR_VALUE

# optional
export AWS_REGION=YOUR_VALUE
export AWS_AVAILABILITY_ZONE=YOUR_VALUE
export AWS_AMI_ID=YOUR_VALUE
export AWS_INSTANCE_TYPE=YOUR_VALUE

## VPC
export AWS_PRIVATE_IP=YOUR_VALUE
export AWS_SUBNET_ID=YOUR_VALUE
export AWS_SECURITY_GROUP_IDS=YOUR_VALUE

## My tags (format is "key1:value1,key2:value2,...,keyN:valueN")
export AWS_TAGS="Name:YOUR_INSTANCE_NAME,Key2:Value2"
```

* vagrant up

```bash
$ VAGRANT_LOG=debug vagrant up --provider=aws [--provision] >& vagrant_up.log
```

* vagrant ssh

```bash
$ vagrant ssh
```
