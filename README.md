# Node Test Helper

This test-helper module makes the node unit test framework from the Node-RED core available for node contributors.

Using the test-helper, your tests can start the Node-RED runtime, load a test flow, and receive messages to ensure your node code is correct.

## Adding to your node project dependencies

To add unit tests your node project test dependencies, add this test helper as follows:

    npm install node-red-node-test-helper --save-dev

This will add the helper module to your `package.json` file as a development dependency:

```json
...
  "devDependencies": {
    "node-red-node-test-helper": "^0.1.6"
  }
...
```

The test-helper requires the node-red runtime to run its tests, but Node-RED is **not** installed as a dependency.  The reason for this is that test-helper is (or will be) used in Node-RED core tests, and Node-RED itself has a large number of dependencies that you may not want to download if you already have it installed.

You can install the Node-RED runtime available for your unit tests one of two ways:

1. as a dev dependency in your project:

```
npm install node-red --save-dev
```

2. or link to Node-RED installed globally (recommended) using:

```
npm install -g node-red
npm link node-red
```

Both [Mocha](https://mochajs.org/) and [Should](https://shouldjs.github.io/) will be pulled in with the test helper.  Mocha is a unit test framework for Javascript; Should is an assertion library.  For more information on these frameworks, see their associated documentation.

## Linking to additional test dependencies

To reduce disk use further, you can install the test-helper and additional dev dependencies globally and then link them to your node project.  This may be a better option especially if you are developing more than one node.

See the `package.json` file for the additional dependencies used by test-helper.

For example to install express globally:

    npm install -g express

Then link to it in your project:

    npm link express

Depending on the nodes in your test flow, you may also want to link to other global packages.  If a test indicates that a package cannot be found, and you expect to need it for testing other nodes, consider installing the package globally and then linking it to your node project the same way.

## Adding test script to `package.json`

To run your tests you can add a test script to your `package.json` file in the `scripts` section.  To run all of the files with the `_spec.js` prefix in the test directory for example:

```json
  ...
  "scripts": {
    "test": "mocha \"test/**/*_spec.js\""
  },
  ...
```

This will allow you to use `npm test` on the command line.

## Creating unit tests

We recommend putting unit test scripts in the `test/` folder of your project and using the `*_spec.js` (for specification) suffix naming convention.

## Example unit test

Here is an example test for testing the lower-case node in the [Node-RED documentation](https://nodered.org/docs/creating-nodes/first-node).  Here we name our test script `test/lower-case_spec.js`.

### `test/lower-case_spec.js`:

```javascript
var should = require("should");
var helper = require("node-red-node-test-helper");
var lowerNode = require("../lower-case.js");

describe('lower-case Node', function () {

  afterEach(function () {
    helper.unload();
  });

  it('should be loaded', function (done) {
    var flow = [{ id: "n1", type: "lower-case", name: "lower-case" }];
    helper.load(lowerNode, flow, function () {
      var n1 = helper.getNode("n1");
      n1.should.have.property('name', 'lower-case');
      done();
    });
  });

  it('should make payload lower case', function (done) {
    var flow = [
      { id: "n1", type: "lower-case", name: "lower-case",wires:[["n2"]] },
      { id: "n2", type: "helper" }
    ];
    helper.load(lowerNode, flow, function () {
      var n2 = helper.getNode("n2");
      var n1 = helper.getNode("n1");
      n2.on("input", function (msg) {
        msg.should.have.property('payload', 'uppercase');
        done();
      });
      n1.receive({ payload: "UpperCase" });
    });
  });
});
```

In this example, we require `should` for assertions, this helper module, as well as the `lower-case` node we want to test, located in the parent directory.

We then have a set of mocha unit tests.  These tests check that the node loads correctly, and ensures it makes the payload string lower case as expected.

## Getting nodes in the runtime

The asynchronous `helper.load()` method calls the supplied callback function once the Node-RED server and runtime is ready.  We can then call the `helper.getNode(id)` method to get a reference to nodes in the runtime.  For more information on these methods see the API section below.

## Receiving messages from nodes

The second test uses a `helper` node in the runtime connected to the output of our `lower-case` node under test.  The `helper` node is a mock node with no functionality. By adding "input" event handlers as in the example, we can check the messages received by the `helper`.

To send a message into the `lower-case` node `n1` under test we call `n1.receive({ payload: "UpperCase" })` on that node.  We can then check that the payload is indeed lower case in the `helper` node input event handler.

## Running your tests

To run your tests:

    npm test

Producing the following output (for this example):

    > red-contrib-lower-case@0.1.0 test /dev/work/node-red-contrib-lower-case
    > mocha "test/**/*_spec.js"

    lower-case Node
      ✓ should be loaded
      ✓ should make payload lower case

    2 passing (50ms)

## Creating test flows with the editor

To create a test flow with the Node-RED editor, export the test flow to the clipboard, and then paste the flow into your unit test code.  One helpful technique to include `helper` nodes in this way is to use a `debug` node as a placeholder for a `helper` node, and then search and replace `"type":"debug"` with  `"type":"helper"` where needed.

## Using `catch` and `status` nodes in test flows

To use `catch` and `status` or other nodes that depend on special handling in the runtime in your test flows, you will often need to add a `tab` to identify the flow, and associated `z` properties to your nodes to associate the nodes with the flow.  For example:

```javascript
var flow = [{id:"f1", type:"tab", label:"Test flow"},
  { id: "n1", z:"f1", type: "lower-case", name: "test name",wires:[["n2"]] },
  { id: "n2", z:"f1", type: "helper" }
```

## Additional examples

For additional test examples taken from the Node-RED core, see the `.js` files supplied in the `test/examples` folder and the associated test code at `test/nodes` in [the Node-RED repository](https://github.com/node-red/node-red/tree/master/test/nodes).

## API

> *Work in progress.*

### load(testNode, testFlows, testCredentials, cb)

Loads a flow then starts the flow. This function has the following arguments:

* testNode: (object|array of objects) Module object of a node to be tested returned by require function. This node will be registered, and can be used in testFlows.
* testFlow: (array of objects) Flow data to test a node. If you want to use flow data exported from Node-RED editor, export the flow to the clipboard and paste the content into your test scripts.
* testCredentials: (object) Optional node credentials.
* cb: (function) Function to call back when testFlows has been started.

### unload()

Return promise to stop all flows, clean up test runtime.

### getNode(id)

Returns a node instance by id in the testFlow. Any node that is defined in testFlows can be retrieved, including any helper node added to the flow.

### clearFlows()

Stop all flows.

### request()

Create http ([supertest](https://www.npmjs.com/package/supertest)) request to the editor/admin url.

Example:

```javascript
helper.request().post('/inject/invalid').expect(404).end(done);
```

### startServer(done)

Starts a Node-RED server for testing nodes that depend on http or web sockets endpoints like the debug node.
To start a Node-RED server before all test cases:

```javascript
before(function(done) {
    helper.startServer(done);
});
```

### stopServer(done)

Stop server.  Generally called after unload() complete.  For example, to unload a flow then stop a server after each test:

```javascript
afterEach(function(done) {
    helper.unload().then(function() {
        helper.stopServer(done);
    });
});
```

### url()

Return the URL of the helper server including the ephemeral port used when starting the server.

### log()

Return a spy on the logs to look for events from the node under test.  For example:

```javascript
var logEvents = helper.log().args.filter(function(evt {
    return evt[0].type == "batch";
});
```

## Running helper examples

    npm run examples

This runs tests on an included lower-case node (as above) as well as snaphots of some of the core nodes' Javascript files to ensure the helper is working as expected.