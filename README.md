# rockon-registry

This repository consists of Rock-on (Docker based apps) configuration profiles formatted as JSON files.
The Rock-on framework of Rockstor parses a well formatted profile and provides a generic management and workflow such as install, uninstall, update, start and stop.

## Can you show me an example??

Look at any <name>.json file in this repository.
A simpler example is [syncthing.json](https://github.com/rockstor/rockon-registry/blob/master/syncthing.json).
The structure is fairly intuitive though cumbersome.
Using existing examples and the description below of the `json` structure should make it clearer on how this framework is organized.

## How can I add my own Rock-on?

If you are familiar with Docker and know how to deploy docker containers using the command line, you can create a Rock-on for the same with little effort.
You can find instructions in the [Rockstor Documentation](https://rockstor.com/docs/contribute/contribute_rockons.html) both for running your own Rock-on locally, as well as how to contribute to the existing public repository of available Rock-ons

## What is the structure of a Rock-on profile file?

It is a big mass of JSON with nested objects, arrays and values.

### Top level structure
```
{
    "<Rock-on name. eg: LSIO-Plex>": {
      "description": "<description of the Rock-on. Eg: Plex brought to you by Linuxserver.io>",
      "version": "<arbitrary version string>",
      "website": "<Underlying app website>",
      (optional)"icon": "<link to icon, if any>",
      (optional)"more_info": "<string or html with more information to display to the user in the Rockstor UI>",
      (optional)"ui":{
                "slug":"gui", link to webui becomes ROCKSTOR_IP:PORT/gui with slug value gui
      },
      (optional)"volume_add_support": true, If the app allows arbitrary Shares to be mapped to the main container>,
      (optional)"container_links": { <network object representing links between containers using separate docker networks, see below>
      },
      "containers": {
        "<container1 name>": <container object representing the main container. see below>,
        "<container2 name>": <container object representing the second container, if any. see below>, ...
      },
      (optional)"custom_config": <custom configuration object that a special install handler of this Rock-on expects>
    }
}
```

### Structure of the container_links object

The basic structure of the optional `container_links` is:

```
{
  "<target container": [
    {
      "name": "<Name of new docker network 1 to be created",
      "source_container": "<source container 1 that should connect to target container>"
    },
    {
      "name": "<Name of new docker network 2 to be created",
      "source_container": "bareos-storage"
    },
    {
      ...
    }
  ]
}
```

_N.B. Current restrictions: a source/destination/network combination is considered a unique entry:_

_A destination container can have multiple sources, but each of these connections will generate its own docker network.
This essentially provides only a point-to-point connection._

_In future releases this constraint will be removed, so, like with Rocknets, the network becomes the center and any container defined in the Rockon can be connected to it,
allowing for multi-directional communication._

### Structure of a `container` object

_N.B. on the container names.
If you are adding a net new Rockon that is a newer version of an existing one, or in a multi-container scenario uses a container for which a Rockon already exists (e.g. mariadb),
ensure that the container name in your new Rockon is slightly different,
otherwise a duplicate error will be thrown when the final json file is imported._

Each container object is key'd by its name and nested within "containers" of the top level structure above.
A typical container object has the following structure

```
{
  "image": "<docker image. eg: linuxserver/plex>",
  (optional)"tag": "tag of the docker image, if any. latest is used by default.>",
  "launch_order": "1 or above. If there are multiple containers and they must be started in order, specify here.>",
  (optional) "uid": <user>[:<group>] that's used to execute a command or entrypoint inside the container. see below>, 
  (optional)"ports": {
    "<container side port number1>": <port object represending a port mapping between host and container. see below>,
    "<port number2>": <another port object, if necessary. see below>, ...
  },
  (optional)"volumes": {
    "<path1 inside container>": <volume object representing a Share<->directory mapping in the container. see below>,
    "<path2 inside container>": <another volume object, if necessary. see below>, ...
  },
  (optional)"opts": [ An array of option objects that represent container options such as --net=host etc.. see below],
  (optional)"cmd_arguments": [ An array of cmd_arguments objects that represent arguments to pass to the 'docker run' command. See below],
  (optional)"environment": {
    "<env var1 name>": <env object representing one environment variable required by this container. see below>,
    "<env var2 name>": <another env object, if necessary. see below>, ...
  },
  (optional)"devices": {
    "<device1 name>": <device object representing one device to be passed to this container. see below>,
    "<device2 name>": <another device object, if necessary. see below>, ...
  }
}
```

### `uid` element

The optional 'uid' element represents the `--user` option in the `docker run` command that underlies the Rockon execution.
Unless the container image was specifically developed to run with a non-root user, it will automatically be started with it.
This element allows for specifying a user with which the main process or other commands inside the container will be run.

The available combinations are:
- set to `"uid"=""` or omitted, then no overriding action is taken and the container is started with its default user (typically `root`).
- UID (which can be found running `id <username>` at the command line or via the `Users` page in the WebUI.
- `-1` this indicates to the Rockon service that the UID of the owner from the first share, defined in the `volume` object, is used for the `--user` option.
In future releases, this will be extended to be able to take advantage of the full scope that the `--user` option allows.

As it is evident from above, a `container` object also has nested objects for port and volume mappings, container options, command arguments, and environment variables.
These are described below.

### `ports` Object
This optional object can be used for mapping of external ports onto the ports published by the container image.

```
{
  "description": "<A detailed description of this port mapping, why it's for etc..>",
  "label": "<A short label for this mapping. eg: Web-UI port>",
  "host_default": <integer: suggested port number on the host. eg: 8080>,
  (optional)"protocol": "<tcp or udp>",
  (optional)"ui":true,  not needed if false
}
```

_Note that protocol is optional and if it's not present, both tcp and udp ports are mapped simultaneously.
If you wish to allow both tcp and udp, just don't specify protocol in the Port object._

### `volumes` Object
This optional object allows the mapping of Rockstor shares onto volume paths exposed by the container image.
```
{
  "description": "<A detailed description. Eg: This is where all incoming syncthing data will be stored>",
  "label": "<A short label. eg: Data Storage>",
  (optional)"min_size": <integer: suggested minimum size of the Share, in KB>
}
```

### `options` object

An (optional) options object is a list of exactly two elements.
(This needs to be improved or deprecated in favor of more specific design.)

`--net=host` would be represented as:

```
["--net", "host"]
```

Note that the `opts` field is a 2-d array, so the complete line for the above example looks like

```
"opts": [ ["--net", "host"] ],
```

### `cmd` object

The optional `cmd` (command arguments) object is a list of exactly two elements detailing specific arguments to be passed onto the `docker run` command.
As these arguments will simply be appended to the `docker run` command, they need to follow the same syntax and order.
For instance, 

`docker run <...> image/name argument1 argument2="text2"` would be represented as:

```
["argument1", "argument2="text2"]
```

Note that, as for the options object, the cmd_arguments field is a 2-d array, so the complete line for the above example looks like

```
"cmd_arguments": [ ["argument1", "argument2="text2"] ],
```

### `environment` object
This optional object allows to pass environment variables that are exposed by the container image into the container

```
{
  "description": "<Detailed description. Eg: Login username for Syncthing UI>",
  "label": "Web-UI username",
  (optional)"index": <integer: 1 or above. order of this environment variable, if relevant>
}
```


### `devices` object

This optional object allows to pass a specific device to the Rock-on, and similarly to the Environment object, must have a "description" and "label".  

```
{
  "description": "<Detailed description of the device and its intent or specificities. Eg: path to device (/dev/xxx)>",
  "label": "Hardware encoding device",
  (optional)"index": <integer: 1 or above. order of this environment variable, if relevant>
}
```
Note that for the user, filling the fields corresponding to this object during the Rock-on installation is optional, allowing the users to leave the fields blank if not applicable to them.  
