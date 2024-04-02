---
title: "Prototype pollution"
author: filip
date: 2024-04-02 19:10:00 +0800
categories: [Hack The Box, Machines, Web Security]
tags: [Notes, Prototype Pollution, Web Security, nodeJS]
render_with_liquid: false
---

## Introduction

I have prepared material on the Prototype pollution topic for today’s posts based on my notes from 2023 for a speech. It might be already deprecated partially, or there are some new materials or approaches on this topic, and maybe I'll update this page one day. Nevertheless this note contains a lot of basics that less probable to change over time, so I hope you'll find it useful.

To begin, let's start with a brief overview:

## Short statistics

Node.js is a JavaScript runtime that allows developers to build server-side applications using JavaScript. It is an open-source, cross-platform environment for executing JavaScript code on the server side. Node.js uses the V8 JavaScript engine, which is also used in Google Chrome, to execute JavaScript code.

In 2021 and 2022, Node.js continues to be a popular choice among developers for building server-side applications. According to the Stack Overflow Developer Survey 2021, Node.js is the fourth most popular technology among developers, behind only HTML/CSS, JavaScript, and SQL. The same survey also found that Node.js is the second most loved technology among developers, behind only Rust. Additionally, ~~According~~ to the Node.js User Survey of 2022, Node.js is used by more than 50% of developers in the US and Europe, and it is the most popular runtime for microservices.

According to cve.mitre.org, the prototype pollution vulnerability was discovered 226 times in total from 2018 to 2022 affecting different products and quite popular JavaScript libraries. But what is it?

## Prototype pollution

Prototype pollution is a type of vulnerability that occurs when an attacker could modify an object's prototype, either by directly modifying it or by injecting properties or methods into it. The injected value is processed by a JavaScript function that merges user-provided properties into an existing object without sanitizing user input.

In most cases, the vulnerability occurs when an attacker can inject a malicious property using the **`__proto__`** property. This allows the attacker to corrupt the prototype with properties that contain harmful values, which the application can then use dangerously.

