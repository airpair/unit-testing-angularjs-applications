Unit testing, as the name implies, is about testing individual units of code.

This post tries to show some patterns and guidelines to help us with unit testing Angular applications after setting some basic configuration.

## Configuration

Before we start testing we need to install and configure some dependencies. For this we will be using the package manager [npm](https://www.npmjs.com/), although [Bower](http://bower.io/) can be used as well.

```
$ npm install angular
$ bower install angular
```

### ngMock

Angular provides [ngMock](https://docs.angularjs.org/api/ngMock) to inject and mock services into unit tests. And one of the most useful parts of the `ngMock` module is `$httpBackend` which lets us mock XHR requests.

```
$ npm install angular-mocks --save-dev
```

### Karma

[Karma](http://karma-runner.github.io/) is a test runner written by the Angular team which allow us to execute tests in multiple browsers.

```
$ npm install karma --save-dev
```

After installing it we need to crete a configuration file running `karma init` but to extend its functionality we need to install first some plugins.

#### Testing framework

Angular can be tested using any JavaScript unit testing framework out there, but [Jasmine](http://jasmine.github.io/) is probably the most popular.

```
$ npm install karma-jasmine jasmine-core --save-dev
```

#### Browsers

Karma takes care of auto-capturing and killing the browsers, but we need to install at least one of these launchers.

```
$ npm install karma-chrome-launcher --save-dev
$ npm install karma-phantomjs-launcher --save-dev
$ npm install karma-firefox-launcher --save-dev
$ npm install karma-safari-launcher --save-dev
$ npm install karma-opera-launcher --save-dev
$ npm install karma-ie-launcher --save-dev
```

#### Templates preprocessor

When testing directives we need to set up Karma to serve our templates using a preprocessor to convert HTML into JS string.

[ng-html2js](https://github.com/karma-runner/karma-ng-html2js-preprocessor) creates a "templates" module and put the HTML into the `$templateCache`.

```
$ npm install karma-ng-html2js-preprocessor --save-dev
```

#### karma.conf.js

If we haven't generated yet a configuration file now it's the time.

```
$ karma init
```

After answering some questions it will create a ``karma.conf.js`` file which we still have to modify for at least configuring the installed preprocessors.

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
      'src/**/*.html': ['ng-html2js']
    },

    ngHtml2JsPreprocessor: {
      // strip this from the file path
      stripPrefix: 'src/',
      // create a single module that contains templates from all the files
      moduleName: 'templates'
    },

    reporters: ['progress'],

    port: 9876,

    colors: true,

    logLevel: config.LOG_INFO,

    autoWatch: true,

    browsers: ['Chrome'],

    singleRun: false
  });
};
```

## Mocks

To test the functionality of a piece of code in isolation we need to mock any dependency.

Angular is written with testability in mind and come with [dependency injection](https://docs.angularjs.org/guide/di) built in, what help us to achieve decoupling and therefore to test our objects in isolation.

A service (service, factory, value, constant, or provider) is the most common type of dependency in Angular applications and they are defined via providers.

The Angular [injector](https://docs.angularjs.org/api/auto/service/$injector) retrieves object instances as defined by providers.

So there are at least two ways we can mock our services:

### Using $provide

We can provide our own implementation in a `beforeEach` Jasmine block.

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

### Creating a provider mock

Or we can reuse it creating the implementation in a separate `my-service.mock.js` file.

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

## Testing

We can make testing simpler if we follow the same pattern to test each of our objects:

1. Describe the object with type and name.
2. Load our object's module.
3. Load mock modules as needed.
4. Inject dependencies and spy on methods.
5. Initialize the object:
  1. Services just need to get injected.
  2. Controllers are instantiated using the $controller service.
  3. We need to $compile directives.
6. Write expectations grouped in describe blocks.

As code and specs are better placed side-by-side we can put all our files inside the `src` folder, using the `.spec.js` suffix to differentiate test files.

```
src/my-controller.js
src/my-controller.spec.js
src/my-directive.js
src/my-directive.spec.js
src/my-service.js
src/my-service.spec.js

```

### Controllers

Controllers are easy to test as long as we don't manipulate the DOM and functions have a single and clear purpose.

We should test for the state, synchronous and asynchronous calls to services, and events.

#### Code

We create our basic controller using the `controller as` syntax and injecting the required dependencies.

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

####Specs

We create the basic structure of the test suite loading the mocked service after the controller's module. 

```javascript
describe('Controller: MyController', function() {

  beforeEach(module('myControllerModule'));
  beforeEach(module('myServiceMock'));

  var myService;
  var deferred;
  // Mock services and spy on methods
  beforeEach(inject(function($q, _myService_) {
    deferred = $q.defer();
    myService = _myService_;
    spyOn(myService, 'syncCall').and.callThrough();
    spyOn(myService, 'asyncCall').and.returnValue(deferred.promise);
  }));

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
  
  // Write specs.
  
});
```

And we start testing the state of our controller. As we use the `controller as` syntax we don't need to test for the `$scope` but directly for the exposed properties and methods of the controller.


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

    it('should change myProperty', function() {
      MyController.changeProperty(true);
      expect(MyController.myProperty).toBe(true);
      MyController.changeProperty(false);
      expect(MyController.myProperty).toBe(false);
    });

  });
```

For testing calls to other services we need to spy their methods. And Jasmine provides us with the [spyOn](http://jasmine.github.io/2.0/introduction.html#section-Spies) function to track calls and arguments passed.

When setting the test suite we have made the asynchronous call to return a promise, that we can now resolve or reject as our convenience.

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

  });
