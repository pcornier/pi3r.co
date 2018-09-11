---
title: A better ECS for LUA
date: 2018-04-24 13:46:51
tags:
- gamedev
- ECS
- lua
- TIC-80
---

A better dynamic ECS implementation.

![example game](/images/ecs.gif)
<center>*An ECS based game made with TIC-80*</center>

<!-- more -->

```lua
------------------------------
-- Entity Component System
------------------------------

World = function()
 local systems = {}
 local entities = {}
 local world = {}

 local function match(filter, entity)
  for _, component in pairs(filter) do
   if entity[component] == nil then return false end
  end
  return true
 end

 function world:addEntity(entity)
   entities[#entities+1] = entity
   for _, s in pairs(systems) do
    local system = s.system
    if match(system.filter, entity) and system.newEntity then
     system:newEntity(entity)
    end
   end
 end

 function world:addSystem(system)
  system.filter = system.filter or {}
  systems[#systems+1] = {
   update = function(entities, ...)
    system.entities = {}
    for _, entity in pairs(entities) do
     if match(system.filter, entity) then
      system.entities[#system.entities+1] = entity
     end
    end
    if system.update then system:update(...) end
    if system.process then
     for _, entity in pairs(system.entities) do
      system:process(entity, ...)
     end
    end
   end,
   system = system
  }
  system.world = world
  if system.added then system:added() end
 end

 function world:filter(filter)
  matches = {}
  for _, entity in pairs(entities) do
   if match(filter, entity) then
    matches[#matches+1] = entity
   end
  end
  return matches
 end

 function world:pick(filter)
  for _, entity in pairs(entities) do
   if match(filter, entity) then
    return entity
   end
  end
 end

 function world:update(...)
  for _, system in pairs(systems) do
   system.update(entities, ...)
  end
 end

 function world:removeEntity(entity)
  for i, e in pairs(entities) do
   if e == entity then
    table.remove(entities, i)
    break
   end
  end
 end

 function world:getEntities()
  return entities
 end

 function world:getSystems()
  return systems
 end

 return world
end
```

A the test file.
```lua
local world = World()

-- can add entities

local e1 = {
  pos = { x = 10, y = 10 },
  color = 'blue'
}

local e2 = {
  pos = { x = 20, y = 10 }
}

world:addEntity(e1)
world:addEntity(e2)
assert(#world:getEntities() == 2, 'wrong number of entities')

-- can filter entities
assert(#world:filter({ 'pos' }) == 2, 'cannot filter entities')

-- can pick one entity
assert(world:pick({ 'color' }) == e1, 'cannot pick up entity')

-- can add systems, trigger added() and access world ref

local posSys = {
  filter = { 'pos' }
}

function posSys:added()
  assert(self.world == world, 'cannot access world reference')
end

function posSys:process(entity, data)
  entity.pos.x = entity.pos.x + 1
  entity.pos.y = entity.pos.y + 1
  assert(data.payload == 'ok', 'payload not received in process()')
end

function posSys:update(data)
  assert(data.payload == 'ok', 'payload not received in update()')
  assert(#self.entities == 2, 'wrong number of entities in pos system')
end

local colSys = {
  filter = { 'pos', 'color' }
}

function colSys:update()
  assert(#self.entities == 1, 'wrong number of entities in col system')
end

world:addSystem(posSys)
world:addSystem(colSys)
assert(#world:getSystems() == 2, 'wrong number of systems')


-- can trigger update() and process() on systems

world:update({ payload = 'ok' })

-- can remove entities

world:removeEntity(e1)
assert(#world:getEntities() == 1, 'wrong number of entities after deletion')
world:removeEntity(e2)
assert(#world:getEntities() == 0, 'wrong number of entities after deletion')
```

https://github.com/pcornier/TIC80-boilerplate
https://github.com/pcornier/tic80-mini-platformer
