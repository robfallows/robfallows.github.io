# Introduction

Meteor has had first-class support for Promises since v1.2 (September 2015), and first-class support for ES7 generators and async/await since v1.3 (March 2016).
For Meteor server-side code, that first-class support means that Meteor Promises play nicely with Fibers/Futures, which makes it possible to mix the sync-style coding we've become used to with the more mainstream Promise-based coding, including `async/await` found in most of the JavaScript ecosystem.

If you want to know more about the basics of Promises, generators or async/await, there's a huge number of resources on the Internet for you to refer to. However, there's neither a huge amount of information available, nor examples showing how we can make use of these awesome additions to our Meteor toolbox. This article assumes you know the fundamentals of what they are and instead concentrates on how we can use async/await in Meteor for fun and profit.

# Use Case

I'm going to be querying a MySQL database using the npm package [promise-mysql]( https://github.com/lukeb-uk/node-promise-mysql). This uses the popular [Bluebird]( https://github.com/petkaantonov/bluebird) Promise library, so I'll be able to demonstrate how to make use of 3rd-party Promises within the context of a Meteor application using inbuilt Meteor Promises. Even more important, I'll show you how to get past the "`Error: Meteor code must always run within a Fiber`" issue.

# Setting up

I'm going to base this on the queries presented in the `promise-mysql` README to give you a taste of what you need to do, so I need a database to play with. My database was created with the following SQL (I did this through the `mysql` command line):

```sql
CREATE DATABASE `mordor`;
USE `mordor`;

CREATE TABLE `hobbits` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `name` varchar(255) NOT NULL,
 PRIMARY KEY (`id`),
 KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `hobbits` VALUES
  (1, 'Bilbo'),
  (2, 'Frodo'),
  (3, 'Smeagol'),
  (4, 'Sam');

CREATE TABLE `items` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `name` varchar(255) NOT NULL,
 `owner` int(11) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `name` (`name`),
 KEY `owner` (`owner`),
 CONSTRAINT `fk_owner` FOREIGN KEY (`owner`) REFERENCES `hobbits` (`id`) ON DELETE SET NULL ON UPDATE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `items` VALUES
  (1,'ring', 3);

CREATE USER 'sauron'@'localhost' IDENTIFIED BY 'theonetruering';
GRANT ALL ON `mordor`.* TO 'sauron'@'localhost';
```

## Create a Meteor application

```
meteor create tolkien
cd tolkien
meteor npm i
meteor npm install --save babel-runtime promise-mysql
```

The README uses this code:

```js
var mysql = require('promise-mysql');
var connection;

mysql.createConnection({
  host: 'localhost',
  user: 'sauron',
  password: 'theonetruering',
  database: 'mordor'
}).then(function(conn){
  connection = conn;

  return connection.query('select `name` from hobbits');
}).then(function(rows){
  // Logs out a list of hobbits
  console.log(rows);
});
```

I could use that as-is, but I'm going to make some changes for Meteor and ES6. Otherwise, this is the same Promise chain as above:

### server/main.js

```js
import { Meteor } from 'meteor/meteor';
import mysql from 'promise-mysql';

Meteor.startup(() => {
  let connection;

  mysql.createConnection({
    host: 'localhost',
    user: 'sauron',
    password: 'theonetruering',
    database: 'mordor'
  }).then(conn => {
    connection = conn;

    return connection.query('select `name` from hobbits');
  }).then(rows => {
    console.log(rows);
  }).then(() => {
    connection.end(); // Let's close the connection when we've done with it
  });

});
```

Logs the following to the server console:

```
I20170118-14:46:47.207(0)? [ RowDataPacket { name: 'Bilbo' },
I20170118-14:46:47.323(0)?   RowDataPacket { name: 'Frodo' },
I20170118-14:46:47.323(0)?   RowDataPacket { name: 'Smeagol' },
I20170118-14:46:47.323(0)?   RowDataPacket { name: 'Sam' } ]
```

## Using `async/await`

The above's all well and good, but it's not often that we want to use `console.log` as a user interface to a Meteor application. Let's make some changes so we can call a server-side method from the client and get the results back in the browser.

Did you know you can use Promises in Meteor methods? Did you know you can write `async` methods?

### server/main.js

```js
import { Meteor } from 'meteor/meteor';
import mysql from 'promise-mysql';

Meteor.methods({
  async getHobbits() {
    const connection = await mysql.createConnection({
      host: 'localhost',
      user: 'sauron',
      password: 'theonetruering',
      database: 'mordor'
    });
    const rows = connection.query('select `name` from hobbits');
    connection.end();
    return rows;
  },
});
```

I hope you agree that looks much simpler - and much more like traditional Meteor "sync style" coding. Now for the client stuff. Yes, I'm using Blaze. Deal with it.

### client/main.html

```handlebars
<head>
  <title>tolkien</title>
</head>

<body>
  {{> hello}}
</body>

<template name="hello">
  {{#each hobbit in hobbits}}
    <div>{{hobbit.name}}</div>
  {{/each}}
</template>
```

