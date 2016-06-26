---
layout: post
published: true
author: Jack Williams
mathjax: false
featured: true
comments: true
title: AWS AutoScaling with Octopus Deploy
category: AWS
tags:
  - AWS
  - Octopus
  - Autoscale
subtitle: Subtitle
modified: ""
---
## Fear Not, It Can Be Done

Let me first give accolades to Jeffery Palermo. He did a bootcamp on Continuous Delivery at the Kansas City Developer Conferencce that I attended - it really made me re-think our CD process. It was his kindness and helpfulness to other developers that reminded me I need to give back as well. I've been a long time reader of his blogs and books, and overall fan. It was really cool to meet him in person and see how much of a helping spirit he has.

I've been meaning to blog about this but just haven't found the time. Now is as good a time as any!

### Background

On the project I'm currently working on I had to figure out how to setup autoscaling with Octopus Deploy - and it was no simple feat. Especially since this is Gov cloud. AWS Gov Cloud does not have the full feature set of AWS (such as various helpful DevOps features). There wasn't a ton of information on this topic, so this post details the process I came up with to handle this. If you know of a better way, please let me know!

### Process Outline

1. Create scripts to
	- Delete old, stale tentacles
    - Create a new tentacle
    - Assign the tentacle the appropriate roles
    - Assign the tentacle the appropriate environments
    - Push the latest Octopus Deployment to the new tentacle
2. Create an AWS Launch Configuration which
	- Spins up a new instance
    - Pulls in all needed scripts (from above) and executables
    - Execute the scripts
    
### What this post is NOT about.

- Setting up Autoscaling in AWS
- Setting up a Cloud Formation template

### Requirements

- An understanding of Autoscaling groups and LaunchConfigurations in AWS
- An understanding of CloudFormation in AWS
- Patience
- More Patience

I mention patience twice because testing your scripts takes killing one instance and spinning up a new one a few times. This isn't always the case, but once you get them running it takes a few scaling operations to make sure everything is on the up and up.

### Tips
1. To avoid the stop and restart instance cycle
	- Get everything setup like you want, then do an initial scaling operation.
    - If it doesn't work, RDP into your instance and GET them working manually.
    - If it does work, REJOICE.
