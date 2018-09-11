---
title: Goal Oriented Action Planning
date: 2018-04-27 10:32:23
tags:
- GOAP
- gamedev
- Javascript
- LUA
---

Two GOAP (Goal Oriented Action Planning) implementations in Javascript and LUA.

<!-- more -->

## Javascript
```javascript
const Action = (...args) => {
  [name, effects = {}, conditions = {}, weight = 1] = args
  return { name, effects, conditions, weight }
}

const Planner = (...args) => {

  let actions = args

  const planner = {

    findPlan: (start, goal) => {

      let buildGraph = (parent, goal, actions, weight = 0) => {
        actions.forEach(action => {
          let state = parent.effects || parent
          if (Object.keys(action.conditions).every(prop => action.conditions[prop] === state[prop])) {
            action.cost = action.weight + weight
            action.parents = action.parents || []
            action.parents.push(parent)
            if (Object.keys(goal).filter(prop => prop != 'parents').every(prop => action.effects[prop] === goal[prop])) {
              goal.parents = goal.parents || []
              goal.parents.push(action)
            }
            else {
              let subset = actions.slice()
              subset.splice(subset.indexOf(action), 1)
              buildGraph(action, goal, subset, action.cost)
            }
          }
        })
      }

      buildGraph(start, goal, actions)
      let plan = []
      let node = goal
      while (node.parents) {
        node = node.parents.sort((a, b) => a.cost > b.cost)[0]
        if (node.name) plan.push(node.name)
      }

      return plan.reverse()

    }
  }
  return planner
}

let planner = Planner(
  //        name             effects          conditions        weight
  Action('goHomeByCar',   {isHome: true},   {hasCar: true},        1),
  Action('goHomeByBus',   {isHome: true},   {foundBus: true},      1),
  Action('getCar',        {hasCar: true},   {hasKeys: true},       4),
  Action('getBus',        {foundBus: true}, {hasMoney: true},      2),
  Action('getCash',       {hasMoney: true}, {hasCreditCard: true}, 1)
)

let plan = planner.findPlan({hasCreditCard: true, hasKeys: true}, {isHome: true})
console.log(plan)

// 0: 'getCash'
// 1: 'getBus'
// 2: 'goHomeByBus'
```

## LUA
```lua
local function all(callback, gen)
  for k,v in pairs(gen) do
    if k ~= 'parents' then
      local result = callback(k, v)
      if not result then return false end
    end
  end
  return true
end

local Planner = function(...)

  local actions = {...}
  local planner = {}

  function planner:buildGraph(parent, goal, actions, weight)
    weight = weight or 0
    for i = 1, #actions do
      local action = actions[i]
      local state = parent.effects or parent

      if all(function(k, v) return v == state[k] end, action.conditions) then
        action.cost = action.weight + weight
        action.parents = action.parents or {}
        table.insert(action.parents, parent)

        if all(function(k, v) return v == action.effects[k] end, goal) then
          goal.parents = goal.parents or {}
          table.insert(goal.parents, action)
        else
          local subset = {unpack(actions)}
          table.remove(subset, i)
          planner:buildGraph(action, goal, subset, action.cost)
        end

      end
    end
  end

  function planner:findPlan(start, goal)

    planner:buildGraph(start, goal, actions)
    local plan = {}
    local node = goal
    while node.parents do
      table.sort(node.parents, function(a, b) return a.cost > b.cost end)
      node = node.parents[1]
      if node.name then table.insert(plan, node.name) end
    end

    local reverse = {}
    for i = #plan, 1, -1 do
      table.insert(reverse, plan[i])
    end
    return reverse

  end

  return planner
end


local planner = Planner(
  { name = 'goHomeByCar', effects = { isHome = true }, conditions = { hasCar = true }, weight = 1 },
  { name = 'goHomeByBus', effects = {isHome = true}, conditions = {foundBus = true}, weight = 1},
  { name = 'getCar',      effects = {hasCar = true},    conditions = {hasKeys = true}, weight = 2 },
  { name = 'getBus',      effects = {foundBus = true},  conditions = {hasMoney = true}, weight = 2 },
  { name = 'getCash',     effects = {hasMoney = true},  conditions = {hasCreditCard = true}, weight = 1 }
)

local plan = planner:findPlan({hasCreditCard = true, hasKeys = true}, {isHome = true})
print(inspect(plan))
```