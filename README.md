# sol-unit

Unit tests for Solidity contracts.

**Disclaimer: Running and using this library is difficult. It is alpha software, and it uses an alpha client to test code written in a language (Solidity) that is still under development. The library should be considered experimental.**

## Table of Content

1. [Introduction](#introduction)
2. [Installation and Usage](#installation-and-usage)
3. [Writing tests](#writing-tests)
4. [Examples](#examples)
5. [Testing](#testing)
6. [Program structure](#program-structure)

## Introduction

solUnit is a unit testing framework for Solidity contracts. It runs unit tests against [javascript Ethereum](https://github.com/ethereumjs).

The tests are run in a controlled environment, using extremely high gas-limit and a standard account with an extreme amount of Ether. More blockchain/VM settings will become available with time, along with extra analysis tools.

**NOTE**: Static coverage analysis (parsing syntax trees) was been removed in `0.3` to be replaced with runtime analysis at some later date.

## Installation and Usage

Officially supported on 64 bit Ubuntu (14.04, 14.10, 15.04).

`npm install sol-unit`

Use the `--global` flag to install the `solunit` command-line version.

The executable name is `solunit`. If you install it globally you should get it on your path. Try `$ solunit -V` and it should print out the version.

To see all the options:

```
$ solunit --help

  Usage: solunit [options] <test ...>

  Options:

    -h, --help           output usage information
    -V, --version        output the version number
    -m, --debugMessages  Print debugging information. [default=false]
    -d, --dir <dir>      Directory in which to do the tests. [default=current directory]
```

Example: `$ solunit ArrayTest BasicTypesTest` will look for `.bin` and `.abi` files for those test-contracts in the current directory. Simply typing `solunit` will run all tests found in the current directory. Test contracts must always end in `Test`.

The library can be run from javascript as well:

```
var sUnit = require('sol-unit');

// ...

sUnit.runTests(['ArrayTest', 'BasicTypesTest'], 'dirWithTestsInThem', true, function(error, stats){
    console.log(error);
    console.log(stats);
});
```

* First param is an array of strings of the test names.
* Second param is the directory in which the tests `.bin` and `.abi` files can be found.
* Third is whether or not to print test data (looks similar to mocha).
* Fourth is an error first callback that returns test statistics.

The stats object contains a `testUnits` field which has data of all the test units in them, along with a `total` and `successful` field which is the total number of tests and the number of tests that succeeded, respectively. Note that each unit is a contract with a number of different tests in them (the test-functions).

```
{
    testUnits: {
        ArrayTest: TestResults,
        BasicTypesTest: TestResults,
        ...
    }
    successful: 5,
    total: 7
}
```

`TestResults` is an object with all the test functions of the test contract in it, mapped to a simple struct containing:

1. The name of the test.
2. The result (true if all assertions in the function were true, otherwise false).
3. The messages of any failed assertions.
4. Error messages (currently not used).

```
TestResults = {
    testPop: { name: 'testPop', result: false, messages: ["Array length not 1"], errors: [] },
    testPush: { name: 'testPush', result: true, messages: [], errors: [] },
    ...
}
```

Event-listeners are documented in the library structure section near the bottom of this document.

## Writing tests

The easiest way to start writing unit-testing contracts is to look at the examples in `contracts/src`.

### Test format

The rules for a unit testing contract right now (0.3.x) are these:

1. Test-contracts must use the same name as the test target but end with `Test`. If you want to test a contract named `Arrays`, name the test-contract `ArraysTest`. If you want to test a contract named `Coin`, name the test-contract `CoinTest`, and so on.

2. Test function names must start with `test`, e.g. `function testAddTwoInts()`, and they must be public. There is no limit on the number of test-functions that can be in each test-contract.

3. The test-contract may use a test event: `event TestEvent(bool indexed result, string message);`. The recommended way of doing unit tests is to have the test-contract extend `Asserter`, by importing `Asserter.sol` (comes with the library). Its assertion methods has a proper test event already set up which will fire automatically when an assertion is made. There is no limit on the number of assertions in a test method.

NOTE: If none of the existing assertions would fit, extend the Asserter.sol contract or calculate the result in the test-function and use `assertTrue` or `assertFalse`.

- A test **passes** if no assertion fails (i.e. `result` is `true` in every test event). If no assertions are made, the test will pass automatically. If at least one assertion fails, the test will **not pass**.

**Example of a simple storage value test**


```
import "./assertions/Asserter.sol";

contract Demo {
    
    uint _x = 0;
    
    function setX(uint x){
        _x = x;
    }
    
    function getX() constant returns (uint x){
        return _x;
    }
}

// Test contract named DemoTest as per (1).
contract DemoTest is Asserter {

    uint constant TEST_VAL = 55;
    
    // Test method starts with 'test' and will thus be recognized (2).
    function testSetX(){
        Demo demo = new Demo();
        assertAddressNotZero(address(demo), "Failed to deploy demo contract");
        demo.setX(TEST_VAL);
        var x = demo.getX();
        // Use assert method from Asserter contract (3). It will automatically fire off a test event.
        assertUintsEqual(x, TEST_VAL, "Values are not equal");
    }
    
}
```

There is plenty more in the `contracts/src` folder. All test contracts looks pretty much the same - a target contract and a few methods that begin with 'test' and has assertions in them.

### Build constraints

The `.bin` and `.abi` files for the test contracts must be available in the working directory.

*Example*

I want to unit test a contract named `Bank`. I create `BankTest`, compile the sources, and make sure the following files are created: `BankTest.bin`, `BankTest.abi`.

To make sure I get all of this, I compile with: `solc --bin --abi -o . BankTest.sol`. They will end up in the same folder as the sources in this case.

### TestEvent and assertions

`TestEvent` is what the framework listens too. It will run test functions and then listen to `TestEvent` data from the contract in question.

`event TestEvent(bool indexed result, string message);`

The `result` param is true if the assertion succeeded, otherwise it's false.

The `message` param is used to log a message. This is normally used if the assertion fails.

If the contract extends `Asserter`, it will inherit `TestEvent`, and can also use the assert methods which will automatically trigger the test event. Otherwise you can just roll your own. 

**Example of a bare-bones test contract**

```
contract Something {
    
    int _something;
    
    function setSomething(int something){
        _something = something;
    }
    
    function something() constant returns (int something){
        return _something;
    }
}

contract SomethingTest {

    event TestEvent(bool indexed result, string message);
    
    function testSomething(){
        Something testee = new Something();
        int someValue = 5; 
        testee.setSomething(someValue);
        var intOrSomeSuch = testee.something();
        var result = (intOrSomeSuch == someValue);
        if(result){
            TestEvent(true, "");
        } else {
            TestEvent(false, "Something is wrong.");
        }
    }

}
```

## Examples

The contracts folder comes with a number of different examples.

## Testing

`mocha` or `npm test`.

It is also possible to run the executable from the `./contracts/build` folder, either `solunit` if it is installed globally, or `../../../bin/solunit.js`. NOTE: One test always fails, just to show how it looks.

## Program structure

The framework uses Solidity events (log events) to do the tests. It uses the afore-mentioned Solidity `TestEvent` to get confirmation from contract test-methods.

The `SUnit` class is where everything is coordinated. It deploys the test contracts and publish the results via events. It implements nodes EventEmitter. It forwards some of these events from dependencies like the `TestRunner`, which is also an emitter.

The `TestRunner` takes a test contract, finds all test methods in it and run those. It also sets up a listener for solidity events and uses those events to determine if a test was successful or not. It is normally instantiated and managed by a `SUnit` object.

![sol_unit_diag.png](https://github.com/androlo/sol-unit/blob/master/resources/docs/sol_unit_diag.png "SolUnit structure")

In the diagram, the executable gathers the tests and set things up, then `SUnit` deal with the contract execution part.

Presentation of test data is not part of the diagram. The core `SUnit` library does not present - it only passes data through events. The way presentation works in the command-line tool is it listens to all SUnit events and prints the reported data using a special test-logger.

#### Events

`SUnit` extends Node.js' `EventEmitter`. 

The events you may listen to are these:

##### Suite started

id: `'suiteStarted'` - The suite has started. 
 
callback params: `error`, `tests` - array of test names. Keep in mind these may not be the same as you entered, because some may not have existed or you could have skipped the names so that it found all tests in the directory. This is the list of tests (test contracts) that was actually found.

##### Suite done
 
id: `'suiteDone'` - The suite is done. 
 
callback params: `stats` - collection of stats for all test contracts.
 
##### Contracts started
 
id: `'contractStarted'` - A new test contract is starting. 

callback params: `error`, `testName` - the test name.
 
##### Contracts done

id: `'contractDone'` - A contract is done. 

callback params: `error`

##### Methods started

id: `'methodsStarted'` - The test methods in the current contract are about to be run.

callback params: `methodNames` - A list of all the test functions that was discovered in the current contract, and will be run.

##### Methods done

id: `'methodsDone'` - The methods are done. 

callback params: `error`, `results` - the combined test-results.
 
##### Method started

id: `'methodStarted'` - A test method in the current contract are being run.

callback params: `methodName` - The name of the method.

##### Method done

id: `'methodDone'` - The method is done and the results are in. 

callback params: `results` - the test results. 