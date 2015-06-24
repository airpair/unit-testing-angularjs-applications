Unit testing, as the name implies, is about testing individual units of code.

This post tries to show a few patterns and guidelines to help us with unit testing Angular applications after setting some minimum configuration.

It's aimed to people who is already familiar with Angular but haven't gone deep into unit testing their applications. Although it should be helpful too for completely beginners.

## Setting up Angular unit tests

Before we start testing we need to install and configure several dependencies. For this we will be using the package manager [npm](https://www.npmjs.com/), although [Bower](http://bower.io/) can be used as well.

```
npm install angular --save
bower install angular --save
```

Note that we can skip to the [configuration files](#configuration-files) section if we don't want to go step by step.

### The ngMock module

Angular provides [ngMock](https://docs.angularjs.org/api/ngMock) to inject and mock services into unit tests. And one of the most useful parts of the `ngMock` module is `$httpBackend` which lets us mock XHR requests.

```
npm install angular-mocks --save-dev
```

### Configuring Karma with Jasmine

[Karma](http://karma-runner.github.io/) is a test runner written by the Angular team which allow us to execute tests in multiple browsers.

```
npm install karma --save-dev
```

After installing it we need to crete a configuration file running `karma init` but to extend its functionality we need to install first some plugins.

**Testing framework**

Angular can be tested using any JavaScript unit testing framework out there, but [Jasmine](http://jasmine.github.io/) is probably the most popular.

```
npm install karma-jasmine jasmine-core --save-dev
```

**Browsers**

Karma takes care of auto-capturing and killing the browsers, but we need to install at least one of these launchers.

```
npm install karma-chrome-launcher --save-dev
npm install karma-phantomjs-launcher --save-dev
npm install karma-firefox-launcher --save-dev
npm install karma-safari-launcher --save-dev
npm install karma-opera-launcher --save-dev
npm install karma-ie-launcher --save-dev
```

**Templates preprocessor**

When testing directives we need to set up Karma to serve our templates using a preprocessor to convert HTML into JS string.

[ng-html2js](https://github.com/karma-runner/karma-ng-html2js-preprocessor) creates a "templates" module and put the HTML into the `$templateCache`.

```
npm install karma-ng-html2js-preprocessor --save-dev
```

**Code coverage**

It's great to identify which parts of our code are lacking test coverage and generate reports with [Istanbul](https://github.com/gotwarlost/istanbul), that calculates the percentage of code accessed by tests.

```
npm install karma karma-coverage --save-dev
```

### Configuration files

**package.json**

If we haven't followed all the previous steps we must copy the following `package.json` and run `npm install` to have all the required dependencies.

```javascript
{
  "name": "unit-testing-angularjs-applications",
  "version": "1.0.0",
  "description": "Unit testing AngularJS applications.",
  "dependencies": {
    "angular": "^1.3.15"
  },
  "devDependencies": {
    "angular-mocks": "^1.3.15",
    "jasmine-core": "^2.3.4",
    "karma": "^0.12.32",
    "karma-chrome-launcher": "^0.1.12",
    "karma-coverage": "^0.3.1",
    "karma-jasmine": "^0.3.5",
    "karma-ng-html2js-preprocessor": "^0.1.2"
  },
  "scripts": {
    "test": "karma start"
  }
}
```

**karma.conf.js**

If we haven't generated yet a configuration file now it's the time.

```
karma init
```

After answering some questions it will create a `karma.conf.js` file which we still have to modify including the list of files to load in the browser and configuring the installed preprocessors (ng-html2js and coverage).

The final configuration file should look as follows.

```javascript
module.exports = function(config) {
  config.set({

    basePath: '',

    frameworks: ['jasmine'],

    files: [
      'node_modules/angular/angular.js',
      'node_modules/angular-mocks/angular-mocks.js',
      'src/**/*.js',
      'src/**/*.html'
    ],

    exclude: [
    ],

    preprocessors: {
      'src/**/*.html': ['ng-html2js'],
      'src/**/!(*.mock|*.spec).js': ['coverage']
    },

    ngHtml2JsPreprocessor: {
      // strip this from the file path
      stripPrefix: 'src/',
      // create a single module that contains templates from all the files
      moduleName: 'templates'
    },

    reporters: ['progress', 'coverage'],
    
    coverageReporter: {
      type : 'html',
      // output coverage reports
      dir : 'coverage/'
    },

    port: 9876,

    colors: true,

    logLevel: config.LOG_INFO,

    autoWatch: true,

    browsers: ['Chrome'],

    singleRun: false
  });
};
```

And we can run our tests with any of these two commands.

```
karma start
npm test
```

## Mocking external dependencies

To test the functionality of a piece of code in isolation we need to mock any dependency.

Angular is written with testability in mind and come with [dependency injection](https://docs.angularjs.org/guide/di) built in, what help us to achieve decoupling and therefore to test our objects in isolation.

A service (service, factory, value, constant, or provider) is the most common type of dependency in Angular applications and they are defined via providers.

The Angular [injector](https://docs.angularjs.org/api/auto/service/$injector) retrieves object instances as defined by providers.

So there are at least two ways we can mock our services:

### Using the $provide service

We can provide our own implementation in a `beforeEach` Jasmine block, creating an object with the same methods as the original service.

```javascript
var myService;
beforeEach(module(function($provide) {
  myService = {
    syncCall: function() {},
    asyncCall: function() {}
  };
  $provide.value('myService', myService);
}));
```

### Creating a reusable service provider

Or we can reuse it creating the implementation in a separate `my-service.mock.js` file using the provider syntax.

```javascript
angular.module('myServiceMock', [])
  .provider('myService', function() {
    this.$get = function() {
      return {
        syncCall: function() {},
        asyncCall: function() {}
      };
    };
  });
```

And later we load the mocked module in a `beforeEach` Jasmine block inside of our test suite.

```javascript
beforeEach(module('myServiceMock'));

var myService;
beforeEach(inject(function(_myService_) {
  myService = _myService_;
}));
```

In any case it's good to **load the mocked modules after** the module we are testing to be sure the mock overrides the actual implementation.

## Testing patterns

As code and specs are better placed side-by-side we can put all our files inside the `src` folder, using the `.spec.js` suffix to differentiate test files.

```
src/my-controller.js
src/my-controller.spec.js
src/my-directive.js
src/my-directive.spec.js
src/my-service.js
src/my-service.spec.js
```

An we can make testing simpler if we follow the same pattern to test each of our objects:

1. Describe the object with type and name.
2. Load our object's module.
3. Load mock modules as needed.
4. Inject dependencies and spy on methods.
5. Initialize the object:
  1. Services just need to get injected.
  2. Controllers are instantiated using the $controller service.
  3. We need to $compile directives.
6. Write expectations grouped in describe blocks.

Therefore all our test files will have the same structure with the common content in `beforeEach` blocks that run before every test. And nested `describe` blocks containing the self-describing expectations inside of `it` blocks.

```javascript
describe('ObjectType: objectName', function() {

  beforeEach(module('objectModule'));

  var myObject;
  beforeEach(inject(function(_myObject_) {
    myObject = _myObject_;
  }));

  describe('Method: methodName', function() {
    it('should do ...', function() {
      expect(true).toBe(true);
    });
  });
  
});
```

### Controllers

Controllers are easy to test as long as we don't manipulate the DOM and functions have a single and clear purpose.

We should test for the state, synchronous and asynchronous calls to services, and events.

**Code**

We create our basic controller using the `controller as` syntax. And we inject the dependencies on a external service and the `$scope` for using events.

```javascript
angular.module('myControllerModule', ['myServiceModule'])
  .controller('MyController', ['$scope', 'myService', MyController]);

function MyController($scope, myService) {
  var vm = this;
}
```
Then we expose a method and different types of properties to the view.

```javascript
  vm.hasError = false;
  vm.myProperty = 'My Controller';
  vm.myArray = [];
  vm.myObject = myService.syncCall();
  vm.myNumber = 0;
  vm.changeProperty = changeProperty;
  
  function changeProperty(value) {
    vm.myProperty = value;
  }
```

We call the service asynchronously setting the success and error handlers.

```javascript
  myService.asyncCall().then(
    function(data) {
      vm.myArray = data;
    },
    function() {
      vm.hasError = true;
    }
  );
```

And finally we use the `$scope' to handle events.
  
```javascript
  $scope.$emit('my-event');

  $scope.$on('some-event', function() {
    vm.myNumber++;
  });
```

Having thus this code.

```javascript
angular.module('myControllerModule', ['myServiceModule'])
  .controller('MyController', ['$scope', 'myService', MyController]);

function MyController($scope, myService) {
  var vm = this;

  vm.hasError = false;
  vm.myProperty = 'My Controller';
  vm.myArray = [];
  vm.myObject = myService.syncCall();
  vm.myNumber = 0;
  vm.changeProperty = changeProperty;

  myService.asyncCall().then(
    function(data) {
      vm.myArray = data;
    },
    function() {
      vm.hasError = true;
    }
  );

  function changeProperty(value) {
    vm.myProperty = value;
  }

  $scope.$emit('my-event');

  $scope.$on('some-event', function() {
    vm.myNumber++;
  });

}
```

**Specs**

We write the essential structure of the test suite loading the mocked service after the controller's module. 

```javascript
describe('Controller: MyController', function() {

  beforeEach(module('myControllerModule'));
  beforeEach(module('myServiceMock'));
  
});
```

For testing calls to other services we need to spy their methods. And Jasmine provides us with the [spyOn](http://jasmine.github.io/2.0/introduction.html#section-Spies) function to track calls and arguments passed.

With `callThrough()` we delegate to the actual implementation. However for the asynchronous method we need to mock its behaviour returning a promise provided by the [$q](https://docs.angularjs.org/api/ng/service/$q) service.

```javascript
  var myService;
  var deferred;
  // Mock services and spy on methods
  beforeEach(inject(function($q, _myService_) {
    deferred = $q.defer();
    myService = _myService_;
    spyOn(myService, 'syncCall').and.callThrough();
    spyOn(myService, 'asyncCall').and.returnValue(deferred.promise);
  }));
```

To keep it dry we initialize the controller passing the mocked dependencies in a different `beforeEach` block.

```javascript
  var MyController;
  var scope;
  // Initialize the controller and a mock scope.
  beforeEach(inject(function($rootScope, $controller) {
    scope = $rootScope.$new();
    spyOn(scope, '$emit');
    MyController = $controller('MyController', {
      $scope: scope,
      myService: myService
    });
  }));
```

And we start testing the state of our controller. As we use the `controller as` syntax we don't need to test for the `$scope` but directly for the **exposed properties and methods** of the controller.

```javascript
  describe('State', function() {

    it('should expose myProperty to the view', function() {
      expect(MyController.myProperty).toBeDefined();
      // We can use Angular helpers.
      expect(angular.isArray(MyController.myArray)).toBe(true);
      expect(angular.isObject(MyController.myObject)).toBe(true);
      expect(angular.isNumber(MyController.myNumber)).toBe(true);
    });

    it('should expose a method to change myProperty', function() {
      expect(MyController.changeProperty).toBeDefined();
      expect(angular.isFunction(MyController.changeProperty)).toBe(true);
    });

  });
```

For testing the actual **behaviour of a method** we need to fire it with different arguments.

```javascript
    it('should change myProperty', function() {
      MyController.changeProperty(true);
      expect(MyController.myProperty).toBe(true);
      MyController.changeProperty(false);
      expect(MyController.myProperty).toBe(false);
    });
```

While testing **calls to services** it's straight forward using Jasmine spies. 

```javascript
  describe('Synchronous calls', function() {

    it('should call syncCall on myService', function() {
      expect(myService.syncCall).toHaveBeenCalled();
      expect(myService.syncCall.calls.count()).toBe(1);
    });

  });
  
  describe('Asynchronous calls', function() {

    it('should call asyncCall on myService', function() {
      expect(myService.asyncCall).toHaveBeenCalled();
      expect(myService.asyncCall.calls.count()).toBe(1);
    });
    
  });
```

In adittion, we need to test **promise resolution** for asynchronous methods.

When setting the test suite we have made the asynchronous call to return a promise, that we can now resolve or reject as our convenience to test **success and error**.


```javascript
    it('should do something on success', function() {
      var data = ['something', 'on', 'success'];
      deferred.resolve(data); // Resolve the promise.
      scope.$digest();
      // Check for state on success.
      expect(MyController.myArray).toBe(data);
    });

    it('should do something on error', function() {
      deferred.reject(400); // Reject the promise.
      scope.$digest();
      // Check for state on error.
      expect(MyController.hasError).toBe(true);
    });
```  

And with the `controller as` syntax the only place where we need to test the `$scope` it's when we **emit or listen for events**.
  
```javascript
  describe('Events', function() {

    it('should emit an event', function() {
      expect(scope.$emit).toHaveBeenCalledWith('my-event');
      expect(scope.$emit.calls.count()).toBe(1);
    });

    it('should do something when some-event is caught', inject(function($rootScope) {
      $rootScope.$broadcast('some-event');
      // Check for state after event is caught.
      expect(MyController.myNumber).toBe(1);
    }));

  });
```

### Services

Services are even easier to test than controllers.

In addition of testing calls to other services, we should test for the output of our methods and HTTP requests.

But as we don't want to send XHR requests to a real server we use
[$httpBackend](https://docs.angularjs.org/api/ngMock/service/$httpBackend).

**Code**

We lay out a simple service with a synchronous method that calculates the factorial and one asynchronous that uses the native `$http` service to query an external API for data.

```javascript
angular.module('myServiceModule', [])
  .service('myService', MyService);
  
function MyService($http, $q) {

  var f = [];
  
  this.factorial = factorial;
  this.getThings = getThings;

  function factorial(n) {
    if (n === 0 || n === 1) { return 1; }
    if (f[n] > 0) { return f[n]; }
    f[n] = factorial(n-1) * n;
    return f[n];
  }

}
```

Note that if we return a non-promised value from the error callback it will resolve and not reject the derived promise. So we need to reject it explicitly.

```javascript
  function getThings() {
    return $http.get('/api/things').then(
      function(response) {
        return response.data;
      },
      function(error) {
        return $q.reject(error.status);
      }
    );
  }
```

**Specs**

Setting the test we need to get hold of the `$httpBackend` to mock the calls to the API and test for the expected results.

```javascript
describe('Service: myService', function() {

  beforeEach(module('myServiceModule'));

  var myService;
  var httpBackend;
  beforeEach(inject(function($httpBackend, _myService_) {
    httpBackend = $httpBackend;
    myService = _myService_;
  }));

});
```

For testing **synchronous methods** we just need to check that their output is the one we are expecting.

```javascript
  describe('Output of methods', function() {

    it('should return the product of all positive integers less than or equal to n', function() {
      expect(myService.factorial(0)).toBe(1);
      expect(myService.factorial(5)).toBe(120);
      expect(myService.factorial(10)).toBe(3628800);
    });

  });
```

But the fun starts when we test our **asynchronous methods**. 

For our tests to run quickly we use the mocked `$httpBackend` implementation to respond with data to our calls hence avoiding expensive requests to a real server.

As requests are treated asynchronously we need to flush them and that's the reason why we add an `afterEach` block to verify there is nothing pending at the end of the tests.

Testing that our **backend is being called** we just need to create expectations with the address of our API endpoint.

```javascript
  describe('HTTP calls', function() {

    afterEach(function() {
      httpBackend.verifyNoOutstandingExpectation();
      httpBackend.verifyNoOutstandingRequest();
    });

    it('should call the API', function() {
      httpBackend.expectGET(/\/api\/things/).respond('');
      myService.getThings();
      httpBackend.flush();
    });

  });
```

Though testing the **returned values** it's more complicated to set. 

First we need to spy upon an object with success and error handlers that will be passed to the method call.

```javascript
    var myThings;
    var errorStatus; = '';
    var handler;
    beforeEach(function() {
      myThings = [];
      errorStatus = '';
      handler = {
        success: function(data) {
          myThings = data;
        },
        error: function(data) {
          errorStatus = data;
        }
      }; 
      spyOn(handler, 'success').and.callThrough();
      spyOn(handler, 'error').and.callThrough();
    });
```

And after we need to create a backend definition with the response to test the handler has been called with the expected values.

```javascript
    it('should return an array of things on success', function() {
      var response = ['one thing', 'another thing'];

      httpBackend.whenGET(/\/api\/things/).respond(response);
      myService.getThings().then(handler.success, handler.error);
      httpBackend.flush();

      expect(handler.success).toHaveBeenCalled();
      expect(myThings).toEqual(response);
      expect(handler.error).not.toHaveBeenCalled();
      expect(errorStatus).toEqual('');
    });
```

If we need to **simulate an error** we have to pass a numeric value bigger than 300 as the first parameter of the response.

```javascript
    it('should return the status on error', function() {
      httpBackend.whenGET(/\/api\/things/).respond(404, {status: 404});
      myService.getThings().then(handler.success, handler.error);
      httpBackend.flush();

      expect(handler.error).toHaveBeenCalled();
      expect(errorStatus).toEqual(404);
      expect(handler.success).not.toHaveBeenCalled();
      expect(myThings).toEqual([]);
    });
```

### Directives

Directives are a bit more complex to test as we need to [$compile](https://docs.angularjs.org/api/ng/service/$compile) them manually to test the compiled DOM.

Additionally we use [angular.element](https://docs.angularjs.org/api/ng/function/angular.element) and the provided jQuery or jqLite methods to manipulate the DOM.

**Code**

We create a directive with an isolated scope and its own controller.

```javascript
angular.module('myDirectiveModule', [])
  .directive('myDirective', function() {
    return {
      bindToController: true,
      controller: function() {
        var vm = this;
        vm.doSomething = doSomething;
        function doSomething() {
          vm.something.name = 'Do something';
        }
      },
      controllerAs: 'vm',
      restrict: 'E',
      scope: {
        something: '='
      },
      templateUrl: 'my-directive.html'
    };
  });
```

And the external template to see `ng-html2js` in action.

```html
<h1 ng-click="vm.doSomething()">{{vm.something.name}}</h1>
```

**Specs**

To test it we need to load the directive's module but also the one holding all the templates that we have configured with `ng-html2js`.

Next we `$compile`the directive and apply the desired scope. 

```javascript
describe('Directive: myDirective', function() {

  beforeEach(module('myDirectiveModule'));
  beforeEach(module('templates'));

  var element;
  var scope;
  beforeEach(inject(function($rootScope, $compile) {
    scope = $rootScope.$new();
    element = angular.element('<my-directive something="thing"></my-directive>');
    element = $compile(element)(scope);
    scope.thing = {name: 'My thing'};
    scope.$apply();
  }));

});
```

Using the jQuery/jqLite methods we can check for what have been rendered.

```javascript
  it('should render something', function() {
    var h1 = element.find('h1');
    expect(h1.text()).toBe('My thing');
  });
```

And applying changes to the scope the rendered DOM should be updated.


```javascript
  it('should update the rendered text when scope changes', function() {
    scope.thing.name = 'My new thing';
    scope.$apply();
    var h1 = element.find('h1');
    expect(h1.text()).toBe('My new thing');
  });
```

If is needed we can test the directive controller grabbing an instance of it.


```javascript
  describe('Directive controller', function() {

    var controller;
    beforeEach(function() {
      controller = element.controller('myDirective');
    });

    it('should do something', function() {
      expect(controller.doSomething).toBeDefined();
      controller.doSomething();
      expect(controller.something.name).toBe('Do something');
    });

  });
```

### Providers

Providers are the toughest to test as we need to intercept them before they are injected.

But once we have solved that step we can test them as any other service.

**Code**

Our Hello World provider can be configured to say "hello" to anything.

```javascript
angular.module('myProviderModule')
  .provider('helloWorld', helloWorld);
  
function helloWorld() {
  var name;
  return {
    configure: function(value) {
      name = value;
    },
    $get: function() {
      return {
        sayHello: function() {
          return 'Hello ' + name;
        }
      };
    }
  };
}
```

**Specs**

We intercept the provider when we load the module before triggering the injection.

```javascript
describe('Provider: helloWorld', function() {

  var helloWorldProvider;
  beforeEach(function() {
    module('myProviderModule', function(_helloWorldProvider_) {
        helloWorldProvider = _helloWorldProvider_;
      });
  });

  var helloWorld;
  beforeEach(inject(function(_helloWorld_) {
    helloWorld = _helloWorld_;
  }));

});
```

And then we can test both instances.

```javascript
  it('should do something', function() {
    expect(!!helloWorldProvider).toBe(true);
  });

  describe('Service method', function() {

    beforeEach(function() {
      helloWorldProvider.configure('World');
    });

    it('should say hello world', function() {
      expect(helloWorld.sayHello()).toBe('Hello World');
    });

  });
```

## Conclusion and next steps

Although unit testing might seem scary at the beginning, once we master some of these patterns it's going to be a breeze to test our applications, making us feel more confident with the code we are developing.

But this is just the tip of the iceberg as unit testing help us to test isolated pieces of code. To check if all these pieces work well when integrated together we need to do [end to end testing](https://docs.angularjs.org/guide/e2e-testing) (e2e) and for it we have to use the [Protractor](https://angular.github.io/protractor/#/) framework, built by the Angular team.

