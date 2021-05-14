# AWS Kinesis Video Streams on a Raspberry Pi with AWS IoT

## Overview

This script sets up AWS Kinesis Video Streams on a Raspberry Pi and sets up a startup script to load the KVS stream on startup.

### Camera Options

This script works with the official raspberry pi camera or an external USB camera plugged into the Raspberry Pi.

## AWS CLI

### Installing the AWS CLI

The Raspberry Pi supports the AWS CLI v1. Install it with:

```bash
sudo apt-get install awscli
```

### Configuring the AWS CLI

```bash
aws configure --profile iot-profile
```

Enter the IAM access keys, secret keys, region, and output type.

## AWS IoT

### Step 1: Create an IoT Thing Type and an IoT Thing

Register your Raspberry Pi in the AWS IoT thing registry database by creating a thing type and a thing.

```bash
aws --profile iot-profile iot create-thing-type --thing-type-name raspberry-pi-type > iot-thing-type.json
```

Run the following command in the AWS CLI to create a thing.

```bash
aws --profile iot-profile iot create-thing --thing-name raspberry-pi-alpha --thing-type-name raspberry-pi-type --attribute-payload "{\"attributes\": {\"owner\":\"my-name\",\"project\":\"my-project\"}}" > iot-thing.json
```

### Step 2: Create an IAM Role to be Assumed by IoT

#### Create a trust policy

Create a trust policy file that grants the credentials provider permission to assume the role.

```json
cat > iam-policy-document.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": "credentials.iot.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
}
EOF
```

Run the following command in the AWS CLI to create an IAM role with the preceding trust policy.

```bash
aws --profile iot-profile iam create-role --role-name kinesisvideo-iam-role --assume-role-policy-document iam-policy-document.json > iam-role.json
```

#### Create a permissions policy

Create an access policy file that grants Kinesis Video Streams operations. This policy authorizes the specified actions only on a video stream (AWS resource) that is specified by the placeholder (${credentials-iot:ThingName}). This placeholder takes on the value of the IoT thing attribute ThingName when the IoT credentials provider sends the video stream name in the request. Copy the json policy below.

```json
cat > iam-permisson-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kinesisvideo:DescribeStream",
                "kinesisvideo:PutMedia",
                "kinesisvideo:TagStream",
                "kinesisvideo:GetDataEndpoint"
            ],
            "Resource": "arn:aws:kinesisvideo:*:*:stream/\${credentials-iot:ThingName}/*"
        }
    ]
}
EOF
```

Run the following command in the AWS CLI to create the access policy.

```bash
aws --profile iot-profile iam put-role-policy --role-name kinesisvideo-iam-role --policy-name kinesisvideo-iam-policy --policy-document iam-permisson-policy.json 
```

#### Create a role alias

Create a Role Alias for your IAM Role. An IoT credendials provider request must include a role-alias to indicate which IAM role to assume in order to obtain temporary credentials.

```bash
aws --profile iot-profile iot create-role-alias --role-alias kinesisvideo-role-alias --role-arn $(jq --raw-output '.Role.Arn' iam-role.json) --credential-duration-seconds 3600 > iot-role-alias.json
```

Create a policy document that will enable AWS IoT to assume role with the X.509 certificate (once it is attached) using the role alias.

```bash
cat > iot-policy-document.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
	 "Effect":"Allow",
	 "Action":[
	    "iot:Connect"
	 ],
	 "Resource":"$(jq --raw-output '.roleAliasArn' iot-role-alias.json)"
 },
      {
	 "Effect":"Allow",
	 "Action":[
	    "iot:AssumeRoleWithCertificate"
	 ],
	 "Resource":"$(jq --raw-output '.roleAliasArn' iot-role-alias.json)"
 }
   ]
}
EOF
```

Create the policy in AWS IoT.

```bash
aws --profile iot-profile iot create-policy --policy-name kinesisvideo-iot-policy --policy-document iot-policy-document.json
```

## Step 3: Create and Register the X.509 certificate

