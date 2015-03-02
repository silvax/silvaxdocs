# Setting Up The Vagrant AWS Plugin with SSHFS 

## Overview
This Doc explains how you can setup vagrant with the ec2 plugin and sshfs. The result is an instance on ec2 that ounts a directory on your local computer. This allows you to edit code locally and test it immeditaly on the service running on the ec2 instance


Follow the steps on this document to setup the environment using Vagrant

## Prerequisites 
To setup this test we are doing some assumptions and also requiring the following:

### Assumptions 

This test was done on MacOS X Yosemite 10.10.1

### Required Components

You must install Vagrant. More info and download at http://www.vagrantup.com Vagrant is Open Source and free to use.

You must install the Vagrant AWS plugin. More info on the Vagrant AWS plugin can be found here https://github.com/mitchellh/vagrant-aws . To install the plugin, after installing vagrant just enter the following

```
vagrant plugin install vagrant-aws
```

On MacOS you must enable "Remote Login" or ssh login. This is necessary to reverse mount the local directory from the ec2 instance over ssh. However, note that the connection will not be initiated from the ec2 instance. See the documentation below to understand that the connection is first establish from the local computer. 

You must have created a VPC, Subnet, Security Group and EC2 Key Pair that can be used for the vagrant on ec2 setup

## Step by Step Procedure

Create a temporary directory on your computer where you will store the vagrant files. You could also use the directory where you have your code checked out that will need to be made available via sshfs from the remote ec2 instance. In any case after choosing the directory you will need to initilize the Vagrant environment. You can do that with the following command:
```
vagrant init
```

This will create a file called Vagrantfile in the directory . Edit this file and replace it with this one. Note that this file already has vpc, security groups, subnets and key pairs that were created for this test. You can use these if they are still available. Also, please note that you will need to replace the access, secret and token to authenticate with AWS. You will also need to update the path to the SSH key on your local computer. 

```
# -*- mode: ruby -*-
# vi: set ft=ruby :
# This is a Vagrant configuration file. It can be used to set up and manage
# virtual machines on your local system or in the cloud. See http://downloads.vagrantup.com/
# for downloads and installation instructions, and see http://docs.vagrantup.com/v2/
# for more information and configuring and using Vagrant.

Vagrant.configure("2") do |config|
  config.ssh.pty = true
  config.vm.box = "dummy"
  config.vm.synced_folder ".", "/vagrant", type: "rsync"

  config.vm.provider :aws do |aws, override|
    aws.access_key_id = "CHANGE-ME"
    aws.secret_access_key = "CHANGE-ME"
    aws.keypair_name = "DevPOC-JWTVDev"
    aws.ami = "ami-8cb8e7e4"
    aws.instance_type = "t2.micro"
    aws.security_groups = ["sg-5cdf0938"]
    aws.subnet_id = "subnet-02c2685b"
    aws.associate_public_ip = true
    aws.tags = {
      'Name' => 'test_sshfs',
      'Owner' => 'Andres Silva'
    }
    override.ssh.username = "ubuntu"
    override.ssh.private_key_path = "/Users/asilva/.ssh/keys/test-key.pem"
  end
  config.vm.provision "shell", inline: "sudo apt-get install sshfs -y"
end
```

Save the file

Now at the command line in the directory where the Vagrantfile is saved, enter the following command to bring up the vagrant ec2 instance

```
vagrant up --provider=aws
```

This will bring up the ec2 instance and install sshfs on it.

On this next step we are going to establish an SSH tunnel to the instance and redirect a port that will allow us to tunnel back to mount a directory on our local computer onto the ec2 instance via ssh. Here is the step by step process on how to do this

ssh into the instance with the following command. In this example Im using port 8089. This can be any unused port. 

```
vagrant ssh -- -R 8089:localhost:22
```

This will ssh into the ec2 instance and redirect port 22 on the target to port 8089 on the local computer. This will allow us now to setup a reverse tunnel back into the local computer form the remote ec2 instance

Now create a directory that will be used to mount the filesystem. You can use a command like this

```
mkdir -p /home/ubuntu/test
```

Now at the ec2 instance enter the following command to mount the local file system remotely. Note how we use the same port 8089 that we setup on the previous step. Also, note that the user is you user name on the local computer. The host will be "localhost" this will force the connection back to the local computer via the redirected port 8089. 

```
sshfs -o port=8089 asilva@localhost:/Users/asilva/Documents/bethel-aws/ /home/ubuntu/test
```

Now if you navigate to /home/ubuntu/test on the ec2 instance the contents of that directory will be target directory on the local computer. This would allow a developer to change files locally on their computer and immediately reload the service on the ec2 instance to test the changes. 