### client/main.js

```js
import { Template } from 'meteor/templating';
import { ReactiveVar } from 'meteor/reactive-var';

import './main.html';

Template.hello.onCreated(function helloOnCreated() {
  this.list = new ReactiveVar([]);
  Meteor.call('getHobbits', (error, result) => {
    this.list.set(result);
  });
});

Template.hello.helpers({
  hobbits() {
    return Template.instance().list.get();
  },
});
```

In my browser I get this (trust me):

```
Bilbo
Frodo
Smeagol
Sam
```

## Now let's break Meteor

I'm going to go back to the original server code I wrote and modify it to copy the rows read from MySQL into a MongoDB collection. Why? Because this is a classic way to provoke the "must run in a Fiber" error.

### imports/api/Hobbitses.js

```js
import { Mongo } from 'meteor/mongo';
export const Hobbitses = new Mongo.Collection('Hobbitses');
```

### server/main.js

```js
import { Meteor } from 'meteor/meteor';
import mysql from 'promise-mysql';
import { Hobbitses } from '/imports/api/Hobbitses';

Meteor.startup(() => {
  let connection;

  mysql.createConnection({
    host: 'localhost',
    user: 'sauron',
    password: 'theonetruering',
    database: 'mordor'
  }).then(conn => {
    connection = conn;

    return connection.query('select `name` from hobbits');
  }).then(rows => {
    rows.forEach(row => {
      Hobbitses.insert({name: row.name});
    });
  }).then(() => {
    connection.end();
  });

});
```

Running my application gives me that familiar error:

```
W20170118-15:51:20.479(0)? (STDERR) Unhandled rejection Error: Meteor code must always run within a Fiber. Try wrapping callbacks that you pass to non-Meteor libraries with Meteor.bindEnvironment.
```

There are two ways to address this. The first (hardest) is to not use the `minimongo` version of `insert`, which requires a fiber - not present within a Bluebird Promise (or other 3rd party Promise). However, we can instead use a Promised `insert`. Conveniently, that's available by using the [underlying node library](http://mongodb.github.io/node-mongodb-native/2.2/api/Collection.html#insert) via Meteor's `rawCollection` method:

### server/main.js

```js
import { Meteor } from 'meteor/meteor';
import mysql from 'promise-mysql';
import { Hobbitses } from '/imports/api/Hobbitses';

Meteor.startup(() => {
  let connection;

  mysql.createConnection({
    host: 'localhost',
    user: 'sauron',
    password: 'theonetruering',
    database: 'mordor'
  }).then(conn => {
    connection = conn;

    return connection.query('select `name` from hobbits');
  }).then(rows => {
    rows.forEach(row => {
      Hobbitses.rawCollection().insert({name: row.name});
    });
  }).then(() => {
    connection.end();
  });
});
```

```
meteor mongo
meteor:PRIMARY> db.Hobbitses.find()
{ "_id" : ObjectId("587f90926f34f736c5213ff6"), "name" : "Bilbo" }
{ "_id" : ObjectId("587f90926f34f736c5213ff7"), "name" : "Frodo" }
{ "_id" : ObjectId("587f90926f34f736c5213ff8"), "name" : "Smeagol" }
{ "_id" : ObjectId("587f90926f34f736c5213ff9"), "name" : "Sam" }
```

But there's a better way! The observant among you will have noticed that the MongoDB `_id`s you get when using `rawCollection` methods are not simple strings - that may be an issue in your app. In any case, aren't we supposed to be using `async/await`?

### server/main.js

```js
import { Meteor } from 'meteor/meteor';
import mysql from 'promise-mysql';
import { Hobbitses } from '/imports/api/Hobbitses';

Meteor.methods({
  async getHobbits() {
    const connection = await mysql.createConnection({
      host: 'localhost',
      user: 'sauron',
      password: 'theonetruering',
      database: 'mordor'
    });
    const rows = await connection.query('select `name` from hobbits');
    rows.forEach(row => {
      Hobbitses.insert({name: row.name});
    });
    connection.end();
  },
});
```

```
meteor mongo
meteor:PRIMARY> db.Hobbitses.find()
{ "_id" : "CsQLu55nJSLaqSeig", "name" : "Bilbo" }
{ "_id" : "2vwsgW9Latb8fZSb9", "name" : "Frodo" }
{ "_id" : "2gBD7o9CY6trheofH", "name" : "Smeagol" }
{ "_id" : "X5ok4axbZ6Hv6evg4", "name" : "Sam" }
```

We just used `await` to effectively "cast" a 3rd party Promise into a Meteor Promise, so we can mix standard "sync-style" Meteor with 3rd party Promises. Awesome!

# In Conclusion

It's perfectly possible to make use of the vast number of Promise-based libraries available in the wider JavaScript ecosystem in Meteor apps. Using `aysnc/await` is a good way of mixing modern syntax with classic Meteor sync-style (Fiber based) code without hitting the dreaded "must run in a Fiber" error.
