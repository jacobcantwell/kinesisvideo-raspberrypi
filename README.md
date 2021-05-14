# AWS Kinesis Video Streams on a Raspberry Pi

## Overview

This script sets up AWS Kinesis Video Streams on a Raspberry Pi and sets up a startup script to load the KVS stream on startup.

### Camera Options

This script works with the official raspberry pi camera or an external USB camera plugged into the Raspberry Pi.



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


