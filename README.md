# image-builder-lab
Image Builder Lab
Prepare Image Builder session for RHTE RHEL Hackathon #project
Resources:
Planning doc: https://docs.google.com/document/d/1k-NB4D2OlJx2-dKZaWWUblCARBcOOTmu5OS1wkGi3aA/edit#
Brian Smith video to use as a guide: https://www.youtube.com/watch?v=UopGqYs0PKA
Image builder documentation: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/composing_a_customized_rhel_system_image/index
Create custom Image Blueprints using various customizations:
Users
SSH keys
Custom packages
Upload images to EC2
Launch instances using custom images
Make list of feedback to collect (examples in Hackathon doc)
Methods of accessing Image Builder:
composer-cli command line tool
RHEL 8 web console GUI
Image Builder runs as the system service lorax-composer
Image Builder terminology:
Blueprint
Blueprints define customized system images by listing packages and customizations that will be part of the system
Blueprints can be edited and they are versioned
When a system image is created from a blueprint, the image is associated with the blueprint in the Image Builder interface of the RHEL 8 web console
Blueprints are presented to the user as plain text in the Tom’s Obvious, Minimal Language (TOML) format
Compose
Composes are individual builds of a system image, based on a particular version of a particular blueprint
Compose as a term refers to the system image, the logs from its creation, inputs, metadata, and the process itself
Customization
Customizations are specifications for the system, which includes users, groups, and SSH keys
Image Builder output formats
QEMU QCOW2 Image
Ext4 File System Image
Raw Partitioned Disk Image
Live Bootable ISO
TAR Archive
Amazon Machine Image Disk
Azure Disk Image
VMware Virtual Machine Disk
Openstack
The lorax tool underlying Image Builder performs a number of potentially insecure and unsafe actions while creating the system images
For this reason, use a virtual machine to run Image Builder
Install Image Builder:
$ sudo yum install lorax-composer composer-cli cockpit-composer bash-completion
NOTE: The web console is installed as a dependency of the cockpit-composer package
NOTE: If running this on an AWS AMI, also install, enable, and start firewalld
Enable Image Builder and cockpit to start after each reboot:
$ sudo systemctl enable cockpit.socket
$ sudo systemctl enable lorax-composer.socket
NOTE: The lorax-composer and cockpit services start automatically on first access (???)
Configure the system firewall to allow access to the web console:
$ sudo firewall-cmd --add-service=cockpit && sudo firewall-cmd --add-service=cockpit --permanent
Load the shell configuration script so that the autocomplete feature for the composer-cli command starts working immediately without reboot:
$ sudo su
# source  /etc/bash_completion.d/composer-cli
Creating system images with Image Builder command-line interface:
Workflow:
Export (save) the blueprint definition to a plain text file
Edit this file in a text editor
Import (push) the blueprint text file back into Image Builder
Run a compose to build an image from the blueprint
NOTE: To run the composer-cli command, user must be in the weldr or root groups
Export the image file to download it
Procedure:
Create a plain text file named aws-ami-blueprint.toml with the following contents:
name = "AWS-AMI-Blueprint"
description = "This is a blueprint for an AWS AMI"
version = "0.0.1"
modules = []
groups = []
For every package that you want to be included in the blueprint, add the following lines to the file:
[[packages]]
name = "package-name"
version = "package-version"
NOTES:
For a specific version, use the exact version number such as 8.30
For latest available version, use the asterisk *
For a latest minor version, use format such as 8.
By default, images created by composer have the root account locked and no other users are created
To create a root and user account:
[[customizations.user]]
name = "root"
password = "$6$DvY8c//GvFTqRSJI$VK5dtx9iMvE4grYPX5.jTEsLbvkpV.kzPwHDQYkIkiZRgmO6VSjZ.RYKDg2VmbGR2tIkCCdkNQQ0gocgoW0CS/"

