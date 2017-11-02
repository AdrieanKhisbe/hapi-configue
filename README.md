# Configue

> ***CONFIGUE ALL THE THINGS \o/***

[![npm version][npm-badge]][npm-url]
[![Build Status][travis-badge]][travis-url]
[![Coverage Status][coverage-badge]][coverage-url]
[![Code Climate][codeclimate-badge]][codeclimate-url]
[![Dependency Status][david-badge]][david-url]
[![NSP Status][nsp-badge]][nsp-url]
[![bitHound Overalll Score][bithound-badge]][bithound-url]

[Configue] is a node.js Config library to easily customize your app with argv, env, files and more.

It's a wrapper on [nconf] node hierarchical config tool.

It defines a *conventional workflow* to load a config from environment variables,
command line arguments, files, that you can easily *configure* and *extend*.

## About Configue

[Configue] builds up on [nconf] and its
[Hierarchical configuration](https://github.com/indexzero/nconf#hierarchical-configuration) system.

It defines a list of _configuration step_ that are executed in order.
Every property defined on a steps will shadow the same key in the following steps.

Quoting [nconf]:
> ***"The order in which you attach these configuration sources determines their priority in the hierarchy"***

Here are the standard steps [Configue] does define:
- `overrides` : properties that would take precedence on all other steps
- `argv` : command line options
- *configueFile* : file specified by `--configue` in argv if any.
- `env` : environment variables
- `file` : config files
- `defaults` : default objects

The plugin loads the various configurations in order using _predefined steps_.

It starts by parsing *argv* then goes through the *env* and the files options
and finishes by loading the default config objects if any.

If `--configue` option is specified, the config the specified file holds would
be loaded after the *argv* and before the *env*. This is to enable you to save
many options in many files, and specify at launch with options you want to use

Hence why every option defined as an argument commandline will override defaults
and environment variables.

## Installation

Just add `configue` has a dependency installing it with npm, or with yarn.

    npm install --save configue
    
    yarn add configue

## Usage

### How To

To use _Configue_ you need first to create an instance passing the option to the `Configue(opts)`
constructor. Resolving of the config is now done synchronously and automaticly unless you specify
the `defer: true` option. In case of an error it will throw an Error.

See the following examples for concrete presentation.

### Basic usage without customization
#### Basic Configue
```js
const Configue = require('configue');
const configue = new Configue();

const who = configue.get('who', 'World');
console.log('Hello ' + who);

```

#### Passing By Options
You can specify the `who` configue in different manners.
Here are some:

```sh
node basic.js --who=Woman
# configue through Env
export who=Man ; node basic.js
who=Human node basic.js
node basic.js --configue=my-who-conf.json
```

The full example is available in the [`examples`](./examples/basic.js) folder.

### Retrieving values
You can retrieve values from the store in different manner, `get` is the most simplee one

#### Simple `get`

To retrieve a configue value, use the `get` method on the config holder.
It takes has argument the key of the argument. For nested value you need to
use `:` to deep access to value.
It's also possible to specify a default value in case key is `undefined`.

```js
configue.get('defined4sure')
configue.get('some:nested:value')
configue.get('mightBeUndefined', 'default')
```

You can also retrieve a list of value with `getAll`, or the first non undefined value from a list with `getFirst`

```js
configue.getAll('defined4sure', 'some:nested:value')
configue.getAll(['defined4sure', 'some:nested:value'])

configue.getFirst('defined4sure', 'some:nested:value')
configue.getFirst(['defined4sure', 'some:nested:value'], optionalDefaultValue)
```

#### Retrieving Specified Object

When you can to retrieve several values in the same time you can forge object so that they have structure you need.

##### Load and getObject for ponctual retrieval
The two main methods are `load` and `getObject`
- `load` by default return the whole merged config as an object. But you can give him a model that would be used 
  to craft an object. The model is a object whose leaves are configue keys, or array of configue key:
    ex: `{serverConfig: {host: 'app:server:host', port: 'PORT'}, ...}`
- `getObject` that takes a list of key, and return an object formed by key / values

```js
const {serverConfig} = configue.load({serverConfig: {host: 'app:server:host', port: 'PORT'}, extraOptions: "..."});

const {file, prefix} = configue.getObject('file', 'prefix');
```
##### *Models* for frequent usage
There are case where the forged object are to be used several times, and you dont want to query them over and over.
To do that you can predefined *models* in the configuration. These would be populated once during automatic resolved,
and they would be made accessible under the `_` key.

```js
const configue = new Configue({models: {
    serverConfig: {host: 'app:server:host', port: 'PORT'},
    otherModel:{a:'a', b:'...'}}});
//...
console.log(configue._.serverConfig) // => host: ..., port: ...
```

#### Template string
One last way you can get config value is via the `configue.template` (aliased to `configue.t`).
This is a template function you can prefix a template string. The interpolated values will be keys of the
configue and then remplaced by their value:

```js
console.log(configue.t`I will say ${'salute'} to ${'who'}`); 
// => I will say Hello to You     
// (supposing called with --salute=Hello --who=You)
```
You can defined default values by passing a default object to the `template` method:
```js
console.log(configue.t({times: 2, who: 'World'})`I will say ${'salute'} to ${'who'} ${'times'} times`); 
// => I will say Hello to You 2 times
```

### Usage with customization of the configuration workflow

#### Passing options to nconf
You can provide options arguments to `argv` (`yargs`underneath), and `env` in order to customize the behavior 
around command line argument and environment variables.
For more in depth readings see nconf options [here][nconf-options-argv-env]

```js
const Configue = require('configue');

const configueOptions = {
    argv: { f: {
                 alias: 'file',
                 demandOption: true,
                 default: '/etc/passwd',
                 describe: 'x marks the spot',
                 type: 'string'
                 }},
    env: ["HOME", "PWD"] // whitelist

};

const configue = new Configue(configueOptions);
```

#### Specifying Files

The files key can contain a single object or an array of objects containing a `file` key containing the path to the config file.
The object can also reference a nconf plugin tasked with the formatting using the key `format`.

Starting from 1.0 the formater to use can be automaticaly deduced for standard files. Supported extensions are
`json`, `yaml`/`yml` but also `properties`/`ini` and `json5` In that case you just need to specify the name of the file.

```js
const Configue = require('configue');

const configueOptions = {
    disable: {argv: true},
    files: [
        {file: './config.json'},
        {
            file: './config.yaml',
            format: require('nconf-yaml')
        },
        'my-own.propeties'
    ]
};

const configue = new Configue(configueOptions);
````

Note that if only one file is needed, its path can be directly given as options.

#### Disabling Steps

The argv and env steps can be skipped using the `disable` object in `options`.

```js
const configue = new Configue({ disable: { argv: true }});
// ...
```

There is no disabling for `overrides`, `files` and `default`; you just have to don't provide the matching option.

#### Step hooks

Every step (`overrides`, `argv`, `env`, `files`, `defaults`)has a post hook available.
Those can be defined using the `postHooks` key and accept a
function that take `nconf` and a callback as parameters.

The special hooks `first` enables you to respectively apply a function on nconf at the very beginning.

```js
const configue = new Configue({
        postHooks: {
            first: function first(nconf, done){
                // Your code here
            },
            overrides: function postOverrides(nconf, done){
                // Your code here
            },
            argv: function postArgv(nconf, done){
                // Your code here
            }
        }
});
// Your code here
```

#### Custom Workflow

If needed you can have your full custom configuration workflow,
simply by providing an object with the single key `customWorkflow`
attached to a function taking the `nconf` object, and a `done` callback.

```js
const configueOptions = { customWorkflow: function(nconf, done){
  // my own config setting
}};

const configue = new Configue(configueOptions);
```

### Loading into Hapi

Thought _Configue_ is usable without hapi, (it was originally just a _Hapi_ plugin),
it can be easily loaded in hapi to have the _configue_ being easily accessible from
the server, or on the request.

To do this, you need to register the plugin. It takes care to resolve the config.
```js
const Hapi = require('hapi');
const Configue = require('configue');

const configue = new Configue({some: 'complex config with a model connexion'});

const server = new Hapi.Server();
server.connection(configue._.connexion); // note usage of the model connexion with port in it.
server.register({register: configue.plugin()}, (err) => {
      // starting the server or else
       
      // access to the config
      const config = server.configue('some'); // => 'config'
      const configGet = server.configue.get('some'); // => 'config'
      // Any other call to server.configue.getAsync/getFirst/getAll/getObject/template/load
      // ...
})
```
A more complete example is available in [`examples`](examples/hapi-server.js) folder.

Note it's possible to provide to `configue.plugin()` a `decorateName` so that you use a custom accessor on `server` or `request`.

### Loading into express
Configue can also be loaded into `express` via it's middleware you can obtain by `configue.middleware()` you just have
to feed to app.use()`

A example is available in the [`examples`](examples/express-server.js) folder.

## Configuration Recap
Configue can be configured into two different way. Either using a config object or using a fluent builder.

### Configuration Object
Here is a recap of how the configuration should look like. All options are optional:

- `customWorkflow`: a function manipulating the `nconf`. This option is exclusive of all others
- `argv`: Config object for `yargv`, a map of config key with an object values (`alias`, `demandOption`, `default`,`describe`, `type`)
- `env`: The options for the `nconf.env` method that can be:
  - a string: separator for nested keys
  - an array of string, the whitelisted env method
  - an object with key: `separator`, `match`, `whitelist
- `disable`: A object with key `argv` and/or `env` attach to a boolean indicated whether the step shouldbe disable. 
        Default to true.
- `files`: file or list of files. (object `file`, `format`)
- `defaults`: Object of key mapped to default values. (or array of them)
- `overrides`: Object of key mapped to overrides values.
- `required`: list of key that are required one way or another
- `postHooks`: an object of (`step`: function hook)
    step being one of `first`, `overrides`, `argv`, `env`, `files` `defaults`

For more details you can see the `internals.schema` in the `configue.js` file around the line 100

### Fluent builder

Instead to use the configuration object provided to the `Configue` constructor, you can use the fluent builder.
This consist in chaining a list of configuration methods before to retrieve the instance to a `get` method.

Here is a simple example:

```js
const configue = Configue.defaults({a: 1})
            .env(["HOME"])
            .get();
```

Here is the builder function list, the function name being the name of the key in he object config (except the postHooks function):
`argv`, `customWorkflow`, `defaults`, `overrides`, `disable`, `env`, `files`, `required`
and `firstHook`, `overridesHook`, `argvHook`, `envHook`, `filesHook`, `defaultsHook`


[Configue]: https://github.com/AdrieanKhisbe/configue
[github-repo]: https://github.com/AdrieanKhisbe/configue
[nconf]: https://github.com/indexzero/nconf
[nconf-options-argv-env]: https://github.com/indexzero/nconf#argv

[npm-badge]: https://img.shields.io/npm/v/configue.svg
[npm-url]: https://npmjs.com/package/configue
[travis-badge]: https://api.travis-ci.org/AdrieanKhisbe/configue.svg
[travis-url]: https://travis-ci.org/AdrieanKhisbe/configue
[david-badge]: https://david-dm.org/AdrieanKhisbe/configue.svg
[david-url]: https://david-dm.org/AdrieanKhisbe/configue
[codeclimate-badge]: https://codeclimate.com/github/AdrieanKhisbe/configue/badges/gpa.svg
[codeclimate-url]: https://codeclimate.com/github/AdrieanKhisbe/configue
[coverage-badge]: https://codeclimate.com/github/AdrieanKhisbe/configue/badges/coverage.svg
[coverage-url]: https://codeclimate.com/github/AdrieanKhisbe/configue/coverage
[bithound-badge]: https://www.bithound.io/github/AdrieanKhisbe/configue/badges/score.svg
[bithound-url]: https://www.bithound.io/github/AdrieanKhisbe/configue
[nsp-badge]: https://nodesecurity.io/orgs/adrieankhisbe/projects/ca973c87-f742-4956-80c9-b402b8a76adb/badge
[nsp-url]: https://nodesecurity.io/orgs/adrieankhisbe/projects/ca973c87-f742-4956-80c9-b402b8a76adb
