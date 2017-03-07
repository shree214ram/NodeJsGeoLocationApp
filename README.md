
Node JS/Socket IO  Real-Time Geolocation with Web Sockets
Installing node

First, you’ll need to install node.js. You can get pre-compiled Node.js binaries for several platforms from the download section of the official website: http://nodejs.org/download.
After the installation is complete you will get access to the node package manager (npm), with the help of which we will install all needed modules for this tutorial. We will use socket.io and node-static, which will serve all the client side files with ease. Go to the directory of your app and run this command in your terminal or command line:

npm install socket.io node-static

Tip: I advice you to install a utility like nodemon that will keep an eye on your files and you won’t need to restart your server after every change:

npm install nodemon -g

“-g” means that it will be installed globally and accessible from every node repo.

The HTML

Let’s first create an “index.html” in our public directory.

<!doctype html>
<html>

	<head>
		<meta charset="utf-8">
		<meta name="author" content="Dmitri Voronianski">
		<title>Real-Time Geolocation with Web Sockets</title>
		<link href='http://fonts.googleapis.com/css?family=Lato:300,400' rel='stylesheet' type='text/css'>
		<link rel="stylesheet" href="./css/styles.css">
		<link rel="stylesheet" href="http://cdn.leafletjs.com/leaflet-0.4/leaflet.css" />

		<!--[if lt IE 9]>
			<script src="//html5shim.googlecode.com/svn/trunk/html5.js"></script>
		<![endif]-->
	</head>

	<body>
		<div class="wrapper">
		  <header>
		  	<h1>Real-Time Geolocation Service with Node.js</h1>
		  	<div class="description">Using HTML5 Geolocation API and Web Sockets to show connected locations.</div>
		  </header>

		  <div class="app">
		  	<div class="loading"></div>
		  	<div id="infobox" class="infobox"></div>
		  	<div id="map">To get this app to work you need to share your geolocation.</div>
		  </div>
		</div>

		<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.8.0/jquery.min.js"></script>
		<script src="./js/lib/leaflet.js"></script>
		<script src="/socket.io/socket.io.js"></script>
		<script src="./js/application.js"></script>
	</body>

</html>

As you can see it’s pretty simple. For rendering our map on the page we will use an incredible open-source JavaScript library for interactive maps – Leaflet.js. It’s free and highly customizable. The API documentation is available on the website. Our styles.css file is inside the “./public/css/” folder. It will include some simple styles for the app. The leaflet.css in the same folder contains the styles for the map.
Server side

Now we are ready to start with the back-end of our app. Let’s take a look at “server.js”:

// including libraries
var http = require('http');
var static = require('node-static');
var app = http.createServer(handler);
var io = require('socket.io').listen(app);

// define port
var port = 8080;

// make html, js & css files accessible
var files = new static.Server('./public');

// serve files on request
function handler(request, response) {
	request.addListener('end', function() {
		files.serve(request, response);
	});
}

// listen for incoming connections from client
io.sockets.on('connection', function (socket) {

  // start listening for coords
  socket.on('send:coords', function (data) {

  	// broadcast your coordinates to everyone except you
  	socket.broadcast.emit('load:coords', data);
  });
});

// starts app on specified port
app.listen(port);
console.log('Your server goes on localhost:' + port);

The code is not complicated at all; everything that it does is serving files and listening to the data from the client. Now we can start our app from the terminal or command line and take a look:

node server.js

Or, if you have followed my advice and used nodemon, write this:

nodemon server.js

Now go to localhost:8080 in your browser (you can change the port to whatever you like). Everything will be static because our main JavaScript function is not ready, yet.

The Client Side

It’s time to open the “./public/js/application.js” file and to write a couple of functions (we’ll be using jQuery):

$(function() {
	// generate unique user id
	var userId = Math.random().toString(16).substring(2,15);
	var socket = io.connect("/");
	var map;

	var info = $("#infobox");
	var doc = $(document);

	// custom marker's icon styles
	var tinyIcon = L.Icon.extend({
		options: {
			shadowUrl: "../assets/marker-shadow.png",
			iconSize: [25, 39],
			iconAnchor:   [12, 36],
			shadowSize: [41, 41],
			shadowAnchor: [12, 38],
			popupAnchor: [0, -30]
		}
	});
	var redIcon = new tinyIcon({ iconUrl: "../assets/marker-red.png" });
	var yellowIcon = new tinyIcon({ iconUrl: "../assets/marker-yellow.png" });

	var sentData = {}

	var connects = {};
	var markers = {};
	var active = false;

	socket.on("load:coords", function(data) {
		// remember users id to show marker only once
		if (!(data.id in connects)) {
			setMarker(data);
		}

		connects[data.id] = data;
		connects[data.id].updated = $.now(); // shorthand for (new Date).getTime()
	});

	// check whether browser supports geolocation api
	if (navigator.geolocation) {
		navigator.geolocation.getCurrentPosition(positionSuccess, positionError, { enableHighAccuracy: true });
	} else {
		$(".map").text("Your browser is out of fashion, there\'s no geolocation!");
	}

	function positionSuccess(position) {
		var lat = position.coords.latitude;
		var lng = position.coords.longitude;
		var acr = position.coords.accuracy;

		// mark user's position
		var userMarker = L.marker([lat, lng], {
			icon: redIcon
		});

		// load leaflet map
		map = L.map("map");

		// leaflet API key tiler
		L.tileLayer("http://{s}.tile.cloudmade.com/BC9A493B41014CAABB98F0471D759707/997/256/{z}/{x}/{y}.png", { maxZoom: 18, detectRetina: true }).addTo(map);
		
		// set map bounds
		map.fitWorld();
		userMarker.addTo(map);
		userMarker.bindPopup("

You are there! Your ID is " + userId + "
").openPopup();

		// send coords on when user is active
		doc.on("mousemove", function() {
			active = true; 

			sentData = {
				id: userId,
				active: active,
				coords: [{
					lat: lat,
					lng: lng,
					acr: acr
				}]
			}
			socket.emit("send:coords", sentData);
		});
	}

	doc.bind("mouseup mouseleave", function() {
		active = false;
	});

	// showing markers for connections
	function setMarker(data) {
		for (i = 0; i < data.coords.length; i++) {
			var marker = L.marker([data.coords[i].lat, data.coords[i].lng], { icon: yellowIcon }).addTo(map);
			marker.bindPopup("

One more external user is here!
");
			markers[data.id] = marker;
		}
	}

	// handle geolocation api errors
	function positionError(error) {
		var errors = {
			1: "Authorization fails", // permission denied
			2: "Can\'t detect your location", //position unavailable
			3: "Connection timeout" // timeout
		};
		showError("Error:" + errors[error.code]);
	}

	function showError(msg) {
		info.addClass("error").text(msg);
	}

	// delete inactive users every 15 sec
	setInterval(function() {
		for (ident in connects){
			if ($.now() - connects[ident].updated > 15000) {
				delete connects[ident];
				map.removeLayer(markers[ident]);
			}
        }
    }, 15000);
});

Magic happens when we use socket.emit to send a message to our node web server on every mouse move. It means that our user is active on the page. We also receive the data from the server with socket.on and after getting initialize markers on the map. The main things that we need for the markers are the latitude and longitude which we receive from the browser. If the user is inactive for more then 15 seconds we remove their marker from our map. If the user’s browser doesn’t support the Geolocation API we’ll show a message that the browser is out-of-date. You can read more about the HTML5 Geolocation API here: Geolocation – Dive Into HTML5.