2. Log files are your friend, you can find them on the new instance in C:\cfn\log.
3. Turn OFF auto-rollback when your cloud fails to start (otherwise you can't look at the logs).

#### Why Multple Environments and Multiple Roles

More explanation as to why my instances belong to multiple environents and have multiple roles. This project doesn't have a ton of users yet, so spinning up a seperate instance for staging, demo and production, along with seperate database instances for each was too costly. **But the infrastructure is in place to do it**.

The current configuration is something like this: Two "environments" can get spun up with a CloudFormation script. There is a Cloud Formation parameter that says whether or not the environment is "aux" (which means it hosts demo, staging and training), or "prod" which means it's the production environment. 

This works great for now and can be changed fairly easy when it's time to seriously scale. The main thing is that it's all done in a CloudFormation template, so adding servers is easy. In addition to that, all of the services are modular (there are a few: the Web Server, Image Server, and Hangfire Server). It doesn't matter if they're deployed to IIS on one instance, or three instances - it all works the same.

Sorry for the long intro, lets get down to business.

### The Scripts

A special thank you to Dalmiro Grañas (he's on the support staff at Octopus Deploy, and is awesome!). The RegisterTentacle.ps1 was from him. I can't remember exactly where it is on the interwebs, but I must give credit where credit is due. I've modified it to suit my needs!

##### RegisterTentacle.ps1

Creates a new tentacle with our octopus server and pushes the latest release to it. Also clears out old tentacles that are dead. I've commented the file for guidance.

```powershell
param (
    # Can be a list
    [Parameter(Mandatory=$True)]
    [string[]] $env, 
	
    # PROD or AUX
    [Parameter(Mandatory=$True)]
    [string] $tentaclePrefix, 
	
    [Parameter(Mandatory=$True)]
    [string] $tentaclePort = "10933",
	
    # Can be a list
    [string[]] $roles = @("Webserver"),
	
    # Octopus API key
    [string] $apiKey = ""
	
)

# Config

# Octopus API Key
$OctopusAPIkey = $apiKey 

# Octopus URL
$OctopusURL = "https://my.octopusserver.com"

# Tentacle IP. Remember it must be https. We can get this from AWS Metadata.
$TentacleIP = (Invoke-WebRequest http://169.254.169.254/latest/meta-data/public-ipv4).Content

# Results in a name like PROD-123.22.33.112 or AUX-123.22.33.112
$TentacleName = "$tentaclePrefix-$TentacleIP"

# Port the Listening Tentacle will use to comunicate with the Octopus server
$TentaclePort = $tentaclePort

# Tentacle URL
$TentacleURL = "https://${TentacleIP}:${TentaclePort}/"

# Tentacle Thumbprint
$TentacleThumbprint = ""

# Tentacle roles (can be a list)
$TentacleRoles = $roles

# Tentacle environments (can be a list) 
$EnvironmentIDs = $env 

# Tentacle exe location on default install
$tentacleExe = 'C:\Program Files\Octopus Deploy\Tentacle\Tentacle.exe'

# This gets the tentacles thumbprint through black magic
[xml]$doc = Get-Content C:\Octopus\Tentacle.config
$nodes = Select-Xml "//set[@key='Tentacle.CertificateThumbprint']" $doc
$TentacleThumbprint = $nodes.Node.'#text'

# Octo deploy-release. This is used to push the latest release to our new tentacle. 

# Path to octo.exe
$octoExe = "c:\setup\octopus\octo\octo.exe"

# What we are doing (deploying a release)
$octoArg1 = "deploy-release"

# Which release? (this gets changed later to an explicit version)
$octoArg2 = "--version=latest"

# Which environment
$octoArg3 = "--deployto"

# Reserved for the environment name
$octoArg4 = ""

# Which machine to deploy to (PROD-XX.XX.XX.XX or AUX-XX.XX.XX.XX)
$octoArg5 = "--specificmachines=${TentacleName}"

# Project and other variables
$octoArg6 = "--project=MY_PROJECT"
$octoArg7 = "--apiKey=$OctopusAPIkey"
$octoArg8 = "--server=$OctopusURL"

# Import and setup Octoposh
Import-Module Octoposh
Set-OctopusConnectionInfo -URL $OctopusURL -APIKey $OctopusAPIkey

# Here we are sort of instantiating a new instance of a machine
$machine = Get-OctopusResourceModel -Resource Machine

# Set the environment Ids of the machine
for($cnt=0; $cnt -lt $env.length; $cnt++){
	$environment = Get-OctopusEnvironment -EnvironmentName $env[$cnt]
	$machine.EnvironmentIds.Add($environment.id)
}

# Set the roles of the machine
for($roleCount=0; $roleCount -lt $roles.length; $roleCount++){
	$machine.Roles.Add($roles[$roleCount])
}

# Set the name of the machine
$machine.name = $TentacleName

# Set the endpoing and thumbprint of the machine
$machineEndpoint = New-Object Octopus.Client.Model.Endpoints.ListeningTentacleEndpointResource
$machine.EndPoint = $machineEndPoint
$machine.EndPoint.Uri = $TentacleURL
$machine.EndPoint.Thumbprint = $TentacleThumbprint

# Clean environment of orphaned machines
# Note: This may not be the best time or place to do this, but I have not found a better way yet.
for ($i=0; $i -lt $env.length;$i++){
	$octoArg4 = $env[$i]
    & $octoExe "clean-environment" "--environment=$octoArg4" "--status=Offline" $octoArg7 $octoArg8
}

# Finally, create the new machine on the Octopus Server
New-OctopusResource -Resource $machine

# Deploy to each environment passed in
for ($i=0; $i -lt $env.length;$i++){
    # Environment name
	$octoArg4 = $env[$i]
    
    # To get the latest version in each env, we have to pull the env
    $octoEnv = Get-OctopusEnvironment $octoArg4

    # Get its latest released version
    $latestVersion = $octoEnv.LatestDeployment.ReleaseVersion

    # And use that version to deploy to our newest tentacle
    $octoArg2 = "--version=$latestVersion"

    # Deploy!
	& $octoExe $octoArg1 $octoArg2 $octoArg3 $octoArg4 $octoArg5 $octoArg6 $octoArg7 $octoArg8
}
```

Notes


- The only time old tentacles are cleared is on a scaling operation, there is probably a better way to handle this - please let me know! 

##### PrepareEC2Instance.ps1

This script just sets up Octoposh for use in the RegisterTentacle.ps1 script

```powershell

# Setup Octoposh for pulling latest octopus deployment
# By copying it to the WindowsPowerShell modules folder

$destDir = "C:\Program Files\WindowsPowerShell\Modules\Octoposh\"

# Create directory if it doesn't exist
if (!(Test-Path -path $destDir)) { New-Item $destDir -itemtype directory -force }

# Copy octoposh to powershell modules folder
copy-item C:\setup\OctoPosh-0.3.5\* $destDir -force -Recurse -Verbose

# IIS Prep
add-windowsfeature web-webserver -includeallsubfeature

add-windowsfeature web-mgmt-tools -includeallsubfeature
```

##### Cloud Formation Setup

The first part of the Cloud Formation setup consists of importing all needed files and executables. I have a few different configs setup in the LaunchConfig's AWS::CloudFormation::Init element: aux, prod, default. Default gets run in all AWS environments, prod gets run in prod, and aux gets run for [staging, demo, training].

Here is the default config section, which includes the commands, packages, files and sources to download / run.

##### default config

```json
"default":{  
      "commands":{  
         "1-prepare-ec2-instance":{  
            "command":"@powershell -NoProfile -ExecutionPolicy Bypass -Command \"C:\\setup\\PrepareEC2Instance.ps1\"",
            "waitAfterCompletion":"0"
         }
      },
      "packages":{  
         "msi":{  
            "octopus":"https://octopus.com/downloads/latest/OctopusTentacle64"
         }
      },
      "files":{  
         "C:\\setup\\octopus\\tentacle\\RegisterTentacle.ps1":{  
            "source":{  
               "Fn::Join":[  
                  "/",
                  [  
                     "http://s3-us-west-1.amazonaws.com",
                     {  
                        "Ref":"BucketName"
                     },
                     "RegisterTentacle.ps1"
                  ]
               ]
            }
         },
         "C:\\setup\\PrepareEC2Instance.ps1":{  
            "source":{  
               "Fn::Join":[  
                  "/",
                  [  
                     "http://s3-us-west-1.amazonaws.com",
                     {  
                        "Ref":"BucketName"
                     },
                     "PrepareEC2Instance.ps1"
                  ]
               ]
            }
         },
         "c:\\cfn\\cfn-hup.conf":{  
            "content":{  
               "Fn::Join":[  
                  "",
                  [  
                     "[main]\n",
                     "stack=",
                     {  
                        "Ref":"AWS::StackName"
                     },
                     "\n",
                     "region=",
                     {  
                        "Ref":"AWS::Region"
                     },
                     "\n"
                  ]
               ]
            }
         },
         "c:\\cfn\\hooks.d\\cfn-auto-reloader.conf":{  
            "content":{  
               "Fn::Join":[  
                  "",
                  [  
                     "[cfn-auto-reloader-hook]\n",
                     "triggers=post.update\n",
                     "path=Resources.WebServerInstance.Metadata.AWS::CloudFormation::Init\n",
                     "action=cfn-init.exe -v -s ",
                     {  
                        "Ref":"AWS::StackName"
                     },
                     " -r LaunchConfig",
                     " --region ",
                     {  
                        "Ref":"AWS::Region"
                     },
                     "\n"
                  ]
               ]
            }
         }
      },
      "sources":{  
         "C:\\setup\\octopus\\octo":"https://download.octopusdeploy.com/octopus-tools/3.3.6/OctopusTools.3.3.6.zip",
         "C:\\setup":"https://github.com/Dalmirog/OctoPosh/archive/0.3.5.zip"
      }
   }

```

Notes

- Under commands, it simply runs the prepare-ec2-instance.ps1 script
- Under packages, I link to the tentacle installation package, it installs for us
- Under files, I put my RegisterTentacle and PrepareEC2Instance PS1 files in AWS S3, an have a Cloud Formation parameter to tell which bucket I put them in
- Under sources, we download the OctopusTools.3.3.6.zip file which gives us access to octo.exe
- Under sources, we download OctoPosh, a nice module made by Dalmirog from Octopus Support

##### aux config

```json
"aux":{  
      "commands":{  
         "1-create-instance":{  
            "command":"\"C:\\Program Files\\Octopus Deploy\\Tentacle\\Tentacle.exe\" create-instance --instance \"Tentacle\" --config \"C:\\Octopus\\Tentacle.config\" --console",
            "waitAfterCompletion":"0"
         },
         "2-new-cert":{  
            "command":"\"C:\\Program Files\\Octopus Deploy\\Tentacle\\Tentacle.exe\" new-certificate --instance \"Tentacle\" --if-blank --console ",
            "waitAfterCompletion":"0"
         },
         "3-config-reset-trust":{  
            "command":"\"C:\\Program Files\\Octopus Deploy\\Tentacle\\Tentacle.exe\" configure --instance \"Tentacle\" --reset-trust --console",
            "waitAfterCompletion":"0"
         },
         "4-config-set-directories":{  
            "command":"\"C:\\Program Files\\Octopus Deploy\\Tentacle\\Tentacle.exe\" configure --instance \"Tentacle\" --home \"C:\\Octopus\" --app \"C:\\Octopus\\Applications\" --port \"10933\" --noListen \"False\" --console ",
            "waitAfterCompletion":"0"
         },
         "5-config-set-trust":{  
            "command":"\"C:\\Program Files\\Octopus Deploy\\Tentacle\\Tentacle.exe\" configure --instance \"Tentacle\" --trust \"OCTOPUS_SERVER_THUMBPRINT\" --console ",
            "waitAfterCompletion":"0"
         },
         "6-add-firewall-rule":{  
            "command":"netsh advfirewall firewall add rule \"name=Octopus Deploy Tentacle\" dir=in action=allow protocol=TCP localport=10933",
            "waitAfterCompletion":"0"
         },
         "7-start-tentacle":{  
            "command":"\"C:\\Program Files\\Octopus Deploy\\Tentacle\\Tentacle.exe\" service --instance \"Tentacle\" --install --start --console ",
            "waitAfterCompletion":"15"
         },
         "8-register-tentacle-with-server":{  
            "command":"powershell.exe Set-ExecutionPolicy Unrestricted & powershell.exe C:\\setup\\octopus\\tentacle\\RegisterTentacle.ps1 -env AWSStaging,AWSTraining,AWSDemoWeb -tentaclePrefix AUX -roles Webserver,ImageServer,HangfireServer -tentaclePort 10933",
            "waitAfterCompletion":"0"
         }
      }
   }
```

Notes

- Most of the steps are just tentacle initialization
- Except for 6, it adds a firewall rule for tentacle / server communication
- And step 8 calls our RegisterTentacle.ps1 script

##### configSets

Here is what the config sets look like

```json
"configSets":{  
      "prod-set":[  
         "default",
         "prod"
      ],
      "aux-set":[  
         "default",
         "aux"
      ]
   }
```

Now in the UserData script, when you call cfn-init.exe, you specify the configSet to use, which mine is based off of a parameter in the Cloud Formation script - but YMMV on how you decide to handle this.

Example

```json
"UserData":{  
      "Fn::Base64":{  
         "Fn::Join":[  
            "",
            [  
               "<script>\n",
               "cfn-init.exe -v -s ",
               {  
                  "Ref":"AWS::StackName"
               },
               " -r LaunchConfig",
               " --region ",
               {  
                  "Ref":"AWS::Region"
               },
               " -c ",
               {  
                  "Fn::Join":[  
                     "-",
                     [  
                        {  
                           "Ref":"EnvName"
                        },
                        "set"
                     ]
                  ]
               },
               "\n",
               "cfn-signal.exe --stack ",
               {  
                  "Ref":"AWS::StackName"
               },
               " --region ",
               {  
                  "Ref":"AWS::Region"
               },
               " --resource LaunchConfig",
               "\n"
               
            ]
         ]
      }
   }
```

Also at the KCDC I heard Damian Brady talk about treating servers as cattle instead of pets, which is a bit refreshing. I remember the days of babying servers, upgrading, fixing and bringing back to life.

He encourages you to go out, have a beer and shoot your servers down and watch them pop back up! Once you get this script setup, you can do that with confidence, as it will just spin up a new one! Disclaimer: Make sure you have minimum instances set to at least 2 before you do this.

