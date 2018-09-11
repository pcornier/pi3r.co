---
title: A basic GA with a grid
date: 2017-12-09 10:52:11
tags:
---

This is a basic genetic algorithm that converges into a simple grid.
Please note that p5.js is required to run this example.

<!-- more -->

```javascript
let grid = [];
let target = [];let grid = [];
let target = [];
let CX = 200;
let CY = 200;

function setup() {
  createCanvas(CX, CY);
  for (let y = 0; y < 10; y++) {
    let row1 = [];
    let row2 = [];
    for (let x = 0; x < 10; x++) {
      row1.push([x * 20, y * 20]);
      row2.push([random(0, 200), random(0, 100)]);
    }
    target.push(row1)
    grid.push(row2);
  }
}

function evaluate(g) {
  let e = 0;
  let cnt = 0;
  for (let y = 0; y < g.length; y++) {
    for (let x = 0; x < g[y].length; x++) {
      let lx = target[y][x][0] - g[y][x][0];
      let ly = target[y][x][1] - g[y][x][1];
      e += lx * lx + ly * ly;
    }
  }
  return e;
}

function draw() {

  let w = evaluate(grid);
  console.log(w);
  for (let i = 0; i < 10; i++) {
    let px = random(0, grid[0].length)|0;
    let py = random(0, grid.length)|0;
    let clone = JSON.parse(JSON.stringify(grid));
    let delta = Math.sqrt(w/100);
    clone[py][px][0] = Math.max(0, Math.min(CX, clone[py][px][0] + random(-delta, delta)));
    clone[py][px][1] = Math.max(0, Math.min(CY, clone[py][px][1] + random(-delta, delta)));
    let e = evaluate(clone);
    if (e < w) {
      grid = JSON.parse(JSON.stringify(clone));
      w = e;
    }
  }


  background(255);
  for (let y = 0; y < grid.length; y++) {
    for (let x = 0; x < grid[y].length; x++) {
      if (grid[y][x+1])
        line(grid[y][x][0], grid[y][x][1], grid[y][x+1][0], grid[y][x+1][1]);
      if (grid[y+1])
        line(grid[y][x][0], grid[y][x][1], grid[y+1][x][0], grid[y+1][x][1]);
    }
  }
}
let CX = 200;
let CY = 200;

function setup() {
  createCanvas(CX, CY);
  for (let y = 0; y < 10; y++) {
    let row1 = [];
    let row2 = [];
    for (let x = 0; x < 10; x++) {
      row1.push([x * 20, y * 20]);
      row2.push([random(0, 200), random(0, 100)]);
    }
    target.push(row1)
    grid.push(row2);
  }
}

function evaluate(g) {
  let e = 0;
  let cnt = 0;
  for (let y = 0; y < g.length; y++) {
    for (let x = 0; x < g[y].length; x++) {
      let lx = target[y][x][0] - g[y][x][0];
      let ly = target[y][x][1] - g[y][x][1];
      e += lx * lx + ly * ly;
    }
  }
  return e;
}

function draw() {

  let w = evaluate(grid);
  console.log(w);
  for (let i = 0; i < 10; i++) {
    let px = random(0, grid[0].length)|0;
    let py = random(0, grid.length)|0;
    let clone = JSON.parse(JSON.stringify(grid));
    let delta = Math.sqrt(w/100);
    clone[py][px][0] = Math.max(0, Math.min(CX, clone[py][px][0] + random(-delta, delta)));
    clone[py][px][1] = Math.max(0, Math.min(CY, clone[py][px][1] + random(-delta, delta)));
    let e = evaluate(clone);
    if (e < w) {
      grid = JSON.parse(JSON.stringify(clone));
      w = e;
    }
  }


  background(255);
  for (let y = 0; y < grid.length; y++) {
    for (let x = 0; x < grid[y].length; x++) {
      if (grid[y][x+1])
        line(grid[y][x][0], grid[y][x][1], grid[y][x+1][0], grid[y][x+1][1]);
      if (grid[y+1])
        line(grid[y][x][0], grid[y][x][1], grid[y+1][x][0], grid[y+1][x][1]);
    }
  }
}
```