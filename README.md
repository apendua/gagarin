![gagarin](https://s3.amazonaws.com/gagarinjs/assets/gagarinLogo.svg)

# What it's all about? [![Circle CI](https://circleci.com/gh/anticoders/gagarin/tree/develop.svg?style=svg)](https://circleci.com/gh/anticoders/gagarin/tree/develop)

Gagarin is a testing framework designed to be used with [Meteor](https://www.meteor.com/). It can spawn multiple instances of your meteor application and run the tests by executing commands on both server and client in realtime. In other words, Gagarin allows you to automate everything you can do with `meteor shell`, browser console and a lot of free time. There's no magic. It just works.

Gagarin is also useful when you need more refined control over the meteor processes and test fancy things, e.g. the behavior of your app on server restarts or when you have multiple app instances writing to the same database. To our knowledge, this is currently not achievable with [Velocity](http://velocity.meteor.com/) - the official Meteor testing framework.

## Quick start

First, install the cli-tool with `npm`:
```
npm install -g gagarin
```
Put your tests in `tests/gagarin/` directory, e.g.
```javascript
// tests/gagarin/myFirstTestSuite.js

describe('My first Gagarin test suite', function () {
  var server = meteor();
  it('should just work', function () {
    return server.execute(function () { console.log('I am alive!'); });
  });
});
```
Finally, in your project root directory run:
```
gagarin --verbose
```
We recommend running tests in `verbose` mode, at least before `1.0.0`.
Try `gagarin --help` if you need more options.

## Important notes

Gagarin is still under heavy development and a new release is published almost
[every week](https://github.com/anticoders/gagarin/releases). Some parts of the API change over time and we can't guarantee backward compatibility before we reach `1.0.0`. Minor version changes may contain breaking changes. Though, we put a lot of effort to reduce the risk of breaking old tests. Gagarin has it's own test suite with more than
[250 test cases](https://github.com/anticoders/gagarin/tree/develop/tests/specs),
which BTW are very good source of examples.

Gagarin should play nicely with node `0.10.x` and `0.12.x`. On the other hand, there are known compatibility issues with `0.11.x` so we don't recommend using that particular version. It only applies to the cli-tool though. Your meteor application will always be run with the `node` from the development bundle corresponding to your current meteor release. Please keep this in mind if you are using any kind of continuos integration system, because it basically means that the appropriate version of meteor dev-tools will need to be downloaded before the tests can be run.

Since version `0.4.0` the `server` object created by `meteor()` helper no longer has the `location` property. To make sure the `browser` starts on the proper location, you need to pass `server` as the first argument, so
```javascript
var server = meteor();
var client = browser(server); // before 0.4.0 you would use server.location here
```

## How is it different from Velocity?

Gagarin is external to meteor. It only takes care of spawning your meteor processes and allows you to execute chunks of source code in your app environment from within your test suite and that's it. On the other hand, Velocity will deeply integrate with your app by making your test cases an integral part of your app source code, but only in a special type of builds called mirrors. This is very clever because your tests will run as fast as they possibly can. The only drawback of using velocity is that you don't have a satisfactory control over your meteor processes. In most situations this is acceptable but there are some very specific scenarios when this is not sufficient. In those cases Gagarin is probably a good choice. Gagarin tests will run a little bit slower because the source code is send to your app through DDP, but in most situations in which you would need Gagarin, this is totally acceptable because the bottleneck of your tests speed is usually somewhere else. Another advantage of using DDP is that the tests can be potentially executed on a deployed application which may be useful in some specific cases.

## How is it different from Laika?

The truth is Gagarin originates from [Laika](http://arunoda.github.io/laika/). In some sense, one may think of it as Laika 2.0, though it's not backward compatible. The main advantages of using Gagarin rather then Laika are the following:
- it does not depend on `phantomjs`
- it does not depend on injected source code, so the test runner does not have to rebuild your app each time you run the tests
- the communication with client is done through a real web driver API, which means that your tests can visit any web page and are not bound to your app's routes
- it does not depend on external mongo processes; the tests runner is clever enough to find mongo executable within your meteor development bundle

# Step-by-step guide

Gagarin is a simple test runner built on top of [mocha](http://mochajs.org/). In it's essence is very similar to [laika](https://github.com/arunoda/laika), though it's much more flexible, up-to-date and compatible with the latest Meteor versions. Currently it's implemented as a custom `mocha` interface, which simply extends the standard `bdd` ui. This may change in the future if there's a a demand to support other testing frameworks.

## Installation

Gagarin consists of two parts: `gagarin` npm module and `anti:gagarin` meteor package.
The first one should be installed globally on your system, while the second one should be
added to your meteor application. In order to work properly, the versions of the two guys
must coincide.

### The minimal setup

Please start by installing the cli tool:
```
npm install -g gagarin
```
If you try to run `gagarin` command in your meteor project directory you should receive an error message telling that there are no tests to run. Let's fix it by creating a dummy test in `tests/gagarin/` directory.
```javascript
// tests/gagarin/dummy.js
describe('A dummy test suite', function () {
  it('should do nothing', function () {});
});
```
This time, everything should work fine and your test should pass. Please note that prior to running
the tests scenarios Gagarin builds your application as well. Should the build fail
you will be notified accordingly. In case of problems, it's always good to try
`gagarin --verbose` mode for better insight.

Gagarin will always look for your test scenarios in `tests/gagarin/` directory. To alter this behavior pass a custom path as the first parameter, e.g.
```
gagarin path/to/my/tests
```
For more details see `gagarin --help`.

### What about the `anti:gagarin` package?

If you forgot to add it manually,
the `gagarin` cli-tool will make sure to add the right version to your project.
If the dummy test passed you should notice that indeed the `anit:gagarin` package
is listed in `.meteor/packages` file.

The role of the smart package is adding some backdoor functionality, similar to `meteor shell`, for testing purposes. But don't worry - it's only activate when `GAGARIN_SETTINGS` environment variable is present. For safety, double check it's not there in your production environment.

## Writing simple tests

The simplest possible test suite may look like this:
```javascript
describe('Example test suite', function () {
  var server = meteor();
  it('execute should work', function () {
    // return a promise
    return server.execute(function () {
      expect(Meteor.release).not.to.be.empty;
    });
  });
});
```
In the above example `meteor` is a global function provided by the framework, which you can use to spawn new meteor instances.For your convenience we've also exposed `expect` from the good old [chai](http://chaijs.com/).

Please note that the function passed to the `server.execute` routine is the only part that
is executed within the server environment. The rest of the code is totally external to your application.
This technique has a lot of advantages but it also have one major drawback. The functions which are passed
to the server do not share their scope with the other ones. In particular, the following code
```javascript
a = 0;
it('should print the value of a', function () {
  return server.execute(function () {
    console.log(a);
  });
});
```
will throw an "undefined variable" error. Instead you should pass `a` as an argument
```javascript
return server.execute(function (a) {
  console.log(a);
}, [ a ]);
```

## Testing with browser

Gagarin makes it really easy to coordinate tests for client and server. For example
```javascript
describe('You can also use browser in your tests', function () {
  var server = meteor();
  var client = browser(server);

  it('should work for both client and server', function () {
    return client.execute(function () {
      // some code to execute
    }).then(function () {
      return server.execute(function () {
        // some code to execute on server
      });
    });
  });
});
```
This idea originated from Laika, but we decided to go for a more promise-oriented API. Basically speaking, you can use the `browser` function to spawn as many clients as you want. The only requirement is that you have a webdriver running somewhere. By default, gagarin will try to find webdriver at port `9515` (chromedriver default). You can customize the webdriver url by providing the corresponding option for the cli tool:
```
gagarin --webdriver http://localhost:9515
```
If you're testing locally, we recommend using **chromedriver** which can be downloaded [from here](http://chromedriver.storage.googleapis.com/index.html). After unpacking the executable the only thing you need to do is to run it in the background. By default the process will listen on port `9515`. This behavior can be altered by specifying the port explicitly
```
./chromedriver --port=1234
```
Other webdrivers can be used as well. If you plan to use [phantomjs](http://phantomjs.org/) and **GhostDriver** please note that due to [a BUG in GhostDriver](https://github.com/detro/ghostdriver/issues/90) all browser sessions will share the same cookie jar, which may be problematic in test scenarios when multiple concurrent users need to be created.

A part of webdriver API is exposed and ready to use within you client promise chain. For example:
```javascript
it('should be able to use webdriver methods', function () {
  return client
    .title()
    .then(function (title) {
      expect(title).to.contain("meteor");
    });
});
```
The full list of supported methods is:
```javascript
[
  'newWindow',
  'close',
  'quit',
  'status',
  'get',
  'refresh',
  'maximize',
  'getWindowSize',
  'setWindowSize',
  'forward',
  'back',
  'takeScreenshot',
  'saveScreenshot',
  'title',
  'allCookies',
  'setCookie',
  'deleteAllCookies',
  'deleteCookie',
]
```
Additionally we've implemented a bunch of useful [helpers](https://github.com/anticoders/gagarin/blob/develop/lib/browser/helpers.js),
which you can use to simplify your tests.

## Testing with Selenium WebDriver

We recommend using selenium `2.45.0` along with Firefox 36 or 34.
Please note that webdriver is broken in Firefox 35, so don't even try to use that one.
Also keep in mind that `selenium` is usually much slower than `chromedriver` so consider
using larger timeouts values when you switch to Firefox.

Lets assume that you have a copy of `selenium-server-standalone-*.jar` available at `/path/to/selenium.jar`. First start a selenium "hub" with the following command:
```
java -jar /path/to/selenium.jar
```
Selenium server should be listening on port `4444` by default.
Then run your Gagarin tests specyfing `--webdriver` option
```
gagarin --webdriver http://localhost:4444/wd/hub
```
Please note the `/wd/hub` suffix! If you only try to connect at port `4444` the webdriver will not respond.
We've been testing Gagarin with `chrome` (38) and `firefox` (34, 36).
At this moment we cannot guarantee it will work with other browsers.

# Caveats

There are a few issues which we are aware of
and we are working hard to minimize their impact on the user experience.

## Mongo throwing 100 error

Sometimes, when you interrupt your tests with `CTRL-C` the test runner will fail to clean-up
the mongo process running in the background. If you see this error, first try to find the process
and kill it. If it does not help go to `.gagarin/local/db` and delete the `mongo.lock` file.

## Tests fail due to timeouts

Depending on the environment and the system performance,
the tests may require timeout values different from the default ones.
Please keep this in mind and see `gagarin --help` for available customization options.
In case of problems, it's generally good to run tests in `--verbose` mode
to have a better picture of what's going on.

## `before all` timeout

If you run your tests in `--verbose` mode you would notice that this is happening
because sometimes webdriver fails to respond within a reasonable time window
when we create a new browser session. This may also depend on your system resources.
It's totally external to Gagarin, so the only thing we can do is to report
this error properly.

# Examples

Since we don't have a comprehensive documentation yet, please consider the following set of simple examples as a current API reference.

## Scope of the a local variable

It's good to keep in mind that the code which is intended to be executed on either server or client is passed as a string. Of course it does not have an immediate access to you local variable scope. In particular, things like:
```javascript
var a = 1;
it("should be able to access local variable", function () {
  return client.execute(function () {
    return a + 1;
  });
});
```
will throw "a is undefined". Trying to set `a = 1;` will throw as well because the code is implicitly executed in `strict mode`, which does not allow introducing new variables to the global scope.

## Passing arguments to client and server code

If you don't need to modify the variables within your "remote" code then probably the easiest way to overcome the problem described above is to pass local scope variables as arguments:

```javascript
var a = 1;
it("should be able to access local variable", function () {
  return client.execute(function (a) {
    return a + 1;
  }, [ a ]); // array of arguments
});
```
Note that this construction will already allow you to do anything you want with your local variables, because you can always update them within `then`, after your client/server computation is done. However, it's not very convenient in more complicated scenarios.

## Copying closure

To simplify the interaction between client and server code, we've added an affordance to reuse the declared closure in all three environments: test scope, server and client. To this end, you need to explicitly provide a list of variables to be synced as well as an accessor function:

```javascript
var a = 1, b = 2, c = 0;
// this is a hack :)
closure(['a', 'b', 'c'], function (key, value) {
  return eval(key + (arguments.length > 1 ? '=' + JSON.stringify(value) : ''));
});
```
Now this code should work without problems:
```javascript
it("should be able to access local variables", function () {
  return client.execute(function (a) {
    c = a + b;
  }).then(function () {
    expect(c).to.equal(3);
  });
});
```
The only reserved variable name for closures is `$`, which you probably would not like to use for other reasons.

## Asynchronous test cases

On both server and client you can also use asynchronous scripts:
```javascript
it("should be able to do work asynchronously", function () {
  return server.promise(function (resolve) {
    setTimeout(function () {
     resolve(1234);
    }, 1000);
  }).then(function (value) {
    expect(value).to.equal(1234);
  });
});
```
The second argument to the `promise` is `reject`, so if you want to throw asynchronously:
```javascript
it("should be able throw asynchronously", function () {
  return server.promise(function (resolve, reject) {
    setTimeout(function () {
     reject(new Error("this is some fake error"));
    }, 1000);
  }).expectError(function (err) {
    expect(err.message).to.contain("fake error");
  });
});
```
If you want to pass additional variables to `promise` method, do it like this:
```javascript
it("should be able to pass arguments", function () {
  return server.promise(function (resolve, reject, arg1, arg2) {
    setTimeout(function () {
     resolve(arg1 + arg2);
    }, 1000);
  }, [ 1, 2 ]).then(function (value) {
    expect(value).to.equal(3);
  });
});
```

## Waiting for conditions

There's also a useful helper to wait for conditions. Again, it works on both server and client:
```javascript
before(function () {
  return client.execute(function () {
    Items.insert({_id: 'someFakeId'});
  });
});

it("should be able to wait on server", function () {
  return server.wait(1000, 'until data is propagated to the server', function () {
    return Items.findOne({_id: 'someFakeId'});
  }).then(function (value) {
    expect(value).to.be.ok;
    expect(value._id).to.equal('someFakeId');
  });
});
```

# For contributors

To test the package locally make sure that a webdriver is listening on port `9515`,
then simply run the tests with the following command
```
npm test
```
or just
```
./test.js [options]
```
in the project root directory. Additionally you can use
```
./test.js --help
```
to display information about all possible options.
For example, to use a different webdriver location, you can specify it with
```
./test.js --webdriver http://localhost:4444/wd/hub
```
For testing purposes it's sometimes useful to intall a version of the npm package from specyfic branch.
```
npm install -g anticoders/gagarin#develop
```

## License

MIT licensed

Copyright (C) 2015 Tomasz Lenarcik, http://apendua.com
