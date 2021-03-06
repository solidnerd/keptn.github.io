---
date: "2020-10-12T14:26:31+02:00"
title: "keptn get service"
slug: keptn_get_service
---
## keptn get service

Get service details

### Synopsis

Get all services or details for a given service within a Keptn project

```
keptn get service [flags]
```

### Examples

```
keptn get service carts --project=sockshop
NAME           CREATION DATE                 
carts          sockshop        2020-05-28T10:25:58+02:00

keptn get services                                   # List all services in keptn

keptn get services --project=sockshop                # List all services in the sockshop project

keptn get services carts --project=sockshop -o=json  # Get details of the carts service in the sockshop project as json output

```

### Options

```
  -h, --help             help for service
  -o, --output string    Output format. One of json|yaml
      --project string   keptn project name
```

### Options inherited from parent commands

```
      --mock                 Disables communication to a Keptn endpoint
  -q, --quiet                Suppresses debug and info messages
      --suppress-websocket   Disables WebSocket communication to suppress info messages from services running inside Keptn
  -v, --verbose              Enables verbose logging to print debug messages
```

### SEE ALSO

* [keptn get](../keptn_get/)	 - Displays an event or Keptn entities such as project, stage, or service

###### Auto generated by spf13/cobra on 12-Oct-2020
