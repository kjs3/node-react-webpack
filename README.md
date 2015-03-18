# Node-react-webpack

An example using Express.js to do server-side rendering with React, React Router, and Webpack 

Installing Node with NVM

If you've already installed via homebrew it's probably a good idea to uninstall that first.

`brew uninstall node`

Check out NVM [here](https://github.com/creationix/nvm) and run the install script in your terminal. 

I'm creating this project with IO.js 1.2 (for node-sass compatibility).

`nvm install 1.2` // nvm knows this is an IOjs version  

`nvm alias default 1.2` // sets IO.js 1.2 as your default node

Note: IO.js is a fork of Node.js motivated (at least partly) because Joyent, who oversees the Node project, is slow to move forward and accept patches from the Node community. In particular, the version of V8 that Node uses (v0.12) isn't even supported by Google anymore. IO.js is staying up to date with V8, merging fixes faster in general, and is getting a ton of support from the Node community. IO.js also uses the SemVer naming scheme and they are on version 1.5.1 as of this writing. 

From this point on I'm going to refer to Node and IO.js interchangably. I'll also probably stop capitalizing both at some point.

Ok, we're not going to do _everything_ manually but almost. At least at first.
We'll start off with a basic Express.js app as opposed to straight Node. Express is comparable to the Sinatra Ruby framework. You get a nice way to define routes and to use middleware to intercept and run code as a request comes into the server and as the response leaves the server. 

### Create a basic Express app.

`mkdir rentals-node-test`

`cd rentals-node-test`

Run `npm init` to start a new node project.

I changed some of the defaults which you can see in the git repo.

Install the debug and express modules.

`npm install debug express --save`

Create a server.js file and put the following in it.

```
// server.js

// requiring needed modules
var debug = require('debug')('server');
var express = require('express');
var app = express();

// setting up a simple route
app.get('/', function (req, res) {
  res.send('Hello Rentals!');
});

// defining a port app variable
app.set('port', process.env.PORT || 3000);

// binding the app to the port with some debug output 
app.listen(app.get('port'), function () {

  var env = app.get('env');
  var port = this.address().port;

  debug('Serving ' + app.get('env') + ' rentals on port %s', port);
});
```

Start the app:  

`DEBUG=server iojs server.js`

Go to `http://localhost:3000` and you should see "Hello Rentals!".

### Sending HTML

Let's send an actual HTML document to the browser instead of plain text.

At first we'll just do it inline.

Change the response send command to the following.

`res.send('<doctype!><html><body><h1>HTML Rentals!</h1></body></html>');`

Note: Sadly you can't break the quoted string over multiple lines. Actually it should work if you end each line with a "/" and no trailing space but just don't. :) It's non-standard js.

Alright, lets shift this response to its own file. 
Make a file at the root of the directory called index.html and copy/paste our one-liner into it (don't restructure the HTML, you'll need it back in one line in a second).
We need to add the path module (part of Node standard lib so no need to install).

`var path = require('path');`

And make a small change to our response.

`res.sendFile(path.join(__dirname, 'index.html'));`

`__dirname` is a global that node sets up that points to wherever you started the node process.

Restart the server, refresh your browser, and you should see the output of the index.html file.

sendFile automatically sets the ContentType header based on the file extension.

### Rendering HTML from a JS file

Alright, let's shift to rendering some a HTML that is output from Javascript.

Rename `index.html` to `app.js` and edit to be like the following.

```
module.exports = "<doctype!><html><body><h1>HTML from JS!</h1></body></html>";
```

We've just made our first module.

Now _require_ `app.js` in `server.js` and store the returned string in the variable `htmlString`.

`server.js` should now look like this: 

```
// server.js
var debug = require('debug')('server');
var express = require('express');
var htmlString = require('./app');
var app = express();

app.get('/', function (req, res) {
  res.setHeader('Content-Type', 'text/html');
  res.send(htmlString);
});

app.set('port', process.env.PORT || 3000);

app.listen(app.get('port'), function () {

  var env = app.get('env');
  var port = this.address().port;

  debug('Serving ' + app.get('env') + ' rentals on port %s', port);
});
```

