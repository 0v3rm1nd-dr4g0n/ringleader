Service Commands
================

Introduction
------------
Service Commands are a way of creating commands (e.g. GCLI commands) from a web service.

Commands are defined in a JSON descriptor (which looks a little like a GCLI [command definition](https://github.com/mozilla/gcli/blob/master/docs/writing-commands.md))

The Descriptor:
---------------
The descriptor is a JSON document containing a list of commands:
```json
{
  "commands":[]
}
```

A command is created for each item in the list. The first command should be empty with the exception of a description: This gives your users information on what your tool does. e.g:

if you load a command with the prefix 'test' and the following descriptor:

```json
{
  "commands":[{"description":"this is an example"}]
}
```

then typing 'test' at the command line will give you "this is an example command".

You probably want commands to be a bit more interesting than this, though. Here's a slightly more interesting example:

```json
{
  "commands":[
  {"description":"this is an example command for use as an example"},
  {
    "name": "command",
    "description": "do something",
    "returnType": "string",
    "execAction": {
      "url": "http://localhost:3000/do/something"
    }
  }
  ]
}
```

In this case, we have a sub-command called 'command' that the user can invoke with 'test command'. The command, when executed, results in a (GET) request being made to the url specified in execAction.

This still isn't very interesting, though. What if we want to be able to supply a parameter? And what if we want to actually see something from the response?  Let's continue by looking at a real world example; a command to create a new session in the ZAP intercepting proxy:

```json
{
  "name": "newsession",
  "description": "create a new session",
  "returnType": "string",
  "params": [{
    "name": "name",
    "type": "string",
    "description": "the name of the new session to create"
  }],
  "execAction":{
    "url":"http://localhost:8080/JSON/core/action/newSession/?zapapiformat=JSON&name=${$.args.name}",
    "expression":"$.Result"
  }
}
```

The first thing to notice here is that we are able to specify parameters. Here we have a single parameter called 'name'. String parameters can have any value but it's possible to limit the possible values (and even have default). This will be covered later on.

The second is that we're using the parameter in the url of the execAction - notice '${$.args.name}' on the end of the URL? This a JSONPath expression which will be evaluated against the command's data, the result of which will be substituted with the value the user enters as a command parameter. All you have to know about this for the time being is that $.args.PARAM gets the value of the 'PARAM' parameter. 

Finally, notice "expression" there in execAction - you can specify a JSONPath expression (the tool supports a safe subset of JSONPath) to extract data from the response to give to the user (as the output for the command).


More on Parameters:
-------------------

You can limit the possible values for a parameter by using providing an object (rather than 'string') to as the type. For example:

```json
{
  "name": "param1",
  "type": {
    "name": "selection",
    "data": ["name1", "name2", "name3"]
  },
  "description": "you may only name it name1, name2 or name3",
  "defaultValue": "name2"
}
```
Here we have a parameter called param1 which can take the values name1, name2 or name3 - if the user does not specify a value it will default to name2.

Contextual data in Commands:
----------------------------

Sometimes you might want to make use of certain contextual data in your commands. In most places where it's possible to extract data for command actions (see Tool Operations) or substitution into an URL (e.g. command execAction URLs) you can provide some kind of expression (see use of $.args.name above). In these cases, it will be possible to get certain contextual data. For example, the 'key' attribute of the command data object contains an identifier that is unique for a tab. This can be useful when, for instance, you want to identify a specific tab in your service.

Consider the following command:

```
{
  "name":"something",
  "description":"set something",
  "returnType":"string",
  "params":[
    {
      "name":"state",
      "type": {"name":"selection", "data":["on","off"]},
      "description": "should something be enabled?",
      "defaultValue": "on"
    }
  ],
  "execAction":{
    "url":"http://localhost/something?tab=${$.key}&state=${$.args.state}"
  }
}
```

Here the tab request parameter will be unique per tab when the command is issued by the user.

Tool operations:
----------------

The Service Commands functionality used by this tool supports [callback data](https://github.com/mozmark/ServiceTest/blob/master/doc/service_commands.md#callback-data) being sent back to the embedding tool. PNH makes use of this by providing operations that can be invoked from your commands via callbackData.

This is best illustrated with an example:

```json
{
  "name": "brk",
  "description": "create a new session",
  "returnType": "string",
  "params": [{
    "name": "state",
    "type": {
      "name": "selection",
      "data": ["on", "off"]
    },
    "description": "break on request",
    "defaultValue": "on"
  }, {
    "name": "scope",
    "type": {
      "name": "selection",
      "data": ["tab", "global"]
    },
    "description": "local to tab or global",
    "defaultValue": "tab"
  }],
  "execAction": {
    "expression": "$.Result",
    "callbackData": {
      "conditionalCommands": {
        "expression": "$.args.state",
        "states": {
          "on": [{
            "command": "addToHeader",
            "params": {
              "headerName": "X-Security-Proxy",
              "value": "intercept",
              "scope": {
                "type": "expression",
                "expression": "$.args.scope",
                "extract": true
              }
            }
          }],
          "off": [{
            "command": "removeFromHeader",
            "params": {
              "headerName": "X-Security-Proxy",
              "value": "intercept",
              "scope": {
                "type": "expression",
                "expression": "$.args.scope",
                "extract": true
              }
            }
          }]
        }
      }
    }
  }
}
```

This is a description for the 'brk' command which causes ZAP (with the mitm-config addon) to break on request / response.  callbackData has an attribute called "conditionalCommands" which specifies an expression and a map of states to lists of commands. If the result the expression matches a state, the associated commands will be invoked.

At present only the 'addToHeader' and 'removeFromHeader' commands are supported. This list will be expanded in time.
