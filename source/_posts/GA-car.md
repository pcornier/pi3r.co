---
title: A Genetic car driver
date: 2017-12-09 10:54:38
tags:
- Genetic algorithm
- p5js
---

A genetic algorithm that drives a car with p5js.

<!-- more -->

<p data-height="334" data-theme-id="light" data-slug-hash="vxxmjW" data-default-tab="result" data-user="pcornier" data-pen-title="Parking with Genetic Algorithm (WIP)" class="codepen">See the Pen <a href="https://codepen.io/pcornier/pen/vxxmjW/">Parking with Genetic Algorithm (WIP)</a> by Pierre (<a href="https://codepen.io/pcornier">@pcornier</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

```javascript

let car;
let target;
let preview = [];
let speed = 0.01;
let error = Infinity;
let gen = 0;
let parent;

function setup() {
  createCanvas(320, 240);

  car = {
    w: 10, // half width
    h: 20, // half height
    oa: createVector(160, 100),
    ob: createVector(160, 140),
    a: createVector(0, 0),
    b: createVector(0, 0),
    r: 0,
    draw: function() {
      push();
      let p = p5.Vector.add(this.a, this.b).div(2);
      translate(p.x, p.y);
      rotate(p5.Vector.sub(this.a, this.b).heading() + HALF_PI);
      rect(-this.w, -this.h, this.w * 2, this.h * 2);
      pop();
    }
  };

  target = {
    a: createVector(40, 80),
    b: createVector(40, 40),
    draw: function() {
      push();
      translate(this.a.x, this.a.y + car.h);
      //rect(-car.w, -car.h, car.w * 2, car.h * 2);
      pop();
      fill(255,0,0);
      ellipse(this.a.x, this.a.y, 5);
      fill(255);
      ellipse(this.b.x, this.b.y, 5);
    }
  }

}

function generateSequence() {
  let seq = [];
  for (let i = 0; i < 5; i++) {
    let frames = random(200)|0;
    seq.push({
      angle: random(2*PI),
      frames: 0,
      total: frames
    });
  }
  return seq;
}

function generateChild(p) {
  let cp = random(p.length)|0;
  let child = JSON.parse(JSON.stringify(p));
  let mutations = 1 + random(2);
  for (let i = 0; i < mutations; i++) {
    let mut = random(child.length)|0;
    child[mut].angle = random(2*PI);
    child[mut].total = random(200)|0;
  }
  return child;
}

// t = 0 to 1, b = start value, c = end - start, d = duration (steps)
function ease(t, b, c, d) {
  t /= d/2;
  if (t < 1) return c/2*t*t*t+b;
  t -= 2;
  return c/2*(t*t*t+2)+b;
}


function evaluate(seq) {
  let a = car.oa.copy();
  let b = car.ob.copy();
  let t = 0;
  for (let i = 0; i < seq.length; i++) {
    for (let f = 0; f <= seq[i].total; f++) {
      let p = ease(f, 0, seq[i].total, seq[i].total);
      let cv = p5.Vector.sub(b, a).normalize();
      let d1 = createVector(1, 0).rotate(seq[i].angle).mult(p * speed);
      a.add(d1);
      let l = b.dist(a) - car.h * 2;
      let d2 = p5.Vector.sub(a, b).normalize().mult(l);
      b.add(d2);
    }
  }
  return a.dist(target.a) + b.dist(target.b);
}


function draw() {
  if (preview.length) {
    let c = preview[0];
    if (c.frames < c.total) {

      // calculate new position

      let p = ease(c.frames, 0, c.total, c.total);
      let cv = p5.Vector.sub(car.b, car.a).normalize();
      let d1 = createVector(1, 0).rotate(c.angle).mult(p * speed);
      car.a.add(d1);

      let l = car.b.dist(car.a) - car.h * 2;
      let d2 = p5.Vector.sub(car.a, car.b).normalize().mult(l);
      car.b.add(d2);
      c.frames++;

      // draw

      background(255);

      stroke(color(0, 0, 0));
      target.draw();
      car.draw();

      stroke(color(0, 255, 0));
      line(car.a.x, car.a.y, car.a.x+d1.x*20, car.a.y+d1.y*20);

      fill(color(255, 0, 0));
      noStroke();
      ellipse(car.a.x, car.a.y, 5);
      ellipse(car.b.x, car.b.y, 5);

      noFill();
      stroke(0);
      rect(0, 0, 319, 239);

      textSize(10);
      text('g: ' + gen + ' e: ' + error, 5, 15);

    }
    else {
      preview.shift();
    }
  }
  else {

    if (!parent) {
      parent = generateSequence();
    }

    for (let g = 0; g < 10; g++) {

      let p = parent;

      min = error;
      for (let i = 0; i < 50; i++) {
        let c = generateChild(p);
        let e = evaluate(c);
        if (e < min) {
          parent = c;
          min = e;
        }
      }

      error = min;
      gen++;
    }

    preview = JSON.parse(JSON.stringify(parent));

    car.a = car.oa.copy();
    car.b = car.ob.copy();

  }
}
```