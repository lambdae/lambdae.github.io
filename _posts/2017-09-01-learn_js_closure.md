---
layout: post
title: 快速理解js闭包
description:  通过简单的代码理解js的闭包
categories: javascript tech
author: lambdae
---

------

```javascript
function makeAdder(x) {
  return function(y) {
    return x + y;
  };
}

var add5 = makeAdder(5);
var add10 = makeAdder(10);

console.log(add5(2));  // 7
console.log(add10(2)); // 12

function makeFunc() {
	var name = "Mozilla"
	var cnt = 0;
	var toggle = false;
	return function() {
		toggle = !toggle;
		console.log(cnt++ + " " + name + " " + toggle + " ");
	};
}

var caller = makeFunc();

function trigger(calller) {
	calller();
}

for (var i = 5; i >= 0; i--) {
   caller();
}

trigger(makeFunc());
trigger(caller);

var student = (function() {
	var name = "";
	return {
		set: function(n) {
			name = n;
		},
		get: function() {
			return name;
		}
	}
})();

student.set("ydlme");
console.log(student.get());
```