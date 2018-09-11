---
title: A static ECS in Javascript
date: 2018-04-24 10:16:37
tags:
- ECS
- Javascript
---

This is a very minimal ECS (Entity Componenent System) implementation. This one is static so it misses one of the big functionality of ECS systems. If you want a more complete implementation, you can look at my TIC-80 boilerplate there: https://github.com/pcornier/TIC80-boilerplate/blob/master/src/libs/ecs.lua

<!-- more -->

```javascript
const World = function() {
  let entities = []
  let systems = []
  return {
    addEntity: function(e) {
      entities.push(e)
      systems.forEach(s => {
        if (s.require.every(p => p in e)) s.entities.push(e)
      })
    },
    addSystem: function(s) {
      s.entities = s.entities || []
      systems.push(s)
    },
    update: () => {
      systems.forEach(s => {
        s.entities.forEach(e => s.update(e))
      })
    }
  }
}
```

and some tests:

```javascript
showNameSystem = {}
showNameSystem.require = ['name']
showNameSystem.update = e => console.log(`I am ${e.name}`)

showPosAndNameSystem = {}
showPosAndNameSystem.require = ['name', 'pos']
showPosAndNameSystem.update = e => console.log(`I am ${e.name} and located at ${e.pos.x}, ${e.pos.y}`)

world.addEntity({ name: 'Bill' })
world.addEntity({ name: 'Mary', pos: {x: 1, y: 1} })

world = new World()
world.addSystem(showNameSystem)
world.addSystem(showPosAndNameSystem)
world.update()
```