Notice, we got rid of the `path` module and also explicitly set the `Content-Type` header.

Again, restart the server, refresh your browser and you should see `HTML from JS!`.

### Version Control

Let's commit our code to Git.

First, create a .gitignore file and add `/node_modules/*` to it so we keep our node_modules out of version control for the time being. We may re-evaluate this in the future.

Then:

```
git init
git add .
git commit -m "Initial commit returning HTML string from a JS module."
```

### Rendering React on the server

`npm install react node-jsx --save`

React is obviously the React.js library while Node-jsx is a convenience module that allows us to _require_ modules that contain JSX. When you require a .jsx module, the internal JSX will be run through the React-tools transformer and turned into plain Javascript.

Rename `app.js` to `app.jsx` and change it to export a react component.

```
// app.jsx
var React = require('react/addons');

module.exports = React.createClass({
  render: function(){
    return (
      <h1>Hello React!</h1>
    )
  }
});
```

Now, update `server.js` to require this component and render it to a string.

```
// server.js
var debug = require('debug')('server');
var express = require('express');
var React = require('react/addons');
require('node-jsx').install({extension:'.jsx'});

var server = express();

var App = React.createFactory(require('./app.jsx'));
var htmlString = React.renderToString(App());

server.get('/', function (req, res) {
  res.setHeader('Content-Type', 'text/html');
  res.send(htmlString);
});

server.set('port', process.env.PORT || 3000);

server.listen(server.get('port'), function () {

  var env = server.get('env');
  var port = this.address().port;

  debug('Serving ' + server.get('env') + ' rentals on port %s', port);
});
```

Restart the server and refresh your browser to see the new server-rendered component.

With a typical React _render_ you need to have an actual DOM to insert the component into and setup listeners for component events. On the server side we don't have any of that so we need to render to a plain HTML string to send to the browser.

If you inspect the HTML from the browser you'll notice _data-reactid_ and _data-react-checksum_ attributes on the H1 tag. When React runs in the browser (which we aren't doing yet) it will find and use these attributes to attach listeners instead of remaking what was sent by the server. React intelligently sets up the app state using what is already there so the user doesn't see any sort of flash or DOM rerendering.

### Adding html templates

You can have a React component render the entire HTML document including the _doctype_, _html_, _head_, etc., but we have a handful of static pages that don't make a ton of sense being components. It would be an easy switch if we change our minds on that. For now, we'll use a template for the root levels of HTML and inject our server-rendered React string into it. We're going to be using Swig templates which look a lot like moustache/handlebars. 

> I go back and forth on using Jade/Haml or a more HTML style template. I've been somewhat persuaded by the argument to reduce cognitive overhead on potential designers who already know HTML. It's easy to grok Swig as it keeps the basic tag structure. JSX also looks like HTML and it seems nice too keep things consistent. Swig is also incredibly performant and is way out in front compared to other js templates rendering speed.

Create an `html` directory at the root of the project and create an `app.html` file in it with the following.

	<!-- app.html -->
	
	<!doctype html>
	<html>
	<head>
	  <meta charset="utf-8">
	  <title>Rentals Node Test</title>
	</head>
	<body>
	  <h1>Rentals Node Test</h1>
	  {{ html | safe }}
	</body>
	</html>

We'll pass an `html` variable to this template and the `| safe` tells Swig not to escape the HTML in the variable.

Install swig adding it to package.json and update `server.js`

```
npm install swig --save
```

```
// server.js

require('node-jsx').install({extension:'.jsx'});

var debug = require('debug')('server');
var express = require('express');
var swig = require('swig');
var React = require('react/addons');

var server = express();

//
// TEMPLATE SETUP
//

// assign the swig engine to .html files
server.engine('html', swig.renderFile);

// set .html as the default extension
server.set('view engine', 'html');
server.set('views', __dirname + '/html');

// Swig will cache templates for you, but you can disable
// that and use Express's caching instead, if you like:
server.set('view cache', false);
// To disable Swig's cache, do the following:
// swig.setDefaults({ cache: false });
// NOTE: You should always cache templates in a production environment.
// Don't leave both of these to `false` in production!

var App = React.createFactory(require('./app/temp.jsx'));
var htmlString = React.renderToString(App());

server.get('/', function (req, res) {
  res.render('app', { html: htmlString });
});

server.set('port', process.env.PORT || 3000);

server.listen(server.get('port'), function () {

  var env = server.get('env');
  var port = this.address().port;

  debug('Serving ' + server.get('env') + ' rentals on port %s', port);
});
```

