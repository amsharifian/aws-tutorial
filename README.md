# aws-tutorial
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

    source sdk_setup.sh
    source hdk_setup.sh

# Synthesizing the Example with Xilinx Vivado
