# Packer

<br>

## Overview ([https://www.packer.io/intro](https://www.packer.io/intro))
- Packer is an open source tool for creating identical machine images for multiple platforms from a single source configuration.
- A machine image is a single static unit that contains a pre-configured operating system and installed software which is used to quickly create new running machines. Machine image formats change for each platform. Some examples include AMIs for EC2, VMDK/VMX files for VMware, OVF exports for VirtualBox, etc.

<br>

## Notes

- Packer configuration files are written in `json`
- We will need a base AMI to actually provision -- these can be the default Ubuntu ones given in AWS
- The provisioning can be achieved through several ways: Ansible, Shell scripts etc.

<br>

# AMI's

- These are `Amazon Machine Images` -- snapshots of a virtual machine in a certain state

<br>

## Packer

- Install `Packer` onto a machine

- Packer will create AMI's for us on some cloud service
    - It will use a base AMI and will use a provisioning tool (like Ansible) to put it in a desired state
    - Once the instance has been created and provisioned, it will create an AMI of that instance and store it in the list of AMI's
    - In AWS, this is achieved by spinning up a temporary EC2 instance, provisioning it, taking an AMI of it and finally terminating the instance

- Iteration 1:
```json
{
  "variables": {
    "aws_access_key": "",
    "aws_secret_key": ""
  },
  "builders": [{
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "eu-west-1",
      "source_ami_filter": {
        "filters": {
          "virtualization-type": "hvm",
          "name": "ubuntu/images/*ubuntu-bionic-18.04-amd64-server-*",
          "root-device-type": "ebs"},
        "owners": ["099720109477"],
        "most_recent": true},

      "instance_type": "t2.micro",
      "ssh_username": "ubuntu",
      "ami_name": "packer-example {{timestamp}}"
    }]
}
```

- We now need to ensure that Packer can read the AWS access and secret keys. Since we do not want these keys to be available publicly, a quick fix is to include them within our environment variables
    - We can achieve this through several ways but a quick one is to just put them inside `.bashrc` and run it so it takes effect
    - This ensures that the keys will always be available as environment variables

- Iteration 2:
    - We can reference environment variables in json through the env keyword. The referencing is identical to Jinja2, i.e. the double curly braces
```json
{
//// NEW CODE

  "variables": {
    "aws_access_key": "{{env `AWS_ACCESS_KEY`}}",
    "aws_secret_key": "{{env `AWS_SECRET_KEY`}}"
  },

////

  "builders": [{
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "eu-west-1",
      "source_ami_filter": {
        "filters": {
          "virtualization-type": "hvm",
          "name": "ubuntu/images/*ubuntu-bionic-18.04-amd64-server-*",
          "root-device-type": "ebs"
        },
        "owners": ["099720109477"],
        "most_recent": true
      },
      "instance_type": "t2.micro",
      "ssh_username": "ubuntu",
      "ami_name": "eng74-jared-packer-ami-{{timestamp}}"
    }],
}
```

- As can be seen, this AMI doesn't really change anything -- it is an identical copy to the `bionic64` one we normally use as we have not provisioned it with anything. We will need to add a `provision` section
    - We can provision the AMI using several tools such as bash scripts or Ansible
    - The following is an example of using Ansible

- Iteration 3:
```json
{
  "variables": {
    "aws_access_key": "{{env `AWS_ACCESS_KEY`}}",
    "aws_secret_key": "{{env `AWS_SECRET_KEY`}}"
  },
  "builders": [{
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "eu-west-1",
      "source_ami_filter": {
        "filters": {
          "virtualization-type": "hvm",
          "name": "ubuntu/images/*ubuntu-bionic-18.04-amd64-server-*",
          "root-device-type": "ebs"
        },
        "owners": ["099720109477"],
        "most_recent": true
      },
      "instance_type": "t2.micro",
      "ssh_username": "ubuntu",
      "ami_name": "eng74-jared-packer-ami-{{timestamp}}"
    }],

//// NEW CODE

  "provisioners": [
    {
      "type": "ansible",
      "playbook_file": "/home/ubuntu/packer-files/app_playbook.yaml"
       }
    ]

/////
}
```

- **Notes:**
    1. To ensure the playbook functions as intended, make note of the `hosts` declaration, if one specifies a `connection: local` within the playbook then the playbook will run on the actual ansible controller NOT on the temporary instance packer creates

- Now that's good, but when we create an instance out of the image the app doesn't necessarily run. Some system states aren't kept so one needs to find a way to fix it:

- Iteration 4:
    - The following new code will save the current pm2 processes and ensures they startup/reload when an instance is created

```json
{
  "variables": {
    "aws_access_key": "{{env `AWS_ACCESS_KEY`}}",
    "aws_secret_key": "{{env `AWS_SECRET_KEY`}}"
  },
  "builders": [{
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "eu-west-1",
      "source_ami_filter": {
        "filters": {
          "virtualization-type": "hvm",
          "name": "ubuntu/images/*ubuntu-bionic-18.04-amd64-server-*",
          "root-device-type": "ebs"
        },
        "owners": ["099720109477"],
        "most_recent": true
      },
      "instance_type": "t2.micro",
      "ssh_username": "ubuntu",
      "ami_name": "eng74-jared-packer-provisioned-ami"
    }],

  "provisioners": [
    {
      "type": "ansible",
      "playbook_file": "./app_playbook.yml"
     },

//// NEW CODE

     {
      "type": "shell",
      "inline": ["sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu", "pm2 save"]
     }
    ]

////
}
```