When Express processes `res.render('app', { html: htmlString });` it knows to look in the html directory for an `app.html` file and we're passing `htmlString` into the template as the variable `html`.

Restart the server and you should see the output of the new template when you refresh.

### React Router

It's time to add the incredible React Router.

The project has fantastic [documentation](https://github.com/rackt/react-router) including a [guide](https://github.com/rackt/react-router/blob/master/docs/guides/overview.md) on how it works

React Router isn't totally ready to go on React 0.13 which is what was installed earlier. Uninstall that first and let React Router install the version of React that it wants.

```
npm uninstall react --save
npm install react-router --save
```

You should see react version 0.12.x in package.json along with react-router.

##### App structure

Rename `temp.jsx` to `app.jsx` and edit it to look like the following.

```
var React = require('react/addons');
var Router = require('react-router');

var DefaultRoute = Router.DefaultRoute;
var Link = Router.Link;
var Route = Router.Route;
var RouteHandler = Router.RouteHandler;

var App = React.createClass({
  render: function(){
    return (
      <div>
        <a href='/'>
          <h1>The App component</h1>
        </a>
        <RouteHandler />
      </div>
    );
  }
});

var One = React.createClass({
  render: function(){
    return (
      <div>
        <p>This is component One 
        </p>
        <a href='/two'>
          Go to component Two
        </a>
      </div>
    )
  }
});

var Two = React.createClass({
  render: function(){
    return (
      <div>
        <p>This is component Two</p>
        <a href='/one'>
          Go to component One
        </a>
      </div>
    )
  }
});

var routes = (
  <Route name="app" path="/" handler={App}>
    <Route name="one" handler={One}/>
    <Route name="two" handler={Two}/>
    <DefaultRoute handler={One}/>
  </Route>
);

exports.route = function(url, callback){
  Router.run(routes, url, function (Handler) {
    var content = React.renderToString(<Handler/>);
    callback(content);
  });
};
```

We have three React components. App, One, and Two. 

Our `routes` variable holds the routing hierarchy and the App component contains the very important `<RouteHandler />`. This is used by React Router to pass the correct component at render time. We also export a function that _runs_ the router with a given URL and calls a callback with the rendered HTML string.

Here is our updated `server.js` that uses the router.

```
require('node-jsx').install({extension:'.jsx'});

var debug = require('debug')('server');
var express = require('express');
var swig = require('swig');
var React = require('react/addons');
var Router = require('react-router');

var app = require('./app/app.jsx');

var server = express();

//
// TEMPLATE SETUP
//

// assign the swig engine to .html files
server.engine('html', swig.renderFile);

// set .html as the default extension
server.set('view engine', 'html');
server.set('views', __dirname + '/html');

// Swig will cache templates for you, but you can disable
// that and use Express's caching instead, if you like:
server.set('view cache', false);
// To disable Swig's cache, do the following:
// swig.setDefaults({ cache: false });
// NOTE: You should always cache templates in a production environment.
// Don't leave both of these to `false` in production!

// var App = React.createFactory(require('./app/temp.jsx'));
// var htmlString = React.renderToString(App());

server.get('*', function (req, res) {
  app.route(req.url, function(content){
    res.render('app', { reactContent: content });
  });
});

server.set('port', process.env.PORT || 3000);

server.listen(server.get('port'), function () {

  var env = server.get('env');
  var port = this.address().port;

  debug('Serving ' + server.get('env') + ' rentals on port %s', port);
});
```

Restart server/refresh to see the routing based on url.

TODO:

* add webpack to compile all the js
* add asset compilation for images and styles
* setup a dev watcher so we don't have to keep restarting the server
* lots more
