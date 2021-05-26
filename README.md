# Amazon Kinesis Video Streams on a Raspberry Pi with AWS IoT

## Overview

This script sets up Amazon Kinesis Video Streams on a Raspberry Pi and sets up a startup script to load the KVS stream on startup.

## Prerequisites

### AWS Prerequisites

This script requires that you have an AWS account and permissions to create a new [Identity and Access Management (IAM)] user.

### Raspberry Pi Prerequisites

This script requires that the Raspberry Pi (3B or 4) has:
* complete [Setting up your Raspberry Pi](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up) - [Raspberry Pi OS](https://www.raspberrypi.org/software/) is the Raspberry Pi's official supported operating system.
* [SSH (Secure Shell)](https://www.raspberrypi.org/documentation/remote-access/ssh/) access
* a camera which can be the official [Raspberry Pi Camera Module](https://projects.raspberrypi.org/en/projects/getting-started-with-picamera) or just plug in an external USB web camera

Connect a camera and ssh login to your Raspberry Pi to get started.

### AWS IAM

An IAM user with programatic access is required to use the AWS CLI in your Raspberry Pi.

* Open [Identity and Access Management (IAM)](https://console.aws.amazon.com/iam/home) in the AWS management console.
* Select *Users*
* Select *Add User*
* Enter a user name *raspberry-pi-iot-profile*
* Select only the access type *Programmatic access*
* Select *Next: Permissions*
* Select *Attach existing policies directly*
* Check the box next to the following policies, you can use the policy filter to find each one:
  * AWSIoTFullAccess
  * IAMFullAccess
  * AmazonKinesisVideoStreamsFullAccess
* Select *Next: Tags*
* Enter Key: *project*, Value: *raspberry-pi-camera-project*
* Select *Next: Review*
* Select *Create user*
* Select *Download .csv*. The .csv file contains the user access key ID and secret access key. You can also copy and paste them to your notepad now. You will need these files to setup the AWS CLI.

## AWS CLI

### Installing the AWS CLI

The Raspberry Pi supports the AWS CLI v1. Install it with:

```bash
sudo apt-get install awscli
```

Verify that the AWS CLI installed correctly.

```bash
aws --version
```

### Configuring the AWS CLI

```bash
aws configure --profile iot-profile
```
Enter the IAM access keys, secret keys, region, and output type. For help configuring see [Configuration basics](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)

* Enter AWS Access Key ID
* Enter AWS Secret Access Key
* Enter Default region name, e.g. *ap-southeast-2*
* Default output format *json*

## AWS IoT

### Step 1: Create an IoT Thing Type and an IoT Thing

Register your Raspberry Pi in the AWS IoT thing registry database by creating a thing type and a thing.

```bash
aws --profile iot-profile iot create-thing-type --thing-type-name raspberry-pi-type > iot-thing-type.json
```

Run the following command in the AWS CLI to create a thing.

```bash
aws --profile iot-profile iot create-thing --thing-name raspberry-pi-camera-stream --thing-type-name raspberry-pi-camera --attribute-payload "{\"attributes\": {\"computer-type\":\"raspberry-pi\",\"project\":\"raspberry-pi-camera-project\"}}" > iot-thing.json
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

The following create-keys-and-certificate creates a 2048-bit RSA key pair and issues an X.509 certificate using the issued public key. Because this is the only time that AWS IoT provides the private key for this certificate, be sure to keep it in a secure location.

```bash
aws --profile iot-profile iot create-keys-and-certificate --set-as-active --certificate-pem-outfile certificate.pem --public-key-outfile public.pem.key --private-key-outfile private.pem.key > certificate.json    
```

Attach the policy for IoT to this certificate.

```bash
aws --profile iot-profile iot attach-policy --policy-name kinesisvideo-iot-policy --target $(jq --raw-output '.certificateArn' certificate.json)        
```

Attach your IoT thing to the certificate you just created.

```bash
aws --profile iot-profile iot attach-thing-principal --thing-name raspberry-pi-camera-stream --principal $(jq --raw-output '.certificateArn' certificate.json)
```

Get the IoT credentials endpoint unique to your AWS account ID.

```bash
aws --profile iot-profile iot describe-endpoint --endpoint-type iot:CredentialProvider > iot-credential-provider.json
```

Get the CA certificate to establish trust with the back-end service through TLS.

```bash
curl --silent 'https://www.amazontrust.com/repository/SFSRootCAG2.pem' --output cacert.pem
```

## Step 4: Test the IoT Credentials with Your Kinesis Video Stream 

Create a Kinesis video stream that you want to test this configuration with. The video stream name must be identical to the IoT thing name that you created in the previous step.

```bash
aws --profile iot-profile kinesisvideo create-stream --data-retention-in-hours 24 --stream-name raspberry-pi-camera-stream
```

Call the IoT credentials provider to get the temporary credentials:

```bash
curl --silent -H "x-amzn-iot-thingname:raspberry-pi-camera-stream" --cert certificate.pem --key private.pem.key https://$(jq --raw-output '.endpointAddress' iot-credential-provider.json)/role-aliases/kinesisvideo-role-alias/credentials --cacert cacert.pem > token.json
```

Invoke the Kinesis Video Streams DescribeStream API for the sample kvs_example_camera_stream video stream

```bash
AWS_ACCESS_KEY_ID=$(jq --raw-output '.credentials.accessKeyId' token.json) AWS_SECRET_ACCESS_KEY=$(jq --raw-output '.credentials.secretAccessKey' token.json) AWS_SESSION_TOKEN=$(jq --raw-output '.credentials.sessionToken' token.json) aws kinesisvideo describe-stream --stream-name raspberry-pi-camera-stream
```

## Step 5: Build the AWS Kinesis Video Streams SDK

```bash
git clone https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp.git /tmp/amazon-kinesis-video-streams-producer-sdk
sudo mkdir /usr/local/amazon-kinesis-video-streams-producer-sdk-cpp
cd /usr/local/amazon-kinesis-video-streams-producer-sdk-cpp
sudo cmake /tmp/amazon-kinesis-video-streams-producer-sdk -DBUILD_GSTREAMER_PLUGIN=ON -DBUILD_DEPENDENCIES=OFF
sudo make -j4
```

### Environmental Variables

These environmental variables are needed to start GStreamer.

```bash
export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
```

You have two options for authentication, using AWS access key and secret access key, or using AWS certificates.

* When using IoT authorization, the value of stream-name must be identical to the value of iot-thingname (in IoT provisioning).
* storage-size: The storage size of the device in megabytes.
* access-key: The AWS access key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
* secret-key: The AWS secret key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
* credential-path: A path to a file containing your credentials for accessing Kinesis Video Streams. You must provide either this parameter or access-key and secret-key.

Environmental variables with access keys:

```bash
export STREAM_NAME="raspberry-pi-camera-stream"
export AWS_ACCESS_KEY="[--insert-access-key--]"
export AWS_SECRET_KEY="[--insert-secret-key--]"
export AWS_REGION="[--insert-aws-region-code--]"
```

Test that it is working with:

```
gst-inspect-1.0 kvssink
```

If the build failed, or GST_PLUGIN_PATH is not properly set you will get output like

```text
No such element or plugin 'kvssink'
```

### Run GStreamer

This commands send GStreamer at 640x420 resolution using the AWS access key and secret key.

```
gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=640,height=480 \
! omxh264enc control-rate=2 target-bitrate=512000 periodicity-idr=45 inline-header=FALSE \
! h264parse ! video/x-h264,stream-format=avc,alignment=au,profile=baseline \
! kvssink \
    stream-name="raspberry-pi-camera-stream" \
    aws-region="$AWS_REGION"
    storage-size=128 \
    access-key="$AWS_ACCESS_KEY" \
    secret-key="$AWS_SECRET_KEY" \

```

This commands send GStreamer at 640x420 resolution using the AWS IoT credentials.

```bash
gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=640,height=480 \
! omxh264enc control-rate=2 target-bitrate=512000 periodicity-idr=45 inline-header=FALSE \
! h264parse ! video/x-h264,stream-format=avc,alignment=au,profile=baseline \
! kvssink \
    stream-name="raspberry-pi-camera-stream" \
    aws-region="$AWS_REGION"
    storage-size=128 \
    iot-certificate="iot-certificate,endpoint=iot-credential-endpoint-host-name,cert-path=/home/pi/certificate.pem,key-path=/home/pi/private.pem.key,ca-path=/home/pi/cacert.pem,role-aliases=kinesisvideo-role-alias"
```

This commands send GStreamer at 1280x720 resolution.

```bash
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,format=I420,width=1280,height=720,framerate=10/1 \
! omxh264enc control-rate=1 target-bitrate=8024000 inline-header=FALSE periodicty-idr=4 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,width=1280,height=720,framerate=10/1,profile=baseline \
! kvssink \
    stream-name="raspberry-pi-camera-stream" \
    aws-region="$AWS_REGION"
    storage-size=128 \
    iot-certificate="iot-certificate,endpoint=iot-credential-endpoint-host-name,cert-path=/home/pi/certificate.pem,key-path=/home/pi/private.pem.key,ca-path=/home/pi/cacert.pem,role-aliases=kinesisvideo-role-alias"
```

## Amazon Kinesis Video Streams

### View video stream

* Open [Amazon Kinesis Video Streams](https://console.aws.amazon.com/kinesisvideo/home#/dashboard) in the AWS management console
* Select your AWS region
* Choose *Video streams*
* Select your stream name *raspberry-pi-camera-stream*
* Expand the media playback section to watch the video stream

### Download media clip

* In the media playback section, choose *Download clip*
* Select the Timestamp source
* Select the Time range clip duration or start and end time
* Select the clip duration
* Choose Download

### Download programmatically

In your Raspberry Pi, you can get the data endpoint for your video stream:

```bash
aws --profile iot-profile kinesisvideo get-data-endpoint --stream-name raspberry-pi-camera-stream --api-name GET_CLIP > kvs-clip.json
```

Download a clip with the AWS CLI

```bash
aws --profile iot-profile kinesis-video-archived-media get-clip \
  --stream-name raspberry-pi-camera-stream \
  --clip-fragment-selector "FragmentSelectorType=SERVER_TIMESTAMP,TimestampRange={StartTimestamp=$(date -u '+%FT%T.%S0Z' -d '5 mins ago'),EndTimestamp=$(date -u '+%FT%T.%S0Z')}" \
  raspberry-pi-camera-stream.mp4
```

### HLS Streaming

The data end point for your video stream:

```bash
aws --profile iot-profile kinesisvideo get-data-endpoint --stream-name raspberry-pi-camera-stream --api-name GET_HLS_STREAMING_SESSION_URL > hls-session-url.json
```

The HLS streaming session URL for your video stream from the data end-point:

```bash
aws --profile iot-profile kinesis-video-archived-media get-hls-streaming-session-url --endpoint-url $(jq --raw-output '.DataEndpoint' hls-session-url.json) \
--stream-name raspberry-pi-camera-stream \
--playback-mode LIVE > hls-streaming-session-url.json
```

The output from this request is live HLS video stream that you can open in a web browser. Create a hls.js link:

```bash
https://hls-js.netlify.app/demo/?src=$(jq --raw-output '.HLSStreamingSessionURL' hls-streaming-session-url.json)
```

## Run GStreamer on Pi Startup

### Setup Environmental variables

Set the required environmental variables on Pi startup:

```json
sudo cat > /etc/profile.d/aws-kvs.sh <<EOF
export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
export STREAM_NAME="raspberry-pi-camera-stream"
export AWS_REGION="ap-southeast-2"
EOF
```

Reboot the pi and test with:

```bash
echo %GST_PLUGIN_PATH
echo %LD_LIBRARY_PATH
echo %STREAM_NAME
```

### Configure KVS logging

Copy the text below to configure the KVS log configuration.

```bash
cat > /home/pi/kvs_log_configuration <<EOF
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
EOF
```

### Raspberry Pi startup script

#### Configure startup script

Create the startup script below.

```bash
cat > /usr/local/bin/kinesisvideo.sh <<EOF
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
export STREAM_NAME="raspberry-pi-camera-stream"

gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,format=I420,width=1280,height=720,framerate=10/1 \
! omxh264enc control-rate=1 target-bitrate=8024000 inline-header=FALSE periodicty-idr=4 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,width=1280,height=720,framerate=10/1,profile=baseline \
! kvssink \
    stream-name="kvs_example_camera_stream" \
    iot-certificate="iot-certificate,endpoint=iot-credential-endpoint-host-name,cert-path=/home/pi/certificate.pem,key-path=/home/pi/private.pem.key,ca-path=/home/pi/cacert.pem,role-aliases=kinesisvideo-role-alias" &
EOF
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

## Resources

* [Controlling Access to Kinesis Video Streams Resources Using AWS IoT](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/how-iot.html)
* [How to Eliminate the Need for Hardcoded AWS Credentials in Devices by Using the AWS IoT Credentials Provider](https://aws.amazon.com/blogs/security/how-to-eliminate-the-need-for-hardcoded-aws-credentials-in-devices-by-using-the-aws-iot-credentials-provider/)
* [Manage Raspberry Pi devices using AWS Systems Manager](https://aws.amazon.com/blogs/mt/manage-raspberry-pi-devices-using-aws-systems-manager/)
* [GStreamer Element Parameter Reference](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/examples-gstreamer-plugin-parameters.html)
* [ROS Kinesis Service Common Library](https://github.com/aws-robotics/kinesisvideo-common)
* [GStreamer](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c)
* [Amazon Kinesis Video Streams CPP Producer, GStreamer Plugin and JNI](https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp/)
* [Using Docker images for Producer SDK (CPP and GStreamer plugin)](https://github.com/aws-samples/amazon-kinesis-video-streams-demos/tree/master/producer-cpp/docker-raspberry-pi)

