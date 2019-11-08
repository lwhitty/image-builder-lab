# Creating a Red Hat Enterprise Linux (RHEL) VM and Deploying it on Amazon Web Services (AWS)

  

## Here are Instructions on How To:

- Create a RHEL VM in AMI format using Image Builder

- Add additional users and SSH keys

- Install custom packages

- Upload the image to an AWS S3 bucket

- Import the image into EC2

- Launch the image

  

## Two Methods for Accessing Image Builder  

- composer-cli command line tool  (focus of this document)
- RHEL 8 web console GUI
## Image Builder Terminology  
### Blueprint  
- Blueprints define customized system images by listing packages and customizations that will be part of the system  
- Blueprints can be edited and they are versioned  
- When a system image is created from a blueprint, the image is associated with the blueprint in the Image Builder interface of the RHEL 8 web console  
- Blueprints are presented to the user as plain text in the Tom’s Obvious, Minimal Language (TOML) format  
### Compose  
- Composes are individual builds of a system image, based on a particular version of a particular blueprint  
- Compose as a term refers to the system image, the logs from its creation, inputs, metadata, and the process itself  
### Customization  
- Customizations are specifications for the system, which includes users, groups, and SSH keys  
## Image Builder Output Formats  
- QEMU QCOW2 Image  
- Ext4 File System Image  
- Raw Partitioned Disk Image  
- Live Bootable ISO  
- TAR Archive  
- Amazon Machine Image Disk  
- Azure Disk Image  
- VMware Virtual Machine Disk  
- Openstack  

## Other Considerations
- The lorax tool underlying Image Builder performs a number of potentially insecure and unsafe actions while creating the system images, so it is recommended to use a virtual machine to run Image Builder 
- Image Builder is currently available for RHEL 7 and RHEL 8
- The following examples were run on a RHEL 8 VM
- If running on RHEL 7, the "extras" repo must be enabled  

## Install and Configure Image Builder
### Install required packages
```
$ sudo yum install lorax-composer composer-cli cockpit-composer bash-completion
``` 
### Enable Image Builder and cockpit to start after each reboot:  
```
$ sudo systemctl enable cockpit.socket && sudo systemctl start cockpit.socket
$ sudo systemctl enable lorax-composer.socket && sudo systemctl start lorax-composer.socket
``` 
### Configure the system firewall to allow access to the web console:  
```
$ sudo firewall-cmd --add-service=cockpit && sudo firewall-cmd --add-service=cockpit --permanent  
```
### Load the shell configuration script so that the autocomplete feature for the composer-cli command starts working immediately without reboot:  
```
$ sudo su  
# source /etc/bash_completion.d/composer-cli  
```
## Create system images with Image Builder command-line interface:  
**NOTE:** To run the composer-cli command, user must either be root or be in the weldr or root groups  
### Create a plain text file named aws-ami-blueprint.toml with the following contents:  
```
name = "AWS-AMI-Blueprint"  
description = "This is a blueprint for an AWS AMI"  
version = "0.0.1"  
modules = []  
groups = [] 
```
### For every package that you want to be included in the blueprint, add the following lines to the file:  
```
[[packages]]  
name = "package-name"  
version = "package-version" 
```
**NOTES:**  
- For a specific version, use the exact version number such as 8.30  
- For latest available version, use the asterisk *  
- For a latest minor version, use format such as 8.  
- By default, images created by composer have the root account locked and no other users are created  

