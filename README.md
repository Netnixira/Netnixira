/**********************************************************************
                          Wii Remote emulator
                     By Mark Wilton-Jones 24/7/2007
                              Version 1.0
***********************************************************************

Please see http://www.howtocreate.co.uk/jslibs/ for details and a demo of this script
Please see http://www.howtocreate.co.uk/jslibs/termsOfUse.html for terms of use

Attempts to emulate the Wii Remote API and JavaScript events using a mouse and keyboard.
This should allow many games or special pages written for the Internet Channel on Nintendo
Wii to work on normal desktop browsers.

To use:

The script can operate either as a script that can be included in the Web page by its
author (preferred), or as a User JavaScript that a user can install and run on desired
pages.

To include it in a Web page, inbetween the <head> tags, put:

	<script src="PATH TO SCRIPT/emulatewiimote.js" type="text/javascript"></script>

To use it as a user JavaScript, install it as normal, and specify the pages where it should
run, using the UserScript block below - this script can cause problems with keyboard and
mouse on pages that do not try to use the Wii Remote, so it is important to include it only
on the exact pages where it is needed.
________________________________________________________________________________*/

// ==UserScript==
// @include http://devfiles.myopera.com/articles/149/wiicanvasgame.html
// @include http://example.com/someotherpage.html
// ==/UserScript==

