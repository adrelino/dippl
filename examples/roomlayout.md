---
layout: example
title: Room Layout Inference
description: Finding latent structure that renders to a target image.
custom_js:
  ../assets/js/draw.js
custom_css:
  ../assets/css/draw.css
---

Based on [Computer Vision](vision.html) example.

Drawing a rectangle:

~~~~
var myDraw = Draw(200, 200, true);
var x1 = randomInteger(200);
var y1 = randomInteger(200);
var x2 = randomInteger(200);
var y2 = randomInteger(200);
myDraw.rectangle(x1, y1, x2, y2);
~~~~

Drawing many rectangles:

~~~~
var myDraw = Draw(200, 200, true);

var makeRectangles = function(n, xs){
  var x1 = randomInteger(200);
  var y1 = randomInteger(200);
  var x2 = randomInteger(200);
  var y2 = randomInteger(200);
  var xs2 = xs.concat([[x1, y1, x2, y2]]);
  return (n==0) ? xs : makeRectangles(n-1, xs2);
}

var drawRectangles = function(lines){
  var line = lines[0];
  myDraw.rectangle(line[0], line[1], line[2], line[3]);
  if (lines.length > 1) {
    drawRectangles(lines.slice(1));
  }
}

var lines = makeRectangles(4, []);

drawRectangles(lines);
~~~~

Computing the pixel-by-pixel distance between two images:

~~~~
///fold:
var myDraw = Draw(200, 200, true);

var makeRectangles = function(n, xs){
  var x1 = randomInteger(200);
  var y1 = randomInteger(200);
  var x2 = randomInteger(200);
  var y2 = randomInteger(200);
  var xs2 = xs.concat([[x1, y1, x2, y2]]);
  return (n==0) ? xs : makeRectangles(n-1, xs2);
}

var drawRectangles = function(lines){
  var line = lines[0];
  myDraw.rectangle(line[0], line[1], line[2], line[3]);
  if (lines.length > 1) {
    drawRectangles(lines.slice(1));
  }
}

var lines = makeRectangles(4, []);

drawRectangles(lines);
///
var targetImage = Draw(200, 200, true);
loadImage(targetImage, "../assets/img/boxes.png");

myDraw.distance(targetImage);
~~~~

Inferring rectangles that match the target image:

~~~~
var targetImage = Draw(200, 200, false);
loadImage(targetImage, "../assets/img/boxes.png")
var makeRectangles = function(n, lines, prevScore){
  var x1 = randomInteger(200);
  var y1 = randomInteger(200);
  var x2 = randomInteger(200);
  var y2 = randomInteger(200);
  //var xs2 = xs.concat([[x1, y1, x2, y2]]);
  //return (n==0) ? xs : makeRectangles(n-1, xs2);
  var newLines = lines.concat([[x1, y1, x2, y2]]);
  // Compute image from set of lines
  var generatedImage = Draw(200, 200, false);
  drawRectangles(generatedImage, newLines);
  // Factor prefers images that are close to target image
  var newScore = -targetImage.distance(generatedImage)/1000; // Increase to 10000 to see more diverse samples
  factor(newScore - prevScore);
  generatedImage.destroy();
  // Generate remaining lines (unless done)
  return (n==1) ? newLines : makeRectangles(n-1, newLines, newScore);
}

var drawRectangles = function(drawObj, lines){
  var line = lines[0];
  drawObj.rectangle(line[0], line[1], line[2], line[3]);
  if (lines.length > 1) {
    drawRectangles(drawObj, lines.slice(1));
  }
}

Infer({
  //method: 'SMC', particles: 100,
  method: 'MCMC', samples: 2500,
  model() {
    var lines = makeRectangles(4, [], 0);
    var finalGeneratedImage = Draw(200, 200, true);
	drawRectangles(finalGeneratedImage, lines);
   }
})

'done'
~~~~