# source-configs
A Node.js module that provides structure to define app-level configuration schemas and automatically consume values via multiple sources in Node.js applications. This module was built and is maintained by the [Roosevelt web framework team](https://github.com/rooseveltframework/roosevelt), but it can be used independently of Roosevelt as well. 

## Install

Declare `source-configs` as a dependency in your app.

## Usage

### Basic schema

Create a JS file that exports an object that defines your schema as in this example:

```js
module.exports = {
  websocket: {
    host: {},
    port: {},
    protocol: {}
  }
}
```

In the above example, we are declaring that our application will use a WebSocket with a configurable host, port, and protocol.

### Config property metadata

Next we can define some metadata for each configurable property in order to create constraints if desired or add additional functionality. source-configs offers the following property attributes:

- `description` *[String]*: A place to explain what this configuration variable will be used for.
- `default` *[any]*: Default value. Will be set to null if not set otherwise defined.
- `values` *[Array]*: Enumerated list of values that are valid. If not set, any value will be valid.
- `commandLineArg` *[String]*: The name of a command line argument. The names are expected to be camelCased (ex: the command line argument `--disable-validator` would be set here as `disableValidator`) and will be obtained from a external command line args parser. If not set, source-configs will not listen for command line args for this variable.
- `envVar` *[String]*: The name of an environment variable that can set this configuration variable. If not set, source-configs will not listen for an environment variable to set the value for this variable.

- `envVarParser` *[Function]*: A function to parse environment variables or a string which will be used as a delimiter for simple delimiter split strings. This can only be implemented when writing the schema in a JS file or overwritten before being initialized.

Also, each configurable property can be a function which takes the parent scope config as a parameter to create strings based upon other config primitives.

### Example with config property metadata

```js
module.exports = {
  websocket: {
    host: {
      description: 'WebSocket host URL',
      default: 'localhost',
      envVar: 'WS_HOST'
    },
    port: {
      description: 'WebSocket port',
      default: 8081,
      envVar: 'WS_PORT'
    },
    protocol: {
      description: 'Which WebSocket protocol',
      values: ['ws', 'wss'],
      envVar: 'WS_PROTOCOL'
    },
    allowAnyOrigin: {
      description: 'Boolean to define if any origin can transmit to the websocket.',
      commandLineArg: 'allowAnyOrigin',
      default: false
    },
    allowedOrigins: {
      description: 'List of origin domains allowed to connect to this websocket. Environment variable is defined as a comma separated list',
      default: [],
      envVar: 'WS_ALLOWED_ORIGINS',
      envVarParser: ',' // WS_ALLOWED_ORIGINS='https://google.com,https://github.com' will be parsed into ['https://google.com/', 'https://github.com']
    },
    allowedOriginsDetailed: {
      description: 'List of origin domains allowed to connect to this websocket that is parsed out into objects. Environment variable is defined as a comma separated list',
      default: [],
      envVar: 'WS_ALLOWED_ORGINS_DETAILED',
      envVarParser: (envVar) => {
        let originStrings = envVar.split(',');

        let origins = [];

        originStrings.forEach(originString => {
          let [protocol, rest] = originString.split('://')

          // If protocol is empty, set it to 'ws'
          if (protocol === originString) {
            protocol = 'ws'
            rest = originString
          }

          let [host, port] = rest.split(':')

          // If port is empty, set it to 80
          if (port === undefined) {
            port = 80
          } else {
            port = parseInt(port)
          }

          let origin = {
            protocol: protocol,
            host: host,
            port: port
          }

          origins.push(origin)
        })

        return origins
      }
    }
    fullUrl: (config) => {
      return `${config.protocol}://${config.host}:${config.port}`
    }
  }
}
```

### Use in your app

Here's an example usage of source-configs using the schema defined above:

```js
const sourceConfigs = require('source-configs')

// Grab command line arguments. yargs-parser is used in this example but the minimum requirement is
// a parser that converts command line arguments to a flat key-value map where the keys are camelCased
const commandLineArgs = require('yargs-parser')(process.argv.slice(2))

const config = sourceConfigs.init({ schema: require('./your-schema-js-file.js'), commandLineArguments: commandLineArgs })

// access one of the configs
console.log(config.websocket.port)
```

## Where configs are sourced from 

Configs matching the schema are sourced from the following locations in the following order of precedence:

- Command line argument set in the schema under `commandLineArg`.
- Environment variable set in the schema under `envVar`.
- Deployment config file declared via:
  - Command line argument: `--deploy-config` or `--dc`.
  - Environment variable: `SC_DEPLOY_CONFIG`.
  - package.json as a `deployConfig` field.
- Default defined in the schema.