---
layout: post
published: false
author: Jack Williams
mathjax: false
featured: false
comments: false
title: AWS AutoScaling with Octopus Deploy
category: AWS
tags:
  - AWS
  - Octopus
  - Autoscale
subtitle: Subtitle
---
## Fear Not, It Can Be Done

Let me first give accolades to Jeffery Palermo. He did a bootcamp on Continuous Delivery at the Kansas City Developer Conferencce that I attended - it really made me re-think our CD process. It was his kindness and helpfulness to other developers and I that reminded me I need to give back as well. I've been a long time reader of his blogs and books, and overall fan. It was really cool to meet him in person and see how much of a helping spirit he has.

I've been meaning to blog about this but just haven't found the time. Now is as good a time as any, I just got done with day one of the KCDC.

### Background

On the project I'm currently working on I had to figure out how to setup autoscaling with Octopus Deploy - and it was no simple feat. There wasn't a ton of information on this topic. This post details the process I came up with to handle this.

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

Let me first give a shout out to Dalmiro Grañas (he's on the support staff at Octopus Deploy, and is awesome!). The RegisterTentacle.ps1 was from him. I can't remember exactly where it is on the interwebs, but I must give credit where credit is due.

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
$OctopusURL = "http://my.octopusserver.com"

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

