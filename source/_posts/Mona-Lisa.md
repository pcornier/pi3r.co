---
title: Mona Lisa
date: 2017-11-12 11:04:12
tags:
---

Let's paint Mona Lisa with random dots!

<!-- more -->

<p data-height="499" data-theme-id="light" data-slug-hash="YEmBmP" data-default-tab="result" data-user="pcornier" data-pen-title="MonaLisa" class="codepen">See the Pen <a href="https://codepen.io/pcornier/pen/YEmBmP/">MonaLisa</a> by Pierre (<a href="https://codepen.io/pcornier">@pcornier</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

```javascript
let target;
let canvas;
let best = Infinity;
let graphic;
let palette;

function preload() {
  let url = 'https://upload.wikimedia.org/wikipedia/commons/thumb/e/ec/Mona_Lisa%2C_by_Leonardo_da_Vinci%2C_from_C2RMF_retouched.jpg/260px-Mona_Lisa%2C_by_Leonardo_da_Vinci%2C_from_C2RMF_retouched.jpg';
  target = loadImage(url);
}

function setup() {
  //randomSeed(971846);
	canvas = createCanvas(target.width, target.height);
  graphic = createGraphics(target.width, target.height);
  loadPixels();
  target.loadPixels();
  let colorThief = new ColorThief();
  palette = colorThief.getPalette(target.canvas, 256);
}

function evaluate() {
  let err = 0;
  graphic.loadPixels();
  for (var i = 0; i < target.pixels.length; i++) {
    let diff = target.pixels[i] - graphic.pixels[i];
    err += diff * diff;
  }
  return err;
}

function draw() {
  graphic.canvas.getContext('2d').drawImage(canvas.canvas, 0, 0);
  graphic.loadPixels();
  let x = random(target.width);
  let y = random(target.height);
  let r = random(50);
  let p = random(palette.length)|0;
  let c = color(palette[p][0], palette[p][1], palette[p][2], random(255));
  graphic.noStroke();
  graphic.fill(c);
  graphic.ellipse(x, y, r);
  let score = evaluate();
  if (score < best) {
    best = score;
    image(graphic, 0, 0);
  }
}
```