This vulnerability appears in both JavaScript libraries such as ’**[lodash](https://snyk.io/vuln/SNYK-JS-LODASHMERGE-173732)’**, potentially affecting many frontend projects due to the popularity of ‘lodash’, as well as Node.js backend libraries that deal with JavaScript object cloning such as **[node.extend](https://snyk.io/vuln/SNYK-JS-NODEEXTEND-73641)**, and **[deep-extend](https://snyk.io/vuln/npm:deep-extend:20180409)**.

For the successful exploitation of prototype pollution, the following factors must be present:

1. A source of prototype pollution, such as URL parameters, JSON-based input, or web messages.
2. A vulnerable sink, such as a JavaScript function or DOM element, allows for arbitrary code execution. Where input is processed.
3. A vulnerable property that is passed to the sink without proper sanitation.

### What can it lead to?

It depends on whether the prototype pollution is client-side:

- Cross-Site Scripting (XSS)
- Client-Side DoS
- HTML injection
- etc.

or server-side:

- Remote Code Execution (RCE)
- Privilege escalation
- DoS
- Other types of injections like SQL injections, multiple stored XSS, etc. (depends on the context of sink)

 

But before uncovering more technical details of this kind of web application vulnerability, we need to deep dive into JavaScript basics and understand what’s under the hood.

### About objects, inheritance, and prototypes in JavaScript

Let’s also recap what [w3schools](https://www.w3schools.com/js/js_object_definition.asp) says about the important for us JavaScript basics, so everyone understands the difficulty and impact of a Prototype pollution vulnerability and its exploitation.

#### Objects

In JavaScript, almost "everything" is an object.

- Booleans can be objects (if defined with the `new` keyword)
- Numbers can be objects (if defined with the `new` keyword)
- Strings can be objects (if defined with the `new` keyword)
- Dates are always objects
- Maths are always objects
- Regular expressions are always objects
- Arrays are always objects
- Functions are always objects
- Objects are always objects

All JavaScript values, except primitives, are objects.

#### Primitives

A **primitive value** is a value that has no properties or methods.

**314** is a primitive value

JavaScript defines 7 types of primitive data types:

**Examples**

- `string`
- `number`
- `boolean`
- `null`
- `undefined`
- `symbol`
- `bigint`

Primitive values are immutable (they are hardcoded and cannot be changed).

| Value | Type | Comment |
| --- | --- | --- |
| "Hello" | string | "Hello" is always "Hello" |
| 3.14 | number | 3.14 is always 3.14 |
| true | boolean | true is always true |
| false | boolean | false is always false |
| null | null (object) | null is always null |
| undefined | undefined | undefined is always undefined |

A JavaScript object is a collection of **named values**

It is a common practice to declare objects with the `const` keyword.

**Example**

`let person = {firstName:"John", lastName:"Doe", age:50, eyeColor:"blue"};`

The named values, in JavaScript objects, are called **properties**.

| Property | Value |
| --- | --- |
| firstName | John |
| lastName | Doe |
| age | 50 |
| eyeColor | grey |

#### Properties

A JavaScript object is a collection of unordered properties.

Properties are the values associated with a JavaScript object.

The syntax for accessing the property of an object is:

**`objectName.property      // person.age`**

**`objectName["property"]   // person["age"]`**

**`objectName[expression]   // x = "age"; person[x]`**

#### Methods

A JavaScript **method**
 is a property containing a **function definition.**

| Property | Value |
| --- | --- |
| firstName | John |
| lastName | Doe |
| age | 50 |
| eyeColor | blue |
| fullName | function() {return this.firstName + " " + this.lastName;} |

Methods are **actions** that can be performed on objects. They could be primitive values, other objects, and functions.

Methods are functions stored as object properties.

You access an object method with the following syntax:

`objectName.methodName()`

#### Constructors:

```jsx
function Person(first, last, age, eye) {
  this.firstName = first;
  this.lastName = last;
  this.age = age;
  this.eyeColor = eye;
}
```

In a constructor function keyword **"`this`"**  does not have a value. It is a substitute for the new object. The value of `this.property`
 will become the new object when a new object is created.

<aside>
💡 Keyword `this`
 is not a variable. It is just a keyword. You cannot change the value of `this`.

</aside>

In the example above, `function Person()`
 is an object constructor function.

Objects of the same type could be created by calling the constructor function with the `new` keyword:

const myFather = new Person("John", "Doe", 50, "blue");

you can **not** add a new property to an existing object constructor:

```jsx
function Person(first, last, age, eye) {
  this.firstName = first;
  this.lastName = last;
  this.age = age;
  this.eyeColor = eye;
}
Person.worker = true; //will not work

const myFather = new Person("John", "Doe", 50, "blue");
myFather.worker = true;// will work because it is assigned to the object myFather
```

#### Prototypes:

All JavaScript objects inherit properties and methods from a prototype.

- `Date` objects inherit from `Date.prototype`
- `Array` objects inherit from `Array.prototype`
- `Person` objects inherit from `Person.prototype`

The `Object.prototype` is on the top of the prototype inheritance chain:

`Date` objects, `Array` objects, and `Person` objects  inherit from `Object.prototype`.

**Why prototypes exist?**

Sometimes you want to add new properties (or methods) to **all** existing objects of a given type or to an object constructor.

The JavaScript `prototype` property allows you to add new properties  to object constructors:

```jsx
function Person(first, last, age, eyecolor) {
  this.firstName = first;
  this.lastName = last;
  this.age = age;
  this.eyeColor = eyecolor;
}

Person.prototype.nationality = "Ukrainian";// this would add a new property to the Person globally for further usage.

const myFather = new Person("Mykola", "Parasiuk", 50, "blue"); //note, we are not setting the nationality in this case
//but previously created property 'nationality' will be inherited and set as 'Ukrainian'

console.log(myFather.nationality);//output: Ukrainian
```

![Untitled]( assets/posts/pp/Untitled.png)

The JavaScript `prototype`
 property also allows you to add new **methods** to objects constructors:

```jsx
function Person(first, last, age, eyecolor) {
  this.firstName = first;
  this.lastName = last;
  this.age = age;
  this.eyeColor = eyecolor;
}

Person.prototype.name = function() {
  return this.firstName + " " + this.lastName;
};
```

**One more important moment:**

In javaScript also exists `__proto__` which points to the parental object prototype.

```jsx
const user =  {
    username: "mykolaparasiuk", //is an Object -> String -> username:"value"
    userId: 01234, //is an Object -> number -> userId:value
    isAdmin: false //is an Object -> boolean -> isAdmin:value
}

//so
username.__proto__                        // String.prototype
username.__proto__.__proto__              // Object.prototype
username.__proto__.__proto__.__proto__    // null

//the same but in other syntax

constructor.prototype
constructor[prototype][property] = value
```

Let’s move to the practical examples, shall we?

## Practical examples

**Example:**

**JSON input:**

let’s assume there is a target application that looks like … and successful exploitation of the prototype pollution vulnerability will lead to privilege escalation for all users in the application:

```jsx
function User(username, password, bio) {
  this.username = username;
  this.password = password;
  this.bio = bio;
}

var username = "Mykola_Parasiuk";
var password = "does_not_matter";
var bio = "My kid has electric car";
//create by default a non-admin user (this creation is a back-end process)
let user = new User(username, password, bio);
```

and
assume the admin is created in similar way, 
but then it has an additional logic:
```jsx
var username = "Admin_Zenik";
var password = "I_am_admin";
var bio = "angry ukrainian rap-singer";

let admin = new User(username, password, bio);
...
//some logic that makes admin - isAdmin=true;
if (!admin.isAdmin){
admin.isAdmin = true;
}
//then admin logic.
```

    

```jsx
function deepCopy(dest, src) {
  for (var key in src) {
    if (!src[key] || typeof src[key] !== 'object') {
      dest[key] = src[key];
      continue;
    }
    if (!dest[key] || (typeof dest[key] !== 'object' && typeof dest[key] !== 'function')) {
      dest[key] = {};
    }
    deepCopy(dest[key], src[key]);
  }
  return dest;
}

deepCopy({}, insecureInput);
```

As well as the application allows users to update their user fields an attacker
could send a request with the following input knowing the application does not:
sanitize the input properly:
```plaintext
PATCH /users/example
Content-Type: application/json


{
    "username": "Mykola_Parasiuk",
    "password": "does_not_matter",
    "bio": "I am a simple user, a happy family guy.",
    "__proto__":{
			"isAdmin": true			
}
}
```


```jsx

///the application makes some parsing of the input
const insecureInput = JSON.parse(`
  {
    "username": "Mykola_Parasiuk",
    "password": "does_not_matter",
    "bio": "I am a simple user, happy family guy.",
		"__proto__": { "isAdmin": true }
}
`);

///
// A sample function that processes the user input and copies its key:value pairs
// in to the provided object.
function deepCopy(dest, src) {
  for (var key in src) {
    if (!src[key] || typeof src[key] !== 'object') {
      dest[key] = src[key];
      continue;
    }
    if (!dest[key] || (typeof dest[key] !== 'object' && typeof dest[key] !== 'function')) {
      dest[key] = {};
    }
    deepCopy(dest[key], src[key]);
  }
  return dest;
}

deepCopy({}, insecureInput);

//credits to: https://gist.github.com/sttk/9e83d802c4a1a2f24fab807b0644a8db
```
Simplified illustration:
![Untitled]( assets/posts/pp/Untitled 1.png)
![Untitled]( assets/posts/pp/Untitled 2.png)



#### Other examples:

**Injection through URL parameter:**

https://vulnerable-website.com/?__proto__[badProperty]=payload

```jsx
targetObject.__proto__.badProperty = 'payload';
```

**The application had the following code, which led to the RCE. How? lets inspect the code:**

```jsx
const Message = require('models/Message');
const _ = require('lodash');
const { exec } = require('child_process');

const messages_send = async(req,res)=>{
	const tokent = req.headers['X-Token']
	if(req.body.text){
		
		const message = {
			user_sent: token,
			title: "message for admins",
			};
		
		_.merge(message, req.body);

		exec('log.sh log_message')
		Message.create({
			text: JSON.stringify(message),
			user_sent: token
			});
		return res.json({Status: 200});
	}
	return res.json({Status: 404, Message: "parameter text not found"});
}
...
```

![Untitled]( assets/posts/pp/Untitled 3.png)

As you can see, the example uses the '**lodash**' method **`merge`** and the sink **`exec`**. 

The **`exec`** function, according to its documentation, accepts various parameters that can be set explicitly or implicitly. 

![Untitled]( assets/posts/pp/Untitled 4.png)

The application also accepts from users message submission using JSON.

If an attacker submits a JSON request containing a prototype pollution payload along with legitimate text, they may be able to achieve command execution. To perform a command injection within the context of the **`exec`**function, we can refer to the documentation to construct our payload, by setting properties such as **`shell`**, **`argv0`**, and **`NODE_OPTIONS`**for all objects.

![Untitled]( assets/posts/pp/Untitled 5.png)

As a result:

![Untitled]( assets/posts/pp/Untitled 6.png)

There are also a lot of other scenarios already discovered which lead to RCE ([https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce#vm-gadgets)):

![Untitled]( assets/posts/pp/Untitled 7.png)

## How to find prototype pollution?

- For client-side prototype pollution a DOM Invader (Portswigger) is awesome.
- For back-end side prototype pollution: look for dependencies vulnerable to prototype pollution and investigate the way they are vulnerable. As well as manual approach. Look for suspicious custom functions perform object copy\merge.

## Remediation

- Freeze properties with Object.freeze (Object.prototype)
- Perform validation on the JSON inputs in accordance with the application’s schema
- Avoid using recursive merge functions in an unsafe manner
- Use objects without prototype properties, such as `Object.create(null)`, to avoid affecting the prototype chain
- Use `Map` instead of `Object`
- Regularly update new patches for libraries

## Where to practice a prototype pollution:

- Portswigger academy: client-side prototype pollution labs
- Hack The Box: Pollution, Breaking grad.
- Offensive Security: AWAE (one of labs)
- Pentesterlabs: JS Prototype Pollution lab

## References

[https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution#what-can-i-do-to-prevent](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution#what-can-i-do-to-prevent)

[https://snyk.io/blog/after-three-years-of-silence-a-new-jquery-prototype-pollution-vulnerability-emerges-once-again/](https://snyk.io/blog/after-three-years-of-silence-a-new-jquery-prototype-pollution-vulnerability-emerges-once-again/)

[https://portswigger.net/web-security/prototype-pollution](https://portswigger.net/web-security/prototype-pollution)

[https://portswigger.net/burp/documentation/desktop/tools/dom-invader/prototype-pollution#detecting-sources-for-prototype-pollution](https://portswigger.net/burp/documentation/desktop/tools/dom-invader/prototype-pollution#detecting-sources-for-prototype-pollution)

[https://www.w3schools.com/js/js_object_definition.asp](https://www.w3schools.com/js/js_object_definition.asp)

[https://gist.github.com/sttk/9e83d802c4a1a2f24fab807b0644a8db](https://gist.github.com/sttk/9e83d802c4a1a2f24fab807b0644a8db)

[https://portswigger.net/web-security/prototype-pollution/preventing](https://portswigger.net/web-security/prototype-pollution/preventing)