[[customizations.user]]
name = "student"
password = "$6$HiHi0UmitlOcQRZ9$GWe5Y8HMZmN4657w4nMkMhUpyi3CEcLWZTR.27N6z7RH23yxDLgj3veB0RdgIiKEnVV9qk71m15tEX1tjSfLR1"
Here's an example of a completed aws-ami-blueprint.toml file:
name = "AWS-AMI-Blueprint"
description = "This is a blueprint for an AWS AMI"
version = "0.0.1"
modules = []
groups = []

[[packages]]
name = "httpd"
version = "*"

[[packages]]
name = "cockpit"
version = "*"

[[packages]]
name = "tmux"
version = "*"

[[customizations.user]]
name = "root"
password = "$6$DvY8c//GvFTqRSJI$VK5dtx9iMvE4grYPX5.jTEsLbvkpV.kzPwHDQYkIkiZRgmO6VSjZ.RYKDg2VmbGR2tIkCCdkNQQ0gocgoW0CS/"

[[customizations.user]]
name = "student"
password = "$6$HiHi0UmitlOcQRZ9$GWe5Y8HMZmN4657w4nMkMhUpyi3CEcLWZTR.27N6z7RH23yxDLgj3veB0RdgIiKEnVV9qk71m15tEX1tjSfLR1"
Push the blueprint:
# composer-cli blueprints push aws-ami-blueprint.toml
Verify that the blueprint has been pushed:
# composer-cli blueprints list
Check whether the components and versions listed in the blueprint and their dependencies are valid:
# composer-cli blueprints depsolve aws-ami-blueprint.toml
If necessary, existing blueprints can be edited and re-pushed:
# composer-cli blueprints save aws-ami-blueprint.toml
# vi aws-ami-blueprint.toml (make edits and save)
# composer-cli blueprints push aws-ami-blueprint.toml
Create (compose) the AMI system image:
# composer-cli compose start aws-ami-blueprint.toml ami
To list possible image types:
# composer-cli compose types
To check the status of the compose and to get the UUID:
# composer-cli compose status
To download the finished image file:
# composer-cli compose image UUID #(replace UUID with UUID value from previous step)
Optional:
Alternatively, you can access the image file directly under the path /var/lib/lorax/composer/results/UUID/
To download logs:
# composer-cli compose logs UUID
To download metadata:
# composer-cli compose metadata UUID
Transfer AMI to AWS:
Install Python 3 and the pip tool# yum install python3:
# yum install python3 python3-pip
Install the AWS command-line tools with pip:
# pip3 install awscli
Configure the AWS command-line client according to your AWS access details:
$ aws configure
AWS Access Key ID [None]: <Access Key ID>
AWS Secret Access Key [None]: <Secret Access Key>
Default region name [None]: us-east-1
Default output format [None]: json
Create S3 bucket if one doesn't already exist
Use AWS credentials and GUI to do this
For this lab:
Bucketname is ib-rhte.redhat.com
Make bucket public
Upload image:
$ AMI=7008b4e6-59cb-405b-a8da-97a833f8b553-disk.ami
$ BUCKET=ib-rhte.redhat.com
$ aws s3 cp $AMI s3://$BUCKET
Import the image as a snapshot into EC2:
$ printf '[{ "Description": "RHTE-IB-image", "Format": "raw", "UserBucket": { "S3Bucket": "%s", "S3Key": "%s" } }]' $BUCKET $AMI > containers.json
$ aws ec2 import-snapshot --disk-container file://containers.json
If the above step fails with an error that the service role <vmimport> does not exist, this role must be created on AWS: 
https://bee42.com/de/blog/tutorials/linuxkit-with-initial-aws-support/ (Follow directions in step 4)
To track progress of the import:
$ aws ec2 describe-import-snapshot-tasks --filters Name=task-state,Values=active
Create an image from the uploaded snapshot by selecting the snapshot in the EC2 console, right clicking on it, select Create Image, and provide a name and a description
Select the Virtualization type of Hardware-assisted virtualization in the image you create
Start the VM via CLI or web console
Select t2.micro machine type, then click "Review & Launch"
Click "Launch"
Select "Proceed without a key pair, click the acknowledge box, then select "Launch Instances"
Use your private key via SSH to access the resulting EC2 instance
Log in as ec2-user