The following create-keys-and-certificate creates a 2048-bit RSA key pair and issues an X.509 certificate using the issued public key. Because this is the only time that AWS IoT provides the private key for this certificate, be sure to keep it in a secure location. For documentation see (Create AWS IoT client certificates)[https://docs.aws.amazon.com/iot/latest/developerguide/device-certs-create.html]

```bash
aws --profile iot-profile iot create-keys-and-certificate --set-as-active --certificate-pem-outfile certificate.pem --public-key-outfile public.pem.key --private-key-outfile private.pem.key > certificate.json    
```

Attach the policy for IoT to this certificate.

```bash
aws --profile iot-profile iot attach-policy --policy-name kinesisvideo-iot-policy --target $(jq --raw-output '.certificateArn' certificate.json)        
```

Attach your IoT thing to the certificate you just created.

```bash
aws --profile iot-profile iot attach-thing-principal --thing-name raspberry-pi-alpha --principal $(jq --raw-output '.certificateArn' certificate.json)
```

Get the IoT credentials endpoint unique to your AWS account ID.

```bash
aws --profile iot-profile iot describe-endpoint --endpoint-type iot:CredentialProvider --output text > iot-credential-provider.txt
```

Get the CA certificate to establish trust with the back-end service through TLS.

```bash
curl --silent 'https://www.amazontrust.com/repository/SFSRootCAG2.pem' --output cacert.pem
```





## Build the AWS Kinesis Video Streams SDK

```bash
git clone https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp.git /tmp/amazon-kinesis-video-streams-producer-sdk
mkdir /usr/local/amazon-kinesis-video-streams-producer-sdk-cpp
cd /usr/local/amazon-kinesis-video-streams-producer-sdk-cpp
cmake /tmp/amazon-kinesis-video-streams-producer-sdk -DBUILD_GSTREAMER_PLUGIN=ON -DBUILD_DEPENDENCIES=OFF
make -j4
```

## Run GStreamer

### Environmental Variables

#### GStreamer Environmental Variables

These environmental variables are needed to start GStreamer.

```bash
export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
```

#### AWS Environmental Variables

You have two options for authentication, using AWS access key and secret access key, or using AWS certificates.

* When using IoT authorization, the value of stream-name must be identical to the value of iot-thingname (in IoT provisioning). *

* storage-size: The storage size of the device in megabytes.
* access-key: The AWS access key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
* secret-key: The AWS secret key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
* credential-path: A path to a file containing your credentials for accessing Kinesis Video Streams. You must provide either this parameter or access-key and secret-key.

Environmental variables with access keys:

```bash
export STREAM_NAME="[--insert-unique-stream-name--]"
export AWS_ACCESS_KEY="[--insert-access-key--]"
export AWS_SECRET_KEY="[--insert-secret-key--]"
export AWS_REGION="[--insert-aws-region-code--]"
```

Test that it is working with:

```
gst-inspect-1.0 kvssink
```

This commands send GStreamer at 640x420 resolution.

```
gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=640,height=480 \
! omxh264enc control-rate=2 target-bitrate=512000 periodicity-idr=45 inline-header=FALSE \
! h264parse ! video/x-h264,stream-format=avc,alignment=au,profile=baseline \
! kvssink
    stream-name="$STREAM_NAME" \
    storage-size=128 \
    access-key="$AWS_ACCESS_KEY" \
    secret-key="$AWS_SECRET_KEY" \
    aws-region="$AWS_REGION"
```

This commands send GStreamer at 1280x720 resolution.

```
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,format=I420,width=1280,height=720,framerate=10/1 \
! omxh264enc control-rate=1 target-bitrate=8024000 inline-header=FALSE periodicty-idr=4 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,width=1280,height=720,framerate=10/1,profile=baseline \
! kvssink \
    stream-name="$STREAM_NAME" \
    storage-size=128 \
    access-key="$AWS_ACCESS_KEY" \
    secret-key="$AWS_SECRET_KEY" \
    aws-region="$AWS_REGION"
```

### Run GStreamer on Pi Startup

#### Configure KVS logging

```
nano /home/pi/kvs_log_configuration 
```

Copy the text below to configure the KVS log configuration.

```text
log4cplus.rootLogger=ERROR, KvsConsoleAppender

#log4cplus.logger.MemoryCheck=TRACE, KvsConsoleAppender

#KvsConsoleAppender:
log4cplus.appender.KvsConsoleAppender=log4cplus::ConsoleAppender
log4cplus.appender.KvsConsoleAppender.layout=log4cplus::PatternLayout
log4cplus.appender.KvsConsoleAppender.layout.ConversionPattern=[%-5p][%d] %m%n

#KvsFileAppender
log4cplus.appender.KvsFileAppender=log4cplus::DailyRollingFileAppender
log4cplus.appender.KvsFileAppender.File=./log/kvs.log
log4cplus.appender.KvsFileAppender.Schedule=HOURLY
log4cplus.appender.KvsFileAppender.CreateDirs=true
log4cplus.appender.KvsFileAppender.layout=log4cplus::PatternLayout
log4cplus.appender.KvsFileAppender.layout.ConversionPattern=[%D]-%p-%m%n
```

### Raspberry Pi startup script

#### Configure startup script

```bash
sudo nano /usr/local/bin/kinesisvideo.sh
```

Copy the startup script below.

```text
#!/bin/bash
# Store first parameter in a variable, which should be the log file location.
LOG_FILE="$1"

# Set a default log file location if the parameter was empty, i.e. not specified.
if [ -z "$LOG_FILE" ]
then
  LOG_FILE="/home/pi/kinesisvideo.log"
fi

# Append information to the log file.
echo "----------------------------------------" >> "$LOG_FILE"
echo "System date and time: $(date '+%d/%m/%Y %H:%M:%S')" >> "$LOG_FILE"
echo "Kernel info: $(uname -rmv)" >> "$LOG_FILE"

export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
export STREAM_NAME="[--insert-unique-stream-name--]"
export AWS_ACCESS_KEY="[--insert-access-key--]"
export AWS_SECRET_KEY="[--insert-secret-key--]"
export AWS_REGION="[--insert-aws-region-code--]"

gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,format=I420,width=1280,height=720,framerate=10/1 \
! omxh264enc control-rate=1 target-bitrate=8024000 inline-header=FALSE periodicty-idr=4 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,width=1280,height=720,framerate=10/1,profile=baseline \
! kvssink \
    stream-name="$STREAM_NAME" \
    storage-size=128 \
    access-key="$AWS_ACCESS_KEY" \
    secret-key="$AWS_SECRET_KEY" \
    aws-region="$AWS_REGION" &
```

Make the script executable with:

```bash
sudo chmod +x /usr/local/bin/kinesisvideo.sh
```

Test the script with:

```bash
sudo sh /usr/local/bin/kinesisvideo.sh
```

Check that the KVS stream is appearing in the AWS management console.

Kill the script with

```bash
ps - ef
sudo kill [--insert pid # of th
```

#### Run the script at startup as the root user

If the script is working OK, then create a .service file to load the script on the Pi bootup.

```bash
sudo nano /etc/systemd/system/kinesisvideo.service
```

Copy the text below:

```bash
[Unit]
Description=Systemd service for starting AWS KVS at startup

[Service]
User=pi
ExecStart=/usr/local/bin/testscript.sh /home/pi/kinesisvideo.log

[Install]
WantedBy=default.target
```

To enable the service with Systemd, run the command:

```bash
sudo systemctl enable kinesisvideo.service
```

Reboot the Raspberry PI to test that Systemd actually executed the script during system startup:

```bash
sudo reboot now
```

Check the script output logs:

```bash
cat /home/pi/kinesisvideo.log
```

Check the status of the service:

```bash
systemctl status kinesisvideo.service
```

Disable the service:

```bash
sudo systemctl disable kinesisvideo.service
```


