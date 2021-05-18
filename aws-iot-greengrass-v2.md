# AWS IoT Greengrass V2

## Prerequisites

* Setup SSH access to your Raspberry Pi
* AWS CLI installed and configured with temporary security credentials

## Environment setup

SSH to your Raspberry Pi and run these environment setup commands.

Check that Java 11 is installed.

```bash
java -version
```

To install the Java runtime.

```bash
sudo apt install default-jdk
```

## Install and configure the AWS IoT Greengrass Core software

Download Greengrass

```bash
cd ~
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip > greengrass-nucleus-latest.zip
unzip greengrass-nucleus-latest.zip -d RaspberryPiGreengrassCore && rm greengrass-nucleus-latest.zip
```

Install Greengrass

```bash
sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
  -jar ./RaspberryPiGreengrassCore/lib/Greengrass.jar \
  --aws-region ap-southeast-2 \
  --thing-name RaspberryPiGreengrassCore \
  --thing-group-name RaspberryPiGreengrassCoreGroup \
  --tes-role-name RaspberryPiGreengrassV2TokenExchangeRole \
  --tes-role-alias-name RaspberryPiGreengrassCoreTokenExchangeRoleAlias \
  --component-default-user ggc_user:ggc_group \
  --provision true \
  --setup-system-service true \
  --deploy-dev-tools true
```

You should see the following messages to indicate that the installer succeeded. 

```bash
Provisioning AWS IoT resources for the device with IoT Thing Name: [RaspberryPiGreengrassCoreGreengrassCore]...
Found IoT policy "GreengrassV2IoTThingPolicy", reusing it
Creating keys and certificate...
Attaching policy to certificate...
Creating IoT Thing "RaspberryPiGreengrassCore"...
Attaching certificate to IoT thing...
Successfully provisioned AWS IoT resources for the device with IoT Thing Name: [RaspberryPiGreengrassCoreGreengrassCore]!
Adding IoT Thing [RaspberryPiGreengrassCoreGreengrassCore] into Thing Group: [RaspberryPiGreengrassCoreGroup]...
Successfully added Thing into Thing Group: [RaspberryPiGreengrassCoreGroup]
Setting up resources for aws.greengrass.TokenExchangeService ... 
TES role alias "RaspberryPiGreengrassCoreTokenExchangeRoleAlias" does not exist, creating new alias...
TES role "RaspberryPiGreengrassV2TokenExchangeRole" does not exist, creating role...
IoT role policy "GreengrassTESCertificatePolicyRaspberryPiGreengrassCoreTokenExchangeRoleAlias" for TES Role alias not exist, creating policy...
Attaching TES role policy to IoT thing...
No managed IAM policy found, looking for user defined policy...
No IAM policy found, will attempt creating one...
IAM role policy for TES "RaspberryPiGreengrassV2TokenExchangeRoleAccess" created. This policy DOES NOT have S3 access, please modify it with your private components' artifact buckets/objects as needed when you create and deploy private components 
Attaching IAM role policy for TES to IAM role for TES...
Configuring Nucleus with provisioned resource details...
Root CA found at "/greengrass/v2/rootCA.pem". Skipping download.
Created device configuration
Successfully configured Nucleus with provisioned resource details!
Creating a deployment for Greengrass first party components to the thing group
Configured Nucleus to deploy aws.greengrass.Cli component
Successfully set up Nucleus as a system service
```

Check the status of the deployment.

```bash
aws greengrassv2 list-effective-deployments --core-device-thing-name RaspberryPiGreengrassCore
```

Successful output should be:

```bash
{
    "effectiveDeployments": [
        {
            "deploymentId": "ab044a09-3793-4911-8256-d5c092c59be7",
            "deploymentName": "Deployment for RaspberryPiGreengrassCoreGroup",
            "iotJobId": "52d0a002-b6f9-4bd8-9342-eb1f08726afe",
            "iotJobArn": "arn:aws:iot:ap-southeast-2:418153391183:job/52d0a002-b6f9-4bd8-9342-eb1f08726afe",
            "targetArn": "arn:aws:iot:ap-southeast-2:418153391183:thinggroup/RaspberryPiGreengrassCoreGroup",
            "coreDeviceExecutionStatus": "SUCCEEDED",
            "reason": "SUCCESSFUL",
            "creationTimestamp": 1621337757.356,
            "modifiedTimestamp": 1621337757.356
        }
    ]
}
```

Verify the Greengrass deployment with:

```bash
/greengrass/v2/bin/greengrass-cli help
```

## Make a test HelloWorld Greengrass component

```bash
cd RaspberryPiGreengrassCore/
mkdir recipes
cd recipes
```

Create recipe JSON file:

```json
cat > com.example.HelloWorld-1.0.0.json <<EOF
{
  "RecipeFormatVersion": "2020-01-25",
  "ComponentName": "com.example.HelloWorld",
  "ComponentVersion": "1.0.0",
  "ComponentDescription": "My first AWS IoT Greengrass component.",
  "ComponentPublisher": "Amazon",
  "ComponentConfiguration": {
    "DefaultConfiguration": {
      "Message": "world"
    }
  },
  "Manifests": [
    {
      "Platform": {
        "os": "linux"
      },
      "Lifecycle": {
        "Run": "python3 -u {artifacts:path}/hello_world.py '{configuration:/Message}'"
      }
    }
  ]
}
EOF
```

Create a folder for artifacts

```bash
mkdir -p artifacts/com.example.HelloWorld/1.0.0
```

Create Python script

```json
cat > artifacts/com.example.HelloWorld/1.0.0/hello_world.py <<EOF
import sys
import datetime

message = f"Hello, {sys.argv[1]}! Current time: {str(datetime.datetime.now())}."
message += " Greetings from your first Greengrass component."

# Print the message to stdout.
print(message)

# Append the message to the log file.
with open('/tmp/Greengrass_HelloWorld.log', 'a') as f:
    print(message, file=f)
EOF
```

Deploy the component to the AWS IoT Greengrass core.

```bash
sudo /greengrass/v2/bin/greengrass-cli deployment create \
  --recipeDir ~/RaspberryPiGreengrassCore/recipes \
  --artifactDir ~/RaspberryPiGreengrassCore/artifacts \
  --merge "com.example.HelloWorld=1.0.0"
```

Verify that the component runs and logs messages.

```bash
tail -f /tmp/Greengrass_HelloWorld.log
```







