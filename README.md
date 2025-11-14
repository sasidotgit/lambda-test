Here is the lambda funtion.py script. 

import json
import boto3
import time

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    s3 = boto3.client('s3')
    region = boto3.session.Session().region_name

    # Define resources
    vpc_cidr = '10.0.0.0/16'
    subnet_cidrs = ['10.0.1.0/24', '10.0.2.0/24']
    bucket_name = "lambda-smart-bucket-" + str(region)

    response_data = {
        'S3_Bucket': None,
        'VPC_ID': None,
        'Subnets': [],
        'IGW_ID': None
    }

    try:
        # ========= 1️⃣ S3 BUCKET =========
        existing_buckets = [b['Name'] for b in s3.list_buckets()['Buckets']]
        if bucket_name in existing_buckets:
            print(f"S3 bucket '{bucket_name}' already exists")
            response_data['S3_Bucket'] = bucket_name
        else:
            print(f"Creating S3 bucket '{bucket_name}'")
            if region == "us-east-1":
                s3.create_bucket(Bucket=bucket_name)
            else:
                s3.create_bucket(
                    Bucket=bucket_name,
                    CreateBucketConfiguration={'LocationConstraint': region}
                )
            response_data['S3_Bucket'] = bucket_name
            print(f"✅ S3 bucket created: {bucket_name}")

        # ========= 2️⃣ VPC =========
        vpcs = ec2.describe_vpcs(Filters=[{'Name': 'cidr', 'Values': [vpc_cidr]}])['Vpcs']
        if vpcs:
            vpc_id = vpcs[0]['VpcId']
            print(f"VPC with CIDR {vpc_cidr} already exists: {vpc_id}")
        else:
            vpc = ec2.create_vpc(CidrBlock=vpc_cidr)
            vpc_id = vpc['Vpc']['VpcId']
            ec2.modify_vpc_attribute(VpcId=vpc_id, EnableDnsSupport={'Value': True})
            ec2.modify_vpc_attribute(VpcId=vpc_id, EnableDnsHostnames={'Value': True})
            print(f"✅ Created VPC: {vpc_id}")
        response_data['VPC_ID'] = vpc_id

        # ========= 3️⃣ SUBNETS =========
        existing_subnets = ec2.describe_subnets(Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}])['Subnets']
        existing_cidrs = [s['CidrBlock'] for s in existing_subnets]

        for cidr in subnet_cidrs:
            if cidr in existing_cidrs:
                subnet_id = [s['SubnetId'] for s in existing_subnets if s['CidrBlock'] == cidr][0]
                print(f"Subnet {cidr} already exists: {subnet_id}")
            else:
                subnet = ec2.create_subnet(VpcId=vpc_id, CidrBlock=cidr)
                subnet_id = subnet['Subnet']['SubnetId']
                print(f"✅ Created Subnet {cidr}: {subnet_id}")
            response_data['Subnets'].append(subnet_id)

        # ========= 4️⃣ INTERNET GATEWAY (IGW) =========
        igws = ec2.describe_internet_gateways(
            Filters=[{'Name': 'attachment.vpc-id', 'Values': [vpc_id]}]
        )['InternetGateways']

        if igws:
            igw_id = igws[0]['InternetGatewayId']
            print(f"Internet Gateway already exists: {igw_id}")
        else:
            igw = ec2.create_internet_gateway()
            igw_id = igw['InternetGateway']['InternetGatewayId']
            ec2.attach_internet_gateway(InternetGatewayId=igw_id, VpcId=vpc_id)
            print(f"✅ Created and attached IGW: {igw_id}")

        response_data['IGW_ID'] = igw_id

        return {
            'statusCode': 200,
            'body': json.dumps(response_data)
        }

    except Exception as e:
        print(f"❌ Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e), 'resources': response_data})
        }


step for import and zip the .py file to create the lambda funtion.

use below command to zip formate to upload you file as .zip formate

Compress-Archive -Path "File SourcePath" -DestinationPath function.zip

Lambda --> Functions --> Create function --> Function name --> Runtime --> pyhton3.XX --> Create Funtion --> click created funtion --> code (Upload From) --> your loacl file path (.Zip File)

Objective Summary

Features

Creates an S3 bucket if it does not exist.

Creates a VPC with a specified CIDR block if it does not exist.

Creates subnets in the VPC for specified CIDR blocks if they do not exist.

Creates and attaches an Internet Gateway (IGW) to the VPC if it does not exist.

How It Works

S3 Bucket

Checks if the bucket exists.

Creates the bucket if it doesn’t exist.

Bucket naming includes the current AWS region to ensure uniqueness.

VPC

Checks if a VPC with the specified CIDR exists.

Creates a new VPC if none exists.

Enables DNS support and DNS hostnames for the VPC.

Subnets

Checks if subnets exist within the VPC.

Creates subnets for CIDRs that are not already present.

Internet Gateway (IGW)

Checks if an IGW is attached to the VPC.

Creates and attaches a new IGW if none exists.

Response

Returns a JSON object containing the IDs of created or existing resources.

If an error occurs, it returns a 500 status with error details.

The final output will be you can access the AWS running service without login the aws console.

https://us-east-1dj8rbmhst.auth.us-east-1.amazoncognito.com/login/continue?client_id=5ue9102fv06gjb58jf7ln8k5dr&redirect_uri=https%3A%2F%2Fbddjwufr8k.execute-api.us-east-1.amazonaws.com%2Fdev%2FPythontest1&response_type=code&scope=email+openid+phone

I will share the user id and password in seperate mail.
