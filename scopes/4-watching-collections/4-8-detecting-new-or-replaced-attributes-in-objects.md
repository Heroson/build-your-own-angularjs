### 侦听对象属性的新增或替换
#### Detecting New Or Replaced Attributes in 

我们希望在对象新增属性时也触发一个变更事件：

_test/scope_spec.js_

```js
it('notices when an attribute is added to an object', function() {
  scope.counter = 0;
  scope.obj = {a: 1};

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.obj.b = 2;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

另外，属性值发生变化时我们也希望能被当作一个变更：

_test/scope_spec.js_

```js
it('notices when an attribute is changed in an object', function() {
  scope.counter = 0;
  scope.obj = { a: 1 };

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.obj.a = 2;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

这两个问题都能够用同样的方式进行解决。我们会对新对象的所有属性进行遍历，然后看旧对象同样的位置上是否有一样的值：

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  newValue = watchFn(scope);
  
  if (_.isObject(newValue)) {
    if (isArrayLike(newValue)) {
      // if (!_.isArray(oldValue)) {
      //   changeCount++;
      //   oldValue = [];
      // }
      // if (newValue.length !== oldValue.length) {
      //   changeCount++;
      //   oldValue.length = newValue.length;
      // }
      // _.forEach(newValue, function(newItem, i) {
      //   var bothNaN = _.isNaN(newItem) && _.isNaN(oldValue[i]);
      //   if (!bothNaN && newItem !== oldValue[i]) {
      //     changeCount++;
      //     oldValue[i] = newItem;
      //   }
      // });
    } else {
      // if (!_.isObject(oldValue) || isArrayLike(oldValue)) {
      //   changeCount++;
      //   oldValue = {};
      // }
      _.forOwn(newValue, function(newVal, key) {
        if (oldValue[key] !== newVal) {
          changeCount++;
          oldValue[key] = newVal;
        }
      });
    }
  } else {
    // if (!self.$$areEqual(newValue, oldValue, false)) {
    //   changeCount++;
    // }
    // oldValue = newValue;
  }

  // return changeCount;
};
```