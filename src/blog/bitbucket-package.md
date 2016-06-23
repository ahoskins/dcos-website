---
title: Building A DCOS Universe Package
date: 2016-06-23
author: Andrew Hoskins, Mesosphere
category: services
description: A step-by-step guide for creating a Bitbucket package
layout: article.jade
collection: posts
lunr: true
---

Universe is the DCOS package repository containing services like Spark, Cassandra, Jenkins, and many others.  In this post, we'll walk through the process of creating a Bitbucket package.   

## Getting Started
Packages are published to the Mesosphere Universe repository on Github.  To get started, fork [this](https://github.com/mesosphere/universe) repository, then clone the fork:
    
    $ git clone https://github.com/<your-username>/universe 

Look around the universe repository &mdash; it's mostly JSON files.  Within `/repo/packages` is published packages arranged alphabetically. 
Create a new directory for the Bitbucket package (`repo/packages/B/bitbucket/0`).  The `0` is the package revision number.
Within this directory we will create 4 JSON files, which is everything needed to create a consumable DC/OS package. 

## Step One: package.json

This file specifies the highest-level metadata about the package (comparable to a package.json in Node.js or setup.py in Python). 
Let's go ahead and create it in the most basic way.

```javascript
{
  "packagingVersion": "2.0",
  "name": "bitbucket",
  "version": "4.5",
  "scm": "https://www.atlassian.com/software/bitbucket",
  "description": "Bitbucket package",
  "maintainer": "me@me.com",
  "tags": [],
  "preInstallNotes": "Preparing to install bitbucket-server.",
  "postInstallNotes": "Bitbucket has been installed.",
  "postUninstallNotes": "Bitbucket was uninstalled successfully."
}
```

The `packagingVersion` field specifies the version of universe package spec to adhere to.  The two available version are `2.0` or `3.0`. 
See the [schema](https://github.com/mesosphere/universe/tree/version-3.x/repo/meta/schema) on Github for details.

## Step Two: resource.json

This file declares all the externally hosted assets the package will need &mdash; things like docker containers, png images, etc.  
In our case, we want our Bitbucket package to have an icon and we will also want to use a pre-existing Bitbucket container.

```javascript
{
  "images": {
    "icon-small": "http://i.imgur.com/QGc420u.png",
    "icon-medium": "http://i.imgur.com/QGc420u.png",
    "icon-large": "http://i.imgur.com/QGc420u.png"
  },
  "assets": {
    "container": {
      "docker": {
        "bitbucket-docker": "atlassian/bitbucket-server:4.5"
      }
    }
  }
}
```

We found this container on [Docker Hub](https://hub.docker.com/r/atlassian/bitbucket-server/).
The png images sizes should be 48x48, 96x96, and 256x256, respectively for small, medium, and large.

## Step Three: config.json

This file declares the packages configuration properties, such as the amount of cpus, number of instances, and allotted memory.
These values can be overriden at install time by passing command-line flags, or through DC/OS UI.

```javascript
{
  "$schema": "http://json-schema.org/schema#",
  "properties": {
    "bitbucket": {
      "properties": {
        "instances": {
          "default": 1,
          "description": "Number of instances to run.",
          "minimum": 1,
          "type": "integer"
        },
        "cpus": {
          "default": 2,
          "description": "CPU shares to allocate to each bitbucket instance.",
          "minimum": 2,
          "type": "number"
        },
        "host-volume": {
            "description": "The location of a volume on the host to be used for persisting Bitbucket data. The final location will be derived from this value plus the name set in `name` (e.g. `/mnt/host_volume/bitbucket`). Note that this path must be the same on all DCOS agents.",
            "type": "string",
            "default": "/tmp"
        },
        "mem": {
          "default": 2048.0,
          "description": "Memory (MB) to allocate to each bitbucket task.",
          "minimum": 2048.0,
          "type": "number"
        },
        "minimumHealthCapacity": {
          "default": 0.5,
          "description": "Minimum health capacity.",
          "minimum": 0,
          "type": "number"
        },
        "maximumOverCapacity": {
          "default": 0.2,
          "description": "Maximum over capacity.",
          "minimum": 0,
          "type": "number"
        },
        "name": {
          "default": "bitbucket",
          "description": "Name for this bitbucket application",
          "type": "string"
        },
        "role": {
          "default": "*",
          "description": "Deploy bitbucket only on nodes with this role.",
          "type": "string"
        },
        "virtual-host": {
            "description": "The virtual host address to configure for integration with Marathon-lb.",
            "type": "string"
        }
      },
      "required": ["cpus", "mem", "instances", "name"],
      "type": "object"
    }
  },
  "type": "object"
}
```

## Step Four: marathon.json.mustache

This file is written in the [mustache templating language](https://mustache.github.io/mustache.5.html), and will compiled to JSON later.
The reason it's written in a higher-level templating language is because it references objects and variables declared in `config.json`.
For example, in our `config.json`, the instances property can be accessed from the `marathon.json.mustache` file.

    ...
    "instances": {{bitbucket.instances}},
    ...

When the mustache is compiled, it will replace this special tag with the value of `bitbucket.instances` declared in `config.json`.

Below, is the entire `marathon.json.mustache` needed for our Bitbucket pacakge:

```javascript
{
  "id": "/{{bitbucket.name}}",
  "instances": {{bitbucket.instances}},
  "cpus": {{bitbucket.cpus}},
  "mem": {{bitbucket.mem}},
  "maintainer": "me@me.com",
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "{{resource.assets.container.docker.bitbucket-docker}}",
      "network": "BRIDGE",    
      "portMappings": [
        { "containerPort": 7990, "hostPort": 0 },
        { "containerPort": 7999, "hostPort": 0 }
      ]
    },
    "volumes": [
    {
        "containerPath": "/var/atlassian/application-data/bitbucket",
        "hostPath": "{{bitbucket.host-volume}}",
        "mode": "RW"
    }
    ]
  },
  "healthChecks": [
    {
      "protocol": "COMMAND",
      "command": { "value": "curl --fail ${HOST}:${PORT0}" },
      "gracePeriodSeconds": 300,
      "intervalSeconds": 60,
      "timeoutSeconds": 20,
      "maxConsecutiveFailures": 3
    }
  ],
  "acceptedResourceRoles": [
    "{{bitbucket.role}}"
  ],
  "labels": {
    {{#bitbucket.virtual-host}}
    "HAPROXY_GROUP":"external",
    "HAPROXY_0_VHOST":"{{bitbucket.virtual-host}}",
    {{/bitbucket.virtual-host}}
    "DCOS_SERVICE_NAME": "{{bitbucket.name}}",
    "DCOS_SERVICE_SCHEME": "http",
    "DCOS_SERVICE_PORT_INDEX": "0"
  }
}
```

## Step Five: install your package

At this point, the Bitbucket package has all the files needed to be running on DC/OS.












