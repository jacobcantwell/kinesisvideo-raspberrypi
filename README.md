# AWS Kinesis Video Streams on a Raspberry Pi

## Overview

Setting up AWS Kinesis Video Streams on a Raspberry Pi





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
export STREAM_NAME="raspberry-pi-kvs-stream-v30"
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

### Configure Raspberry Pi startup script

```bash
sudo nano /usr/local/bin/startup.sh
```

Copy the startup script below.

```text
#!/bin/bash
# Store first parameter in a variable, which should be the log file location.
LOG_FILE="$1"

# Set a default log file location if the parameter was empty, i.e. not specified.
if [ -z "$LOG_FILE" ]
then
  LOG_FILE="/var/log/testlog.txt"
fi

# Append information to the log file.
echo "----------------------------------------" >> "$LOG_FILE"
echo "System date and time: $(date '+%d/%m/%Y %H:%M:%S')" >> "$LOG_FILE"
echo "Kernel info: $(uname -rmv)" >> "$LOG_FILE"

export GST_PLUGIN_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/build
export LD_LIBRARY_PATH=/home/pi/amazon-kinesis-video-streams-producer-sdk-cpp/open-source/local/lib
export STREAM_NAME="raspberry-pi-kvs-stream-v30"
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
sudo chmod +x /usr/local/bin/startup.sh
```

Test the script with:

```bash
sudo sh /usr/local/bin/startup.sh
```

Check that the KVS stream is appearing in the AWS management console.

Kill the script with

```bash
ps - ef
sudo kill [--insert pid # of th
```

If the script is working OK, then 