(function () {

//if this seems to be a real Wii, there is no need to run the script
if( window.opera && opera.wiiremote ) { return; }

var rotation = 0, distance = 20, toggle = false, pageX = 0, pageY = 0, midScreenX = -1, midScreenY = -1, validdata = 2, canv, ctx, mousewheelonce, lastkey;

//bug sniffers for bugs that cannot be detected without it (sniffers are evil, so this tries to be as precise as possible)
var reversedwheel = window.opera && opera.version && parseFloat( opera.version() ) < 9.2;
var whatwidth = ( document.compatMode != 'CSS1Compat' || ( window.opera && opera.version && parseFloat( opera.version() ) < 9.5 ) ) ? 'body' : 'documentElement';
var brokenmouseover = window.opera && opera.version;
var opacitycrash = !window.devicePixelRatio; //Safari 2
//Opera moveTo bug can be avoided without needing to be detected
//Firefox keypress bugs can be detected without a sniffer
//Safari mousewheel bugs can be detected without a sniffer

//read key events, work out if they are a key that needs to be remapped
function swapEvents(e) {
	var newCode, evObj, curCode = e.keyCode;
	//workaround for Firefox forgetting to provide a keyCode for keypress events
	if( e.type == 'keydown' ) { lastkey = curCode;
	} else if( e.type == 'keypress' && !curCode ) { curCode = lastkey; }
	if( curCode == 77 || curCode == 109 ) { newCode = 170; //key M (minus) upper/lower becomes button -
	} else if( curCode == 66 || curCode == 98 ) { newCode = 171; //key B upper/lower becomes button 1
	} else if( curCode == 49 ) { newCode = 172; //key 1 becomes button 1
	} else if( curCode == 50 ) { newCode = 173; //key 2 becomes button 2
	} else if( curCode == 80 || curCode == 112 ) { newCode = 174; //key P (plus) upper/lower becomes button +
	} else if( curCode == 38 ) { newCode = 175; //up arrow becomes up button
	} else if( curCode == 40 ) { newCode = 176; //down arrow becomes down button
	} else if( curCode == 39 ) { newCode = 177; //right arrow becomes right button
	} else if( curCode == 37 ) { newCode = 178; //left arrow becomes left button
	} else if( e.type == 'keyup' && curCode == 84 ) {
		//key T lower becomes toggle (no need for upper 116, and upper conflicts with F5 - not useful)
		e.stopPropagation();
		toggle = !toggle;
		updateCanv();
		return;
	} else { return; }
	//key is being remapped, prevent the page from seeing it
	e.stopPropagation();
	//fire replacement event and cancel current if replacement is cancelled
	if( window.KeyEvent ) {
	  evObj = document.createEvent('KeyEvents');
	  evObj.initKeyEvent( e.type, e.bubbles, e.cancelable, window, false, false, false, false, newCode, 0 );
	} else {
		evObj = document.createEvent('UIEvents');
		evObj.initUIEvent( e.type, e.bubbles, e.cancelable, window, e.detail );
		evObj.keyCode = newCode;
	}
	if( !e.target.dispatchEvent(evObj) ) {
		e.preventDefault();
	}
}

//read mousewheel, and change rotation/distance as needed
function mousescroll(e) {
	var wheeldir = e.wheelDelta;
	//just in case the browser supports both mousewheel event types, remove the listener for the other one
	//it may still fire once, but there is nothing I can do about that
	if( !mousewheelonce ) {
		if( e.type == 'mousewheel' ) {
			window.removeEventListener('DOMMouseScroll',arguments.callee,false);
		} else {
			window.removeEventListener('mousewheel',arguments.callee,false);
		}
		mousewheelonce = true;
	}
	//get the intended direction
	if( !wheeldir ) { wheeldir = -1 * e.detail;
	} else if( reversedwheel ) { wheeldir = -1 * wheeldir; }
	//somehow, Safari thinks the wheel can turn without it having moved
	//since there is no way to know which way it turned, small rotations like this have to be ignored
	if( !wheeldir ) { return; }
	//change the setting, and impose limits
	if( toggle ) {
		distance += ( wheeldir < 0 ) ? 1 : -1;
		if( distance < 0 ) { distance = 0;
		} else if( distance > 25 ) { distance = 25; }
	} else {
		rotation += ( wheeldir > 0 ) ? 1 : -1;
		if( rotation < -39 ) { rotation = 40;
		} else if( rotation > 40 ) { rotation = -39; }
	}
	//cancel the events
	e.preventDefault();
	e.stopPropagation();
	//update the status icon
	updateCanv();
}

//update the status icon
function updateCanv() {
	if( !ctx ) { return; }
	var oHeight = ( ( 25 - distance ) / 25 ) * 40;
	var comproll = ( rotation / 40 ) * Math.PI;
	ctx.clearRect(0,0,40,40);
	if( !toggle ) {
		//distance when toggle is set to roll
		ctx.fillStyle = '#aaf';
		ctx.fillRect( 0, 40 - oHeight, 40, oHeight );
	}
	//red crosshair
	ctx.strokeStyle = 'red';
	ctx.beginPath();
	ctx.moveTo( 20, 0 );
	ctx.lineTo( 20, 40 );
	//avoid Opera 9-9.2x bug (path never closes) by having only one moveTo per path
	ctx.closePath();
	ctx.stroke();
	ctx.beginPath();
	ctx.moveTo( 0, 20 );
	ctx.lineTo( 40, 20 );
	ctx.closePath();
	ctx.stroke();
	//rotation
	ctx.fillStyle = toggle ? '#7d7' : '#070';
	ctx.beginPath();
	ctx.arc( 20, 20, 20, rotation/(4*Math.PI), Math.PI + ( rotation/(4*Math.PI) ), false );
	ctx.closePath();
	ctx.fill();
	ctx.fillStyle = ctx.strokeStyle = toggle ? '#bbb' : 'black';
	ctx.beginPath();
	//for some reason, a normal 1px line goes missing at certain rotations, so use a triangle
	ctx.moveTo( 20 - Math.cos(comproll), 20 - Math.sin(comproll) );
	ctx.lineTo( 20 + 20 * Math.sin(comproll), 20 - 20 * Math.cos(comproll) );
	ctx.lineTo( 20 + Math.cos(comproll), 20 + Math.sin(comproll) );
	ctx.closePath();
	ctx.fill();
	ctx.stroke();
	if( toggle ) {
		//distance when toggle is set to distance
		ctx.fillStyle = 'rgba(0,0,255,0.6)';
		ctx.fillRect( 0, 40 - oHeight, 40, oHeight );
	}
}

//create and prepare the status icon
function prepcanvas() {
	//if the browser supports DOMContentLoaded, it will appear earlier
	//remove the listeners to prevent load causing duplication
	window.removeEventListener('DOMContentLoaded',arguments.callee,false);
	window.removeEventListener('load',arguments.callee,false);
	canv = document.createElement('canvas');
	canv.setAttribute('height','40');
	canv.setAttribute('width','40');
	document.documentElement.appendChild(canv);
	canv.sidestate = true;
	//prevent safari 2 crashing with transparent canvas
	canv.cssTextRight = 'position: fixed; top: 0; right: 0; z-index: 999999; opacity: 0.4; '+(opacitycrash?'-khtml-opacity: 1; ':'')+'border: solid #000; border-width: 0 0 1px 1px';
	canv.cssTextLeft = 'position: fixed; top: 0; left: 0; z-index: 999999; opacity: 0.4; '+(opacitycrash?'-khtml-opacity: 1; ':'')+'border: solid #000; border-width: 0 1px 1px 0';
	canv.style.cssText = canv.cssTextRight;
	canv.onclick = function () {
		this.sidestate = !this.sidestate;
		this.style.cssText = this.sidestate ? this.cssTextRight : this.cssTextLeft;
	};
	if( canv.getContext ) { ctx = canv.getContext('2d'); }
	updateCanv();
}

//create the simulated wiiremote object
if( !window.opera ) {
	window.opera = {};
}
opera.wiiremote = {};
opera.wiiremote.update = function (n) {
	//invalid indexes return an empty object
	if( parseInt(n) != n || isNaN(n) || n < 0 || n > 3 ) { return {}; }
	//indexes 1-3 return a disabled wiimote
	if( n ) { return { isEnabled: 0 }; }
	//copy the stored data into a fake KpadStatus object, trying to maintain the normal sort ordering
	var datacopy = { isEnabled: 1, isDataValid: 1, isBrowsing: 1, dpdScreenX: pageX, dpdScreenY: pageY, dpdX: midScreenX,
	  dpdY: midScreenY, hold: 0, dpdRollX: null, dpdRollY: null, dpdDistance: null, dpdValidity: validdata };
	var comproll = ( rotation / 40 ) * Math.PI;
	datacopy.dpdRollX = Math.cos(comproll);
	datacopy.dpdRollY = Math.sin(comproll);
	datacopy.dpdDistance = 0.55 + ( ( 3 - 0.55 ) * ( distance / 25 ) );
	if( !validdata ) {
		delete datacopy.dpdScreenX;
		delete datacopy.dpdScreenY;
	}
	//avoid precision errors (sometimes gives around 1e-16 instead of 0)
	if( Math.abs( datacopy.dpdRollX ) < 0.0001 ) { datacopy.dpdRollX = 0;
	} else if( Math.abs( datacopy.dpdRollY ) < 0.0001 ) { datacopy.dpdRollY = 0; }
	return datacopy;
};

//set up all the load, key, mousewheel and mouse event listeners
if( window.addEventListener ) {
	window.addEventListener('DOMContentLoaded',prepcanvas,false);
	window.addEventListener('load',prepcanvas,false);
	window.addEventListener('keydown',swapEvents,true);
	window.addEventListener('keyup',swapEvents,true);
	window.addEventListener('keypress',swapEvents,true);
	window.addEventListener('mousewheel',mousescroll,true);
	window.addEventListener('DOMMouseScroll',mousescroll,true);
	window.addEventListener('mousemove',function (e) {
		var oWidth = window.innerWidth, oHeight = window.innerHeight;
		pageX = e.pageX;
		pageY = e.pageY;
		midScreenX = ( pageX - ( oWidth / 2 ) ) / ( oWidth / 2 );
		midScreenY = ( pageY - ( oHeight / 2 ) ) / ( oHeight / 2 );
	},true);
	document.addEventListener('mouseout',function () { validdata = 0; },false);
	document.addEventListener('mouseover',function (e) {
		//mouseover fires for the body when moving over the edge of the chrome in Opera
		if( brokenmouseover && e.offsetX == 0 && e.offsetY == 0 && document.body && e.target == document.body ) {
			//chances are that this is over the chrome, but ...
			//there is one pixel (just one - the top left of the body) where this is not true
			//laboriously work out which one it is by checking if the mouse is against the broken edge of the viewport
			//this won't happen too often, so it doesn't matter for performance
			//it will get it wrong if the page is scrolled or styled to put that pixel under the very edge of the chrome, but
			//that is not too bad, and I really don't care
			if( e.clientX < 1 || e.clientX >= document[whatwidth].clientWidth || e.clientY < 1 || e.clientY >= document[whatwidth].clientHeight ) { return; }
		}
		validdata = 2;
	},false);
}

})();
