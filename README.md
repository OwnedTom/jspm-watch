# JSPM Watch

Inspired by [jspm-dev-builder](https://github.com/jackfranklin/jspm-dev-builder) JSPM Watch keeps your code changes in immediate sync with the browser no matter how big your app is and how many dependencies you have. JSPM Watch is designed for a long run: no matter what you do to your project files, whenever you add or remove new dependencies, have compilation errors, or simply switch branches - it should keep running, with no need to ever restart it. There's also an option to do simultaneous unit tests build, so you can do *tdd* and run the app at the same time.

## Why
As your project grows it may take enormous amount of time to reload the application after a file change. JSPM Watch solves that issue by pre-bundling the app, which takes some time at first, but then upon any file change it invalidates it from cache and re-bundles the app, which takes about a second.

### Table of Contents
- [Features](#features)
- [Requirements](#requirements)
- [Install](#install)
- [Usage](#usage)
- [Documentation](#documentation)
- [Events](#events)
- [Change log](#change-log)

## Features

- proper dependencies addition/removal handling: ideally, no need to restart the process **ever**
- minimum and straightforward configuration
- hassle-free file watching with built-in chokidar
- both windows and unix support
- unit tests build support
- proper compilation error handling
- informative console output
- non-javascript modules support - css and html

Other solutions like `jspm-dev-builder` or `systemjs-hot-reloader` lack these features (especially first one).

## Requirements

Installed JSPM of at least 0.16.12 version within the project.

## Install

```sh
npm i jspm-watch
```

## Usage

Please refer to the test-project for full usage example.

### Minimum configuration

```javascript
var JspmWatch = require('jspm-watch');

watcher = new JspmWatch({
   app: {
      input: 'src/my-app.js',
      output: 'dist/my-app.js'
   }
});

watcher.start();
```
> Tip: The "input" attribute of the "app" key, is optional if [jspm.main](https://github.com/jspm/registry/wiki/Configuring-Packages-for-jspm#main-entry-point) is defined in package.json

### Extended configuration with unit tests build

```javascript
var JspmWatch = require('jspm-watch');

watcher = new JspmWatch({
   app: {
      input: 'src/my-app.js',
      output: 'dist/my-app.js',
      buildOptions: {
        minify: false,
        mangle: false,
        sourceMaps: false
      }
   },
   tests: {
      watch: 'src/**/*.spec.js',
      input: 'src/unit-tests.js', // this file will be auto-generated
      output: 'dist/unit-tests.js' // this file should be consumed by your test runner e.g. karma
   },
   batchDelay: 250 // build delay when processing batch changes
});

watcher.start().on('change', function(event) {
    if (!event.hasError) {
       if (event.type == 'app') {
          browserSync.reload();
       }
       if (event.type == 'tests') {
          karmaServer.refreshFiles();
       }
    }
});
```

## Documentation

### Constructor
Returns an instance of JSPM Watch.
```javascript
var JspmWatch = require('jspm-watch');

watcher = new JspmWatch({
    app: {
        watch: ['src/**/*.js', 'src/**/*.html', 'src/**/*.css'],
        input: 'src/my-app.js',
        output: 'dist/my-app.js'
    }
});
```

### Constructor options
#### app.watch (optional)
Since 0.2.0 by default jspm-watch uses SystemJS Builder `trace()` method to watch file changes. Use `app.watch` to override it and provide an array or string of files to watch that will be passed to chokidar. You don't need to exclude `jspm_packages`, it's done by default. Tests files are excluded by default if `tests` section is provided.
#### app.input
You app entry point.
#### app.output
You app ouput destination file .
#### app.buildOptions
Build options which are going to be passed to SystemJS Builder. Default options are:
```javascript
{
    sfx: false,
    minify: false,
    mangle: false,
    sourceMaps: true,
    lowResSourceMaps: true
}
```
`sfx` option is non-overridable, self extracting bunding is not supported at the moment. https://github.com/jackfranklin/jspm-dev-builder/issues/9

#### tests.watch
Unit tests files to watch. Don't need to provide app files pattern if `app.watch` is configured:
```javascript
watcher = new JspmWatch({
    app: {
       watch: ['src/**/*.js']
    },
    tests: {
       watch: ['src/**/*.spec.js'],
       input: 'src/unit-tests.js',
       output: 'dist/unit-tests.js'
    }
});
watcher.start();
```
#### tests.input
Path for a file which will be auto-generated by JSPM Watch with imports of all spec files.

#### tests.output
Tests bundle output destination. Should be consumed by tests runner.

#### tests.buildOptions
Same as `app.buildOptions`.
#### batchDelay
Delay in milliseconds before build will be fired. Default is `250`.

#### debug

Turns on debug output. Useful when trying to understand problems. Default is `false`.

### start([options])
Returns instance of JSPM Watch for chaining.

Does initial build and starts watching for file changes. If no options provided watches both tests and app changes.
```javascript
watcher.start();
```

#### Start options
Start options are usefull when you have watcher configured just once and then you have different gulp (or any other) tasks.
#### App only
Tells watcher to watch only app changes ingnoring tests.
```javascript
watcher.start({ appOnly: true });
```
#### Tests only
Tells watcher to watch only tests changes ingnoring app. 
```javascript
watcher.start({ testsOnly: true });
```
## emitter
JSPM Watch manages events with node EventEmitter which is stored in `emitter` property.
```javascript
watcher.emitter.on('started', method);
```
However, there are shorthand methods for `on` and `once`:
```javascript
watcher.on('started', method); // equivalent of watcher.emitter.on
watcher.once('change', method);  // equivalent of watcher.emitter.once
```

## Events 

### started
Emitted when JSPM Watch started and done initial build (both app and tests if configured). May be usefull for task runner.
```javascript
gulp.task('watch', function(done) {
    watcher.start().on('started', function() {
       done();
    }); 
})
```

### change
Emitted when JSPM Watch detected file change and perfomed a build. Even if build fails with error, event is still emitted. Provides event object with two properties: `type` and `hasError`.
```javascript
watcher.start().on('change', function(event) {
   if (!event.hasError) {
       if (event.type == 'app') {
          browserSync.reload();
       }
       if (event.type == 'tests') {
          karmaServer.refreshFiles();
       }
    }
}); 
```
#### event.type
Possible options are `'app'` and `'tests'`. Indicates which build has happend.
### event.hasError
Indicates whether build has compilation error.

### beforeBuild
Emitter when JSPM Watch is going to perform a build of any kind. May be usefull for some build pre-hook actions. Provides event object with two properties: `type` and `buildState`.
```javascript
 watcher.start().on('beforeBuild', function(event) {
   if (event.type == 'app') {
        fs.closeSync(fs.openSync(paths.src+'/styles.css', 'w'));
        jspmWatch.once('change', function () {
            fs.unlinkSync(paths.src+'/styles.css');
        });
    }
}); 
```
#### event.type
Possible options are `'app'` and `'tests'`. Indicates which build has happend.

#### event.buildState
An object, which properties include:
```javascript
{
    entireBuild: true, // indicates that it's going to be a full build
    hasError: true, // indicates that previous build had an error
    changedModules: [{ moduleName: 'app.js', event: 'change' }] // array of changed modules with names and events (vinyl: change, add, unlink)
}
```
## Unit tests bundle approach described
The idea is simple: JSPM Watch bundles every spec file *and* app files into single giant file which should be consumed by test runner. Otherwise it make take enormous amout of time to load all dependencies with karma-jspm, which makes tdd impossibly painfull. Since your project *should be* fully covered by tests, build time should be marginally the same for `watcher.start({ testsOnly: true })` and  `watcher.start()` since internally JSPM Watch uses single instance of SystemJS Builder, so cache is shared.

## Change log

### 0.3.0
- input attribute of app, is optional if jspm.main is defined in package.json
- baseURL is auto assumed "." instead of causing app build failure
- Moves JSPM Builder into separate process to avoid annoying memory leak on config change

### 0.2.0
- Changed `app.watch` to be an optional configuration param, by default JSPM Watch will use SystemJS trace to get the list of files
- Added progress bar when building entire app
- Unit tests import file is no longer generated and removed upon each file update, it's being written only when needed and removed only on process exit 

### 0.1.9
- JSPM is a peerDependency now, no need to pass it as a constructor option
- Fixed JSPM config changes handling
  
### 0.1.0
- Initial commit

## License
MIT