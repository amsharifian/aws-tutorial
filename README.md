# AWS Tutorial
A quick tutorial on setting on AWS for CL


# Introduction
This guide includes the instructions included in AWS EC2 FPGA Hardware and Software Development Kit as well as information that I have written by trying out the examples myself. At the time of writing, I could not find such a step-by-step guide and I ran into issues here and there so we think that this guide will allow one to easily try out the F1 instances without getting stuck in some setup issue.

In this guide, I will go over two examples:

1. cl_hello_world: A simple example which writes to some FPGA registers from CPU, then reads them back.
2. cl_dram_dma: DMA data to FPGA DDR4 memories then DMA them back to verify correctness.

The same steps for running the cl_hello_world can be applied to cl_dram_dma example, except that for the cl_dram_dma example, you will need to install DMA drivers at the end before executing the software to communicate with the FPGA. Hence, we will describe all the steps for the cl_hello_world first then have additional instructions at the end for installing the dMA drivers and running the cl_dram_dma example.

This guide is devided into two parts:

1. setting up and synthesizing the example n HDL with Xilinx Vivado
2. Running the example on an Amazon EC2 F1 instance.


# Amazon AWS Setup

**NOTE:** Before starting this tutorial, make sure you can access an F1 instance. If this is the first time you’re trying an FPGA instance with your AWS account, you may need to request an instance limit increase for F1, as the default limit was 0 for our account. The instructions on how to request a limit increase are here. We found this out after spending hours on setting up and synthesizing a design. It took a few business days for the limit increase to be approved.

* Sign into your AWS console

* First, you will need to create and download your private key (.pem file). You will need this file to SSH into an AWS instance. The instructions on creating your key can be found here. Make you save this file in a safe place.

* You will also need to create an access key. This is needed once you’ve SSHed into an instance. To create an access key, go to AWS console, click on the top right corner where it shows your username -> My Security Credentials -> Users -> Click on your user name -> Click on security credentials tab -> Create access key. Once your key has been created, download the access key file, as this is the only time where you can do so. Store it somewhere you can easily find it. If you lose this file, you will have to create a new access key.

* Launch an FPGA Developer AMI instance from the AWS console. The FPGA Developer AMI comes with all the tools that are needed to use the EC2 F1.   
          -> AWS ‘Services’ -> EC2 -> Launch Instance -> Select ‘AWS Marketplace’ and search for FPGA Developer AMI

* Select an instance type. For just playing around, use the t2.micro, which is the cheapest instance ($0.012/hour). For actually synthesizing an FPGA project using Vivado, Amazon recommends the more powerful c4.4xlarge instance ($0.796/hour) or the c4.8xlarge instance ($1.591/hour), as even the simple examples can take hours to synthesize. With the c4.4xlarge instance, it took 1 hour to synthesize the cl_hello_world example, 2 hours 40mins for the cl_dram_dma example. 
          -> Click Review and Launch -> Launch

