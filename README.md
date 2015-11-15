# FiniteJS
Write and run composable, stateless integration test plans in JavaScript.

FiniteJS is...

## Composable
* Pick the minimum test plans you need to run and the order to run them.
* FiniteJS helps you minimize and obey functional dependencies across test plans.

## Stateless
* Shared state is limited to explicitly defined output parameters.
* FiniteJS passes shared parameter dependencies into downstream test plans automatically, as needed.
* All test plans that are run in parallel are guaranteed to have their own parameter instances (including browsers).
* Only test plans that are run in series can share and mutate output parameters (such as a browser) among each other.
* FiniteJS manages shared reporting state across all test plans, so that multiple test plans can roll up their results into the same report.

## Fast to Run
* Given enough hardware, no integration test will take longer to complete the single slowest serial test plan sequence.

## Fast to Create
* Declarative syntax makes it quick to arrange complex test plans together.
* Reduces the need to daisy chain Promises by hand, resulting in increased code readability. There's also a reduced chance for intermittent timeout issues to appear, all because you forgot to wait for a Promise to resolve.

## Extensible
* Low-touch approach integrates well with testing libraries like Prototype to leverage the power of Selenium WebDriver for browser automation.

## Debuggable
* When a test fails, it is easy to see the minimum steps required to reproduce the failure.

## Robust Failure Handling
* When a test fails, dependent tests won't run. For example, if the app is down, the test plan will fail fast.

## Slick Reporting
* Beautiful test reports through the Finite-Reports add-on module.


# Getting Started

`npm install finite`

example.js
```
var Website.login = function(user, pw) {
  console.log("Logged in as " + user);
}

var loginTest = {
  name: "Login Test Plan",
  do: [
    {
      call: Website.login,
      in: {user: "Joe", pw: "123"}
    }
  ]
}

require('finite').run(plan);
```

# Documentation

## Core

FiniteJS exposes the following core library methods:

* `run(plan)`: Run a test plan.
* `validate(plan)`: Before running a test plan, validate that the plan is acyclic and each test has its dependencies met.
* `fail()`: Update the current test plan result as Failed.
* `report(metaData)`: Update the current test plan result with additional meta data about the actual result.

## Callable

A **callable** is an individual test case within a test plan. A callable specifies the function that FiniteJS will invoke to actually interact with the application under test.

A callable is a hash containing the following:
* `call`: Specify a function to invoke. TODO describe the specific details of how the function should operate, i.e. the names of the function parameters are a "published" API of sorts.
* `out`: Specify an array of strings indicating all the parameters the callable will return that are intended for downstream test plans to use. TODO describe how multi-dispatch is supported.

## Test Plan

A **test plan** describes what to test. Every test plan is a hash that takes *one* of the following forms: do, call, or doParallel.

### do
* `name`: Optional. Specify a human-readable string name for the test plan.
* `do`: Specify an array of test plans to execute in series.

### call
* `name`: Optional. Specify a human-readable string name for the test plan.
* `call`: Specify a ``callable`` to invoke. FiniteJS will automatically pass all found dependendent parameters to the ``callable`` at runtime.
* `in`: Specify a hash of input parameter key:value pairs to pass to the callable function. Parameter names must match the argument names defined in the callable function declaration.

### doParallel
* `name`: Optional. Specify a human-readable string name for the test plan.
* `doParallel`: Specify an array of test plans to execute in parallel. Forks new parallel execution branches for each plan in the array.
* `thenDo`: Specify an array of test plans to execute in series after all `doParallel` plans have been run. All `thenDo` plans will be run across each forked parallel execution branches that `doParallel` spawned.
* `thenDoParallel`: Specify an array of test plans to execute in parallel after all `doParallel` plans have been run. All `thenDoParallel` plans will be run across each forked parallel execution branches that `doParallel` spawned.



# Examples

## Simple Example

TODO

## Advanced Example

The following example performs a complicated sequence of test plans, some in series, and some in parallel.

### Callables

```
// browser.js
export function() {
  launch = {
    out: "browser",
    call: function(browserType) {
      console.log("Launching " + browserType);
      browser = {}; // REPLACEME with Prototype browser, for example.
      return browser;
    }
  }
}

// account.js
export function() {
  login = {
    out: "account",
    call: function(browser, user, pw) {
      console.log("Logging in as " + user);
      // actually login...
      account = {user: user, pw: pw};
      return account;
    }
  }
}

// document.js
export function() {
  open = {
    out: ["editor"],
    call: function(browser, account, doc_id, finite) {
      console.log("Opening document " + doc_id);
      finite.report({screenshot: 'actual.png'});
      editor = {};
      return editor;
    }
  }
}

// imageEditor.js
export function() {
  testResize = {
    call: function(browser, editor, imageEl, finite) {
      console.log("Verifying that resize image works.");
      finite.fail("Could not resize image.")
    }
  }
  testUpload = {
    call: function(browser, editor, imageEl) {
      console.log("Testing image upload...")
    }
  }
}
```

### Plans

allPlans.js
```
var Finite = require('finite'),
  Browser = require('browser'),
  Account = require('account'),
  Document = require('document'),
  ImageEditor = require('imageEditor');

var verifyEditorPlan = {
  name: "Verify Editor Functionality",
  doParallel: [
    {
      call: ImageEditor.testResize,
      in: {imageEl: 'any'}
    },
    {
      call: ImageEditor.testChange,
      in: {imageEl: 'any'}
    }
  ]
}

var oldDocumentTestPlan = {
  name: "Test an Old Document"",
  do: [
    {
      call: Document.open,
      in: { doc_id: "doc_98765" },
      expected: {
        screenshot: "screen1.png"
      }
    },
    {
      verifyEditorPlan
    }
  ]
}

var recentDocumentTestPlan = {
  name: "Test a Recent Document",
  do: [
    {
      call: Document.open,
      in: { doc_id: "doc_98765" },
      expected: {
        screenshot: "screen1.png"
      }
    },
    {
      verifyEditorPlan
    }
  ]
}

var kitchenSinkTestPlan = {
  name: "Kitchen Sink Test",
  doParallel: [
    {
      name: "Firefox",
      call: Browser.launch,
      in: { browserType: "Firefox" }
    },
    {
      name: "Chrome",
      call: Browser.launch,
      in: { browserType: "Chrome" }
    }
  ],
  thenDo: [
    {
      name: "Login",
      call: Login.login,
      in: { user: "joe", pw: "123" }
    },
    {
      name: "Test All Documents",
      doParallel: [
        oldDocumentTestPlan,
        recentDocumentTestPlan
      ]
    }
  ]
}
```

Finally, run the plan:
```
Finite.run(kitchenSinkTestPlan);
```

Use Finite-Reporter to display results.