### To create a root and user account:  
```
[[customizations.user]]  
name = "root"  
password = "$6$DvY8c//GvFTqRSJI$VK5dtx9iMvE4grYPX5.jTEsLbvkpV.kzPwHDQYkIkiZRgmO6VSjZ.RYKDg2VmbGR2tIkCCdkNQQ0gocgoW0CS/"  
  
[[customizations.user]]  
name = "student"  
password = "$6$HiHi0UmitlOcQRZ9$GWe5Y8HMZmN4657w4nMkMhUpyi3CEcLWZTR.27N6z7RH23yxDLgj3veB0RdgIiKEnVV9qk71m15tEX1tjSfLR1"
```
### Here's an example of a completed aws-ami-blueprint.toml file:  
```
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
```
### Push the blueprint:  
```
# composer-cli blueprints push aws-ami-blueprint.toml  
```
### Verify that the blueprint has been pushed:  
```
# composer-cli blueprints list  
```
### Check whether the components and versions listed in the blueprint and their dependencies are valid:  
```
# composer-cli blueprints depsolve AWS-AMI-Blueprint  
```
### If necessary, existing blueprints can be edited and re-pushed:  
```
# composer-cli blueprints save aws-ami-blueprint.toml  
# vi aws-ami-blueprint.toml #(make edits and save)  
# composer-cli blueprints push aws-ami-blueprint.toml 
``` 
### Create (compose) the AMI system image:  
```
# composer-cli compose start AWS-AMI-Blueprint ami 
``` 
### To monitor the status of the compose and to get the image UUID:  
```
# watch composer-cli compose status 
``` 
### To download the finished image file once compose status is FINISHED:  
```
# composer-cli compose image IMAGE_UUID #(replace IMAGE_UUID with IMAGE_UUID value from previous step)  
```
**Note:** Alternatively, you can access the image file directly under the path /var/lib/lorax/composer/results/UUID/  
### Optional: To download logs:  
```
# composer-cli compose logs IMAGE_UUID 
``` 
### Optional: To download metadata:  
```
# composer-cli compose metadata IMAGE_UUID  
```
## Transfer AMI to AWS:  
### Install Python 3 and the pip tool# yum install python3:  
```
# yum install python3 python3-pip
```  
### As non-root user, install the AWS command-line tools with pip:  
```
# exit
$ pip3 install awscli  
```
### Configure the AWS command-line client according to your AWS access details:
**Note:** Access key owner must have **AmazonS3FullAccess** and **VMImportExportRoleForAWSConnector** IAM policies attached
```
$ aws configure  
AWS Access Key ID [None]: <Access Key ID>  
AWS Secret Access Key [None]: <Secret Access Key>  
Default region name [None]: us-east-1 (or your region of choice)
Default output format [None]: json  
```
### Create S3 bucket on AWS if one doesn't already exist  
- Use AWS GUI or CLI to do this
### Upload image, using the image UUID determined in previous steps: 
```
$ AMI=<IMAGE_UUID_FROM_ABOVE> 
$ BUCKET=<BUCKET_NAME_CREATED_ABOVE> 
$ aws s3 cp $AMI s3://$BUCKET  
```
### Import the image as a snapshot into EC2:  
```
$ printf '{ "Description": "RHTE-IB-image", "Format": "raw", "UserBucket": { "S3Bucket": "%s", "S3Key": "%s" } }' $BUCKET $AMI > containers.json  
$ aws ec2 import-snapshot --disk-container file://containers.json 
```
**Note:** If import fails with:
*"An error occurred (InvalidParameter) when calling the ImportSnapshot operation: The service role vmimport does not exist or does not have sufficient permissions for the service to continue"*, follow these instructions for creating and attaching a vmimport role on AWS:

https://docs.aws.amazon.com/vm-import/latest/userguide/vmie_prereqs.html#vmimport-role
  
After the policy is created you can run the import command

### To track progress of the import:  
```
$ aws ec2 describe-import-snapshot-tasks --filters Name=task-state,Values=active  
```
Wait until import completes before moving to next step
### Create an image from the uploaded snapshot by selecting the snapshot in the EC2 console, right clicking on it, select Create Image, and provide a name and a description  
- Select the Virtualization type of Hardware-assisted virtualization in the image you create  
- Start the VM via CLI or web console  
- Select t2.micro machine type (or image type of your choice), then click "Review & Launch"  
- Click "Launch"  
- Select either "Create a key pair" or "Use existing key pair, click the acknowledge box, then select "Launch Instances"  
- Use your private key via SSH to access the resulting EC2 instance  
### Switch to user "student" to verify that your custom user was created in the image build
```
$ su student
password:
```   
### Run tmux command to verify that Image Builder installed additional packages:
```
$ tmux
```   
> Written with [StackEdit](https://stackedit.io/).
