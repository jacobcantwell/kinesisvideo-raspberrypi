# Amazon Kinesis Video Streams with AWS IoT

## Overview

This script sets up Amazon Kinesis Video Streams on a Ubuntu computer and sets up a startup script to load the KVS stream on startup. 

It has been tested with:
* Raspberry Pi 3 Model B running Raspian OS
* Raspberry Pi 4 Model B running Raspian OS
* Onlogic Fanless Industrial USFF Edge Device running Ubuntu 18

## Prerequisites

### AWS Prerequisites

This script requires that you have an AWS account and permissions to create a new [Identity and Access Management (IAM)] user.

This script requires that your device has:
* Ubuntu based OS [Setting up your Raspberry Pi](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up) - [Raspberry Pi OS](https://www.raspberrypi.org/software/) is the Raspberry Pi's official supported operating system.
* [SSH (Secure Shell)](https://www.raspberrypi.org/documentation/remote-access/ssh/) access
* a camera - either an external USB web camera or the official [Raspberry Pi Camera Module](https://projects.raspberrypi.org/en/projects/getting-started-with-picamera)

Connect a camera and ssh login to your edge device to get started.

### AWS IAM

An IAM user with programatic access is required to use the AWS CLI.

* Open [Identity and Access Management (IAM)](https://console.aws.amazon.com/iam/home) in the AWS management console.
* Select *Users*
* Select *Add User*
* Enter a user name *iot-profile*
* Select only the access type *Programmatic access*
* Select *Next: Permissions*
* Select *Attach existing policies directly*
* Check the box next to the following policies, you can use the policy filter to find each one:
  * AWSIoTFullAccess
  * IAMFullAccess
  * AmazonKinesisVideoStreamsFullAccess
* Select *Next: Tags*
* Enter Key: *project*, Value: *kvs-camera*
* Select *Next: Review*
* Select *Create user*
* Select *Download .csv*. The .csv file contains the user access key ID and secret access key. You can also copy and paste them to your notepad now. You will need these files to setup the AWS CLI.

## AWS CLI

### Installing the AWS CLI v2

Install AWS CLI v2 with:

```bash
git clone https://github.com/aws/aws-cli.git
cd aws-cli && git checkout v2
pip3 install https://github.com/boto/botocore/zipball/v2#egg=botocore --upgrade
pip3 install -r requirements.txt
pip3 install .
```

Verify that the AWS CLI installed correctly.

```bash
aws --version
```

Response should be something like

```bash
aws-cli/2.2.8 Python/3.7.3 Linux/5.10.17-v7+ source/armv7l.raspbian.10 prompt/off
```

### Configuring the AWS CLI

```bash
aws configure --profile iot-profile
```
Enter the IAM access keys, secret keys, region, and output type. For help configuring see [Configuration basics](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)

The IAM user must have these IAM permissions:
* [AWS Managed Policy] AWSIoTFullAccess
* [AWS Managed Policy] IAMFullAccess
* [AWS Managed Policy] AmazonKinesisVideoStreamsFullAccess

* Enter AWS Access Key ID
* Enter AWS Secret Access Key
* Enter Default region name, e.g. *ap-southeast-2*
* Default output format *json*

## AWS IoT

Start in your home directory

```bash
cd ~
mkdir certs
mkdir aws-iot
```

### Step 1: Create an IoT Thing Type and an IoT Thing

Register your device in the AWS IoT thing registry database by creating a thing type and a thing.

```bash
aws --profile iot-profile iot create-thing-type --thing-type-name kvs-camera > ./aws-iot/iot-thing-type.json
```

Check your thing type was created

```bash
aws --profile iot-profile iot list-thing-types
```

Run the following command in the AWS CLI to create a thing.

```bash
aws --profile iot-profile iot create-thing --thing-name kvs-camera-01 --thing-type-name kvs-camera --attribute-payload "{\"attributes\": {\"device-type\":\"raspberry-pi\",\"project\":\"kvs\"}}" > ./aws-iot/iot-thing.json
```

Check your thing was created

```bash
aws --profile iot-profile iot list-things
```

### Step 2: Create an IAM Role to be Assumed by IoT

#### Create a trust policy

Create a trust policy file that grants the credentials provider permission to assume the role.

```json
cat > ./aws-iot/iam-policy-document.json <<EOF
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
aws --profile iot-profile iam create-role --role-name kvs-camera --assume-role-policy-document file://aws-iot/iam-policy-document.json > ./aws-iot/iam-role.json
```

#### Create a permissions policy

Create an access policy file that grants Kinesis Video Streams operations. This policy authorizes the specified actions only on a video stream (AWS resource) that is specified by the placeholder (${credentials-iot:ThingName}). This placeholder takes on the value of the IoT thing attribute ThingName when the IoT credentials provider sends the video stream name in the request. Copy the json policy below.

```json
cat > ./aws-iot/iam-permisson-policy.json <<EOF
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
aws --profile iot-profile iam put-role-policy --role-name kvs-camera --policy-name kvs-iam-policy --policy-document file://aws-iot/iam-permisson-policy.json 
```

#### Create a role alias

Create a Role Alias for your IAM Role. An IoT credendials provider request must include a role-alias to indicate which IAM role to assume in order to obtain temporary credentials.

```bash
aws --profile iot-profile iot create-role-alias --role-alias kvs-role-alias --role-arn $(jq --raw-output '.Role.Arn' ./aws-iot/iam-role.json) > ./aws-iot/iot-role-alias.json
```

Create a policy document that will enable AWS IoT to assume role with the X.509 certificate (once it is attached) using the role alias.

```bash
cat > ./aws-iot/iot-policy-document.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
	 "Effect":"Allow",
	 "Action":[
	    "iot:Connect"
	 ],
	 "Resource":"$(jq --raw-output '.roleAliasArn' ./aws-iot/iot-role-alias.json)"
 },
      {
	 "Effect":"Allow",
	 "Action":[
	    "iot:AssumeRoleWithCertificate"
	 ],
	 "Resource":"$(jq --raw-output '.roleAliasArn' ./aws-iot/iot-role-alias.json)"
 }
   ]
}
EOF
```

Create the policy in AWS IoT.

```bash
aws --profile iot-profile iot create-policy --policy-name kvs-iot-policy --policy-document file://aws-iot/iot-policy-document.json
```

## Step 3: Create and Register the X.509 certificate

The following create-keys-and-certificate creates a 2048-bit RSA key pair and issues an X.509 certificate using the issued public key. Because this is the only time that AWS IoT provides the private key for this certificate, be sure to keep it in a secure location.

```bash
aws --profile iot-profile iot create-keys-and-certificate --set-as-active --certificate-pem-outfile ./certs/certificate.pem --public-key-outfile ./certs/public.pem.key --private-key-outfile ./certs/private.pem.key > ./aws-iot/certificate.json    
```

Attach the policy for IoT to this certificate.

```bash
aws --profile iot-profile iot attach-policy --policy-name kvs-iot-policy --target $(jq --raw-output '.certificateArn' ./aws-iot/certificate.json)        
```

Attach your IoT thing to the certificate you just created.

```bash
aws --profile iot-profile iot attach-thing-principal --thing-name kvs-camera-01 --principal $(jq --raw-output '.certificateArn' ./aws-iot/certificate.json)
```

Get the IoT credentials endpoint unique to your AWS account ID.

```bash
aws --profile iot-profile iot describe-endpoint --endpoint-type iot:CredentialProvider > ./aws-iot/iot-credential-provider.json
```

Get the CA certificate to establish trust with the back-end service through TLS.

```bash
curl --silent 'https://www.amazontrust.com/repository/SFSRootCAG2.pem' --output ./certs/cacert.pem
```

## Step 4: Test the IoT Credentials with Your Kinesis Video Stream 

Create a Kinesis video stream that you want to test this configuration with. The video stream name must be identical to the IoT thing name that you created in the previous step.

```bash
aws --profile iot-profile kinesisvideo create-stream --data-retention-in-hours 24 --stream-name kvs-camera-01
```

Invoke the Kinesis Video Streams DescribeStream API for the kinesis video stream

```bash
aws --profile iot-profile kinesisvideo describe-stream --stream-name kvs-camera-01
```

## Step 5: Request a security token and set AWS environmental keys

Call the IoT credentials provider to get the temporary credentials:

```bash
curl --silent -H "x-amzn-iot-thingname:kvs-camera-01" --cert ./certs/certificate.pem --key ./certs/private.pem.key https://$(jq --raw-output '.endpointAddress' ./aws-iot/iot-credential-provider.json)/role-aliases/kvs-role-alias/credentials --cacert ./certs/cacert.pem > ./aws-iot/token.json
```

Set the credentials to session variables

```bash
export AWS_ACCESS_KEY_ID=$(jq --raw-output '.credentials.accessKeyId' ./aws-iot/token.json)
export AWS_SECRET_ACCESS_KEY=$(jq --raw-output '.credentials.secretAccessKey' ./aws-iot/token.json)
export AWS_SESSION_TOKEN=$(jq --raw-output '.credentials.sessionToken' ./aws-iot/token.json)
export STREAM_NAME="kvs-camera-01"
export AWS_REGION="ap-southeast-2"
```

Test with

```bash
echo $AWS_ACCESS_KEY_ID
echo $AWS_SECRET_ACCESS_KEY
echo $AWS_SESSION_TOKEN
echo $STREAM_NAME
echo $AWS_REGION
```

## Step 6: Build the AWS Kinesis Video Streams SDK

### Install KVS Producer

```bash
git clone https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp.git
mkdir -p amazon-kinesis-video-streams-producer-sdk-cpp/build
cd amazon-kinesis-video-streams-producer-sdk-cpp/build
cmake .. -DBUILD_GSTREAMER_PLUGIN=ON -DBUILD_JNI=TRUE
make
```

### Install GStreamer


```bash
sudo apt-get install libssl-dev libcurl4-openssl-dev liblog4cplus-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev gstreamer1.0-plugins-base-apps gstreamer1.0-plugins-bad gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly gstreamer1.0-tools
```

Test the GStreamer installation with:

```bash
gst-inspect-1.0 fakesrc
gst-launch-1.0 -v fakesrc silent=false num-buffers=3 ! fakesink silent=false
gst-launch-1.0 videotestsrc ! videoconvert ! autovideosink
```

Use Control-C to exit the test.

### Environmental Variables

These environmental variables are needed to start GStreamer.

```bash
export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
```

Test that it is working with:

```bash
gst-inspect-1.0 kvssink
```

If the build failed, or GST_PLUGIN_PATH is not properly set you will get output like

```text
No such element or plugin 'kvssink'
```

### Check for cameras in your system

Run the gst-device-monitor-1.0 command to identify available media devices in your system.

```bash
gst-device-monitor-1.0
```

Look for the device.path of the camera that you want to use. e.g /dev/video0

## Step 7: Run GStreamer

You have two options for authentication, using AWS access key and secret access key, or using AWS certificates.

* When using IoT authorization, the value of stream-name must be identical to the value of iot-thingname (in IoT provisioning).
* storage-size: The storage size of the device in megabytes.
* access-key: The AWS access key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
* secret-key: The AWS secret key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
* credential-path: A path to a file containing your credentials for accessing Kinesis Video Streams. You must provide either this parameter or access-key and secret-key.

### GStreamer at 640x420 with AWS access keys

This commands starts GStreamer at 640x420 resolution using the AWS access key and secret key.

```
gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=640,height=480 \
! omxh264enc control-rate=2 target-bitrate=512000 periodicity-idr=45 inline-header=FALSE \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,profile=baseline \
! kvssink \
    stream-name="$STREAM_NAME" \
    aws-region="$AWS_REGION"
    storage-size=128 \
    access-key="$AWS_ACCESS_KEY_ID" \
    secret-key="$AWS_SECRET_ACCESS_KEY" \
```

### GStreamer at 640x420 with AWS IoT certificates

This commands starts GStreamer at 640x420 resolution using the AWS IoT certificates.

```bash
gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=640,height=480 \
! omxh264enc control-rate=2 target-bitrate=512000 inline-header=FALSE periodicity-idr=45 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,profile=baseline \
! kvssink \
    stream-name="$STREAM_NAME" \
    aws-region="$AWS_REGION" \
    storage-size=128 \
    iot-certificate="iot-certificate,endpoint=iot-credential-endpoint-host-name,cert-path=/home/pi/certs/certificate.pem,key-path=/home/pi/certs/private.pem.key,ca-path=/home/pi/certs/cacert.pem,role-aliases=kvs-role-alias"
```

### GStreamer at 1280x720 with AWS IoT certificates

This commands starts GStreamer at 1280x720 resolution using the AWS IoT certificates.

```bash
gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=1280,height=720,framerate=10/1 \
! omxh264enc control-rate=1 target-bitrate=8024000 inline-header=FALSE periodicty-idr=4 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,width=1280,height=720,framerate=10/1,profile=baseline \
! kvssink \
    stream-name="$STREAM_NAME" \
    aws-region="$AWS_REGION" \
    storage-size=128 \
    iot-certificate="iot-certificate,endpoint=iot-credential-endpoint-host-name,cert-path=/home/pi/certs/certificate.pem,key-path=/home/pi/certs/private.pem.key,ca-path=/home/pi/certs/cacert.pem,role-aliases=kvs-role-alias"
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

You can get the data endpoint for your video stream:

```bash
aws --profile iot-profile kinesisvideo get-data-endpoint --stream-name kvs-camera-01 --api-name GET_CLIP > ./aws-iot/kvs-clip.json
```

Download a clip with the AWS CLI

```bash
aws --profile iot-profile kinesis-video-archived-media get-clip \
  --stream-name kvs-camera-01 \
  --clip-fragment-selector "FragmentSelectorType=SERVER_TIMESTAMP,TimestampRange={StartTimestamp=$(date -d "1 minute ago" -u "+%FT%T+0000"),EndTimestamp=$(date -u "+%FT%T+0000")}" \
  kvs-camera-01.mp4
```

### HLS Streaming

The data end point for your video stream:

```bash
aws --profile iot-profile kinesisvideo get-data-endpoint --stream-name kvs-camera-01 --api-name GET_HLS_STREAMING_SESSION_URL > ./aws-iot/hls-session-url.json
```

The HLS streaming session URL for your video stream from the data end-point:

```bash
aws --profile iot-profile kinesis-video-archived-media get-hls-streaming-session-url --endpoint-url $(jq --raw-output '.DataEndpoint' ./aws-iot/hls-session-url.json) \
--stream-name kvs-camera-01 \
--playback-mode LIVE
```

The output from this request is live HLS video stream that you can open in a web browser. Test in [hls.js](https://hls-js.netlify.app/demo/)

## Run GStreamer on Pi Startup

### Setup Environmental variables

Set the required environmental variables on Pi startup:

```json
sudo cat > /etc/profile.d/aws-kvs.sh <<EOF
export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
export STREAM_NAME="kvs-camera-01"
export AWS_REGION="ap-southeast-2"
EOF
```

Reboot the pi and test with:

```bash
echo $GST_PLUGIN_PATH
echo $LD_LIBRARY_PATH
echo $STREAM_NAME
echo $AWS_REGION
```

### Configure KVS logging

Copy the text below to configure the KVS log configuration.

```bash
cat > /home/pi/kvs_log_configuration <<EOF
log4cplus.rootLogger=ERROR, KvsConsoleAppender
log4cplus.logger.MemoryCheck=ERROR, KvsFileAppender
log4cplus.logger.MemoryCheck=ERROR, KvsConsoleAppender

#KvsConsoleAppender:
log4cplus.appender.KvsConsoleAppender=log4cplus::ConsoleAppender
log4cplus.appender.KvsConsoleAppender.layout=log4cplus::PatternLayout
log4cplus.appender.KvsConsoleAppender.layout.ConversionPattern=[%-5p][%d] %m%n

#KvsFileAppender
log4cplus.appender.KvsFileAppender=log4cplus::DailyRollingFileAppender
log4cplus.appender.KvsFileAppender.File=/home/pi/logs/kvs-camera-01.log
log4cplus.appender.KvsFileAppender.Schedule=HOURLY
log4cplus.appender.KvsFileAppender.CreateDirs=true
log4cplus.appender.KvsFileAppender.layout=log4cplus::PatternLayout
log4cplus.appender.KvsFileAppender.layout.ConversionPattern=[%D]-%p-%m%n
EOF
```

### Raspberry Pi startup script

#### Configure bash startup script

Create the bash startup script below.

```bash
mkdir startup
nano /home/pi/startup/aws-kvs.sh
```

Copy the text below

```bash
#!/bin/bash
echo "----------------------------------------"
echo "System date and time: $(date '+%d/%m/%Y %H:%M:%S')"
echo "Kernel info: $(uname -rmv)"

export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
echo $GST_PLUGIN_PATH
echo $LD_LIBRARY_PATH

gst-inspect-1.0 kvssink

export STREAM_NAME="kvs-camera-01"
export AWS_REGION="ap-southeast-2"
echo $STREAM_NAME
echo $AWS_REGION

gst-launch-1.0 -q v4l2src device=/dev/video0 \
! videoconvert \
! video/x-raw,format=I420,width=1280,height=720,framerate=10/1 \
! omxh264enc control-rate=1 target-bitrate=8024000 inline-header=FALSE periodicty-idr=4 \
! h264parse \
! video/x-h264,stream-format=avc,alignment=au,width=1280,height=720,framerate=10/1,profile=baseline \
! kvssink \
    stream-name="$STREAM_NAME" \
    aws-region="$AWS_REGION" \
    storage-size=128 \
    iot-certificate="iot-certificate,endpoint=$(jq --raw-output '.endpointAddress' /home/pi/aws-iot/iot-credential-provider.json),cert-path=/home/pi/certs/certificate.pem,key-path=/home/pi/certs/private.pem.key,ca-path=/home/pi/certs/cacert.pem,role-aliases=kvs-role-alias" &
```

Make the script executable with:

```bash
sudo chmod a+x /home/pi/startup/aws-kvs.sh
```

Test the script with:

```bash
sh /home/pi/startup/aws-kvs.sh
```

Check that the KVS stream is appearing in the AWS management console.

Kill the script with

```bash
ps - ef
sudo kill [--insert pid # of th
```

#### Configure python startup script (in-progress)

Create the python startup script below.

```bash
nano /home/pi/startup/aws-kvs.py
```

```python
import logging
logging.basicConfig(filename='aws-kvs.py.log', level=logging.DEBUG, format='%(asctime)s %(message)s') 
logging.debug('Debug - startup script')
logging.info('Info - startup script')
logging.warning('Warning - startup script')
logging.error('Error - startup script')
```

Make the script executable with:

```bash
sudo chmod a+x /home/pi/startup/aws-kvs.py
```

Test the script with:

```bash
python /home/pi/startup/aws-kvs.py
```


#### Run the script at startup as the root user

If the script is working OK, then edit your crontab list to load the script on the Pi bootup.

```bash
crontab -e
```
Select nano. Add these lines to the end of the file.

```bash
@reboot python3 /home/pi/startup/aws-kvs.py
@reboot /home/pi/startup/aws-kvs.sh
```

Reboot the Raspberry PI to test that the script during system startup:

```bash
sudo reboot now
```

Check the script output logs:

```bash
cat /home/pi/logs/kvs-camera-01.log
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
* [Troubleshooting Kinesis Video Streams](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/troubleshooting.html)