* Once the instance has been started, SSH into the instance. Click on the “Connect” button to see how to SSH into the instance. Note that the username that shows there is wrong. It should be centos, not root.  (e.g., ssh -i “xxx.pem” centos@ec2-xx-xxx-xxx-xxx.compute-1.amazonaws.com).
Once you’ve SSHed in, run “aws configure”. You will need to enter your Access Key ID and Secret Access Key ID, which are in the access key file you downloaded earlier.
Clone the aws fpga git repo (git clone https://github.com/aws/aws-fpga.git $AWS_FPGA_REPO_DIR), which contains all SDK, HDK, and all the examples. 
CD into the cloned git repo, run and run:

```bash
source sdk_setup.sh
source hdk_setup.sh
```

# Synthesizing the Example with Xilinx Vivado

Change into an example directory and se the CL_DIR environment variable to the path of the example. You will need to set this again if you change examples:

```bash
cd $HDK_DIR/cl/examples/cl_hello_world
export CL_DIR=$(PWD)
```

Verify if Vivado is installed.

```bash
vivado -mode batch
```

Run Vivado synthesis

```bash
cd $CL_DIR/build/scripts
./aws_build-dcp_from_cl.sh
```

Note that this can take a long time. If you want to be notified by email when synthesis is done, do this before running the synthesis:

```bash
pip2 install --user boto3
export EMAIL=your.email@example.com
$HDK_COMMON_DIR/scripts/notify_via_sns.py
```

For the email to work, you need to set your region name properly during "aws configure". We set our region name as "us-east-1".

No you can run synthesis with:

```bash
./aws_build_dcp_from_cl.sh -notify
```

By default, the build runs in the background, but it can be nice to be able to see the synthesis messages to see what's going on. If you want to run it in foreground:

```bash
./aws_build_dcp_from_cl.sh -notify -foreground
```

# Creating an Amazon FPGA Image (AFI)

Now that synthesis is done, we need to create an Amazon FPGA Image (AFI) from the specified design checkpoint (DCP). The AFI contains the FPGA bitstream that will be programmed on the FPGA F1 instance.

* To create an AFI, the DCP must be stored on S3. So we first need to create and s3 bucket. Make sure your credentials are set up correctly for this (aws configure).


```bash
aws s3 mb s3://<bucket-name> --region <region-name> # Create an S3 bucket. Choose a unique bucket name (e.g., aws s3 mb s3://your_awsfpga --region us-east-1
aws s3 mb s3://<bucket-name>/<dcp-folder-name> # Create a folder for your tarball files (e.g.,aws s3 mb s3://your_awsfpga/dcp)
```

Now copy the output files from synthesis to the new s3 bucket.

```bash
aws s3 cp $CL_DIR/build/checkpoints/to_aws/*.Developer_CL.tar s3://<bucket-name>/<dcp-folder-name>/
```

* Create a folder for yor log files

```bash
aws s3 mb s3://<bucket-name>/<logs-folder-name>  # Create a folder to keep your logs
touch LOGS_FILES_GO_HERE.txt                     # Create a temp file
aws s3 cp LOGS_FILES_GO_HERE.txt s3://<bucket-name>/<logs-folder-name>/
```

* Copying to s3 bucket may not work if your s3 bucket policy is not set up properly. To set the bucket polity, go to https://s3.console.aws.amazon.com/ -> Click on your bucket -> Click on Permissions tab -> Click on Bucket Policy.

* Set the policy as listed below, and try copying the files again.

```json
{
    "Version": "2012-10-17",
    "Statement": [
    {
        "Sid": "Bucket level permissions",
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::365015490807:root"
        },
       "Action": [
           "s3:ListBucket"
        ],
       "Resource": "arn:aws:s3:::<bucket-name>"
    },
    {
        "Sid": "Object read permissions",
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::365015490807:root"
        },
        "Action": [
            "s3:GetObject"
        ],
        "Resource": "arn:aws:s3:::<bucket-name>/<dcp-folder-name>/*.tar"
    },
    {
        "Sid": "Folder write permissions",
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::365015490807:root"
        },
        "Action": [
            "s3:PutObject"
        ],
        "Resource": "arn:aws:s3:::<bucket-name>/<logs-folder-name>/*"
    }
    ]
}
```

* Verify that the bucket policy grants the required permissions by running the following script:

```bash
check_s3_bucket_policy.py --dcp-bucket <bucket-name> --dcp-key <dcp-folder-name>/<tar-file-name> --logs-bucket <bucket-name> --logs-key <logs-folder-name>
```

* Once your policy passes the checks, create the Amazon FPGA image (AFI).

```bash
aws ec2 create-fpga-image --name <afi-name> --description <afi-description> --input-storage-location Bucket=<dcp-bucket-name>,Key=<path-to-tarball> --logs-storage-location Bucket=<logs-bucket-name>,Key=<path-to-logs>      
```

    <path-to-tarball> is <dcp-folder-name>/<tar-file-name>

    <path-to-logs> is <logs-folder-name>

The output of this command includes two identifiers that refer to your AFI: Write these down, as you will need them later.

* **FPGA Image Identifier** or **AFI ID**: this is the main ID used to manage your AFI through the AWS EC2 CLI commands and AWS SDK APIs.

This ID is regional, i.e., if an AFI is copied across multiple regions, it will have a different unique AFI ID in each region.  An example AFI ID is **`afi-06d0ffc989feeea2a`**.

* **Glogal FPGA Image Identifier** or **AGFI ID**: this is a global ID that is used to refer to an AFI from within an F1 instance. For example, to load or clear an AFI from an FPGA slot, you use the AGFI ID.

Since the AGFI IDs is global (by design), it allows you to copy a combination of AFI/AMI to multiple regions, and they will work without requiring any extra setup. An example AGFI ID is **`agfi-0f0e045f919413242`**.

* Check if the AFI generation is done. You must provide the **FPGA Image Identifier** returned by `create-fpga-image`:

```bash
aws ec2 describe-fpga-images --fpga-image-ids <AFI ID>
```

The AFI can only be loaded to an instance once the AFI generation completes and the AFI state is set to `available`. This can also take some time (Took ~30 minutes for the cl_dram_dma example).

```json
{
    "FpgaImages": [
    {
        ...
        "State": {
            "Code": "available"
        },<
        ...
        "FpgaImageId": "afi-06d0ffc989feeea2a",
        ...
    }
    ]
}
```

* Once you have gotten to this point, you have successfully synthesized an HDL design for the EC2 F1. Now you’re ready to program the FPGA and run the example.

* Go to the EC2 Management Console from AWS console and stop your EC2 instance.