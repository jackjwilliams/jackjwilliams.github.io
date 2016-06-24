---
layout: post
published: false
author: Jack Williams
mathjax: false
featured: false
comments: false
title: AWS AutoScaling with Octopus Deploy
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
More explanation as to why my instances belong to multiple environents and have multiple roles. This project doesn't have a ton of users yet, so spinning up a seperate instance for staging, demo and production, along with seperate database instances for each was too costly. But the infrastructure is in place to do it.

The current configuration is something like this: We have two "environments" that get spun up with a CloudFormation script. There is a parameter that says whether or not the environment is "aux" (which means it hosts demo, staging and training), or "prod" which means it's the production environment. 

This works great for now and can be changed fairly easy when it's time to seriously scale. The main thing is that it's all done in a CloudFormation template, so adding servers is easy. In addition to that, all of the services are modular (there are a few: the Web Server, Image Server, and Hangfire Server). It doesn't matter if they're deployed to IIS on one instance, or three instances - it all works the same.

Sorry for the long intro, lets get down to business.

### The Scripts

Let me first give a shout out to Dalmiro Grañas (he's on the support staff at Octopus Deploy, and is awesome!). The RegisterTentacle.ps1 was from him. I can't remember exactly where it is on the interwebs, but I must give credit where credit is due.

##### RegisterTentacle.ps1

```powershell
param (
	[Parameter(Mandatory=$True)]
	[string[]] $env, # Can be a list
	
	#PROD or AUX
	[Parameter(Mandatory=$True)]
	[string] $tentaclePrefix, # So we can have tentacles named PROD-123.23.23.33 or AUX-123.23.23.33
	
	[Parameter(Mandatory=$True)]
	[string] $tentaclePort = "10933",
	
	[string[]] $roles = @("Webserver"), # Can be a list
	
	[string] $apiKey = "" # Otopus API Key
	
)

# Config - This is used to automatically register the tentacle with the octopus server

# API Key to authenticate in Octopus
$OctopusAPIkey = $apiKey

# Octopus server url
$OctopusURL = "http://my.octopusserver.com" 

# Tentacle URL. Handy way to get the IP using AWS metadata
$TentacleIP = (Invoke-WebRequest http://169.254.169.254/latest/meta-data/public-ipv4).Content 

# So we can have tentacles named PROD-123.23.23.33 or AUX-123.23.23.33
$TentacleName = "$tentaclePrefix-$TentacleIP"

# Port the Listening Tentacle will use to comunicate with the Octopus server
$TentaclePort = $tentaclePort

# Tentacle URL
$TentacleURL = "https://${TentacleIP}:${TentaclePort}/"

# Tentacle Thumbprint (we'll get it later with black magic)
$TentacleThumbprint = ""

# What roles will this tentacle have?
$TentacleRoles = $roles 

# What environments will this tentacle be registered to?
$EnvironmentIDs = $env

$tentacleExe = 'C:\Program Files\Octopus Deploy\Tentacle\Tentacle.exe'

# Get the Tentacles Thumbprint with black magic (I'm no Powershell or regex expert, but this works)
$TentacleThumbprint = & $tentacleExe "show-thumbprint" "--nologo" | Out-String
$TentacleThumbprint = $TentacleThumbprint -match "\:(.*)" ; $matches[1].Trim()

# Octo.exe deploy-release prep. This is used to push the latest release to our new tentacle.
$octoExe = "c:\setup\octopus\octo\octo.exe" # Path to the executable
$octoArg1 = "deploy-release" # What we are doing
$octoArg2 = "--version=latest" # The version we want (actually gets changed later)
$octoArg3 = "--deploy-to" # Deploying to ... which environment?
$octoArg4 = "" # Reserved for the environment
$octoArg5 = "--specificmachines=${TentacleName}" # Only deploy to this tentacle

# Octoposh import
Import-Module Octoposh

# Use Octoposh to start the new machine creation
$machine = Get-OctopusResourceModel -Resource Machine

# Add all environments to the machine
for ($cnt=0; $cnt -lt $env.length; $cnt++) {
	$environment = Get-OctopusEnvironment -EnvironmentName $env[$cnt]
	$machine.EnvironmentIds.Add($environment.id)
}

# Add all roles to the machine
for ($roleCount=0; $roleCount -lt $roles.length; $roleCount++) {
	$machine.Roles.Add($roles[$roleCount])
}

# Assign the machine name
$machine.name = $TentacleName

# Make the new Listening Tentacle Endpoint
$machineEndpoint = New-Object Octopus.Client.Model.Endpoints.ListeningTentacleEndpointResource
$machine.EndPoint = $machineEndPoint # Set endpoint
$machine.EndPoint.Uri = $TentacleURL # Set the machines tentacle URL
$machine.EndPoint.Thumbprint = $TentacleThumbprint # Set the thumbprint

# Finally, create the new resource! Now it's in our Octopus Deploy Server
New-OctopusResource -Resource $machine

#Deploy to each environment passed in
for ($i=0; $i -lt $env.length;$i++){
    # Get environment name
	$octoArg4 = $env[$i]

    # We have to pull the environment info from the octopus server, to get the latest release version
    $octoEnv = Get-OctopusEnvironment $octoArg4

    # Get the latest version number from the environment
    $latestVersion = $octoEnv.LatestDeployment.ReleaseVersion

    # Set the version argument for octo.exe
    $octoArg2 = "--version=$latestVersion"

    # Finally, deploy to our machine
	& $octoExe $octoArg1 $octoArg2 $octoArg3 $octoArg4 $octoArg5
}

```




Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
