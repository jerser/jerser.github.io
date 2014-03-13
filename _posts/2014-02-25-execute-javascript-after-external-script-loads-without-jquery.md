---
layout: post
title: "Execute Javascript after external script successfully loads"
description: ""
tags:
  - javascript
  - jquery
---
{% include JB/setup %}

a.k.a. [jQuery.getScript()](https://api.jquery.com/jQuery.getScript/) without jQuery.

Useful for when you need to use external dependencies in your Javascript but you cannot
control when your external dependency is loaded. E.g.: A bookmarklet that injects code
and loads an external librarie as dependency.

The code below is more or less exactly the way jQuery implements getScript().
It does an [XMLHTTPRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) and injects the received text in a script tag in the parent
document, after which it executes your code.

{% highlight javascript %}
(function(){
  var xmlhttp = new XMLHttpRequest();

  // This is the dependency we want to load, socket.io in this case
  xmlhttp.open('GET', 'http://localhost:8080/socketio.min.js');

  // It's important here to both check for the correct
  // HTTP Return Code as the readyState variable
  xmlhttp.onreadystatechange = function() {
    if ((xmlhttp.status == 200) && (xmlhttp.readyState == 4)) {
      // Create a new script element, add it as a child to the head
      // and cleanup the variable
      var script = document.createElement("script");
      script.text = xmlhttp.responseText;
      document.head.appendChild(script).
                    parentNode.removeChild(script);

      // This is the actual code that depends on the external library
      var socket = io.connect('http://localhost:8080');
      socket.on('connect', function () {
        // do something
      });
    }
  }
  xmlhttp.send();
})();

{% endhighlight %}

Just for demonstration purposes, you should obviously extend it to respond to errors
if applicable and you should not use it to load just any external script, only for trusted
sources.

If the external dependency being loaded is on a different domain than the webpage
where you are using it the destination server will need to set the appropriate
[CORS](https://en.wikipedia.org/wiki/Cross-Origin_Resource_Sharing) headers.
