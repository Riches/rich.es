+++
title = "Accessing the instance number from within containers deployed with CapRover"
date = "2025-05-20"
tags = ["CapRover", "Docker", "Docker Swarm"]
+++

I came across an issue today whereby I wanted to run Wireguard within my Docker containers within CapRover. However,
there was an issue. There is no way to reliably assign separate configs to multiple instances of the same app
(Additional replicas from setting a value in the 'Instance Count' box). There's nothing in the container which is both
consistent between deploys yet also unique enough to distinguish multiple containers.

Luckily, CapRover runs on top of Docker Swarm which exposes some helpful variables to distinguish containers in the way
that I require.

## In short

Inside CapRover:

1. Open your app.
2. Go to the 'App Configs' tab.
3. Add an environment variable:
    - **Key**: `TASK_SLOT`
    - **Value**: `{{.Task.Slot}}`
4. Save and update.

That's it, TASK_SLOT is exposed as an environment variable within your container. It will be an integer which reflects
its instance number (starting at 1).

## Example Usage

You can access it just like any other env var. In Node:

`const slot = process.env.TASK_SLOT;`

Or in Python:

```
import os
slot = os.environ.get("TASK_SLOT")
```

## Why this works

As mentioned earlier, CapRover is built on Docker Swarm, and Swarm supports a few template variables in service definitions. One of those is {{.Task.Slot}}. If you use it in an environment variable, Swarm evaluates it at runtime when creating the container.

A few other variables worth knowing:
* {{.Service.Name}} - The name of the CapRover app
* {{.Task.ID}} - The full Docker task ID (long and ugly but unique)
* {{.Node.ID}} - The ID of the Docker node the container is running on

You can use these in the same way, in the 'Environment Variables' section in your CapRover app.