```  

And the only place where we need to test the `$scope`, it's when we emit or listen for events.
  
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

####Code

We layout a simple service with a couple of synchronous methods and one asynchronous that use the native `$http` service to query an external API for data.

```javascript
angular.module('myServiceModule', [])
  .service('myService', MyService);
  
function MyService($http, $q) {

  var f = [];

  this.factorial = factorial;
  this.syncCall = syncCall;
  this.asyncCall = asyncCall;

  function factorial(n) {
    if (n === 0 || n === 1) {
      return 1;
    }
    if (f[n] > 0) {
      return f[n];
    }
    f[n] = factorial(n-1) * n;
    return f[n];
  }

  function syncCall() {
    return {
      name: 'Synchronous Call'
    };
  }

  function asyncCall() {
    return $http.get('http://jsonplaceholder.typicode.com/users').then(
      function(response) {
        return response.data;
      },
      function(error) {
        return $q.reject(error.status);
      }
    );
  }

}
```


####Specs

For testing the synchronous methods we just need to check that their output is the one we are expecting.

```javascript
describe('Service: myService', function() {

  beforeEach(module('myServiceModule'));

  var myService;
  var httpBackend;
  beforeEach(inject(function($httpBackend, _myService_) {
    httpBackend = $httpBackend;
    myService = _myService_;
  }));

  describe('Output of methods', function() {

    it('should return the product of all positive integers less than or equal to n', function() {
      expect(myService.factorial(0)).toBe(1);
      expect(myService.factorial(5)).toBe(120);
      expect(myService.factorial(10)).toBe(3628800);
    });

  });

});
```

But the fun starts when we test our asynchronous methods as we need to get hold of the `$httpBackend` to mock the calls to the API and test for the expected results.

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

    it('should return an array of things on success', function() {
      var response = ['one thing', 'another thing'];
      var myThings = [];
      var errorStatus = '';
      var handler = {
        success: function(data) {
          myThings = data;
        },
        error: function(data) {
          errorStatus = data;
        }
      };
      spyOn(handler, 'success').and.callThrough();
      spyOn(handler, 'error').and.callThrough();

      httpBackend.whenGET(/\/api\/things/).respond(response);
      myService.getThings().then(handler.success, handler.error);
      httpBackend.flush();

      expect(handler.success).toHaveBeenCalled();
      expect(myThings).toEqual(response);
      expect(handler.error).not.toHaveBeenCalled();
      expect(errorStatus).toEqual('');
    });

  });
```

### Directives

Directives are a bit more complex to test as we need to [$compile](https://docs.angularjs.org/api/ng/service/$compile) them manually to test the compiled DOM.

Additionally we use [angular.element](https://docs.angularjs.org/api/ng/function/angular.element) and the provided jQuery or jqLite methods to manipulate the DOM.

####Code

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
      templateUrl: 'app/my-directive.html'
    };
  });
```

And the external template to see `ng-html2js` in action.

```html
<h1 ng-click="vm.doSomething()">{{vm.something.name}}</h1>
```

####Specs

To test it we need to load the directive's module but also the one holding all the templates that we have configured with `ng-html2js`.

Then we `$compile`the directive and apply the desired scope so we can check for what have been rendered using the jQuery/jqLite helpers.

```js
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

  it('should render something', function() {
    var h1 = element.find('h1');
    expect(h1.text()).toBe('My thing');
  });

  it('should update the rendered text when scope changes', function() {
    scope.thing.name = 'My new thing';
    scope.$apply();
    var h1 = element.find('h1');
    expect(h1.text()).toBe('My new thing');
  });

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

####Code

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

####Specs

To test it we first intercept the provider and after we inject the service so we can test both instances.

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

});
```

##Conclusion

Although unit testing might seem scary at the beginning, once we master some of these patterns it's going to be a breeze to test our applications, making us feel more confident with the code we are developing.

