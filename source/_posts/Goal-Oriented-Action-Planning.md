---
title: Goal Oriented Action Planning
date: 2018-04-27 10:32:23
tags:
- GOAP
- gamedev
- LUA
---

A GOAP (Goal Oriented Action Planning) implementation in LUA.

<video autoplay loop muted markdown="1">
  <source src="/images/goap.webm" type="video/webm" markdown="1">
</video>

<!-- more -->

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

  local actions = {}
  for i,_a in pairs({...}) do
    local a = {}
    for j,v in pairs(_a) do a[j] = v end
    actions[i] = a
  end

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
          for k,v in pairs(state) do if not action.effects[k] then action.effects[k] = v end end
          planner:buildGraph(action, goal, subset, action.cost)
        end

      end
    end
  end

  function planner:findPlan(start, goal)

    local state = {}
    for k,v in pairs(start) do state[k] = v end

    planner:buildGraph(state, goal, actions)
    local plan = {}
    local node = goal
    while node.parents do
      table.sort(node.parents, function(a, b) a.cost = a.cost or 0 b.cost = b.cost or 0 return a.cost > b.cost end)
      if node.name then table.insert(plan, node.name) end
      node = node.parents[1]
    end

    local reverse = {}
    for i = #plan, 1, -1 do
      table.insert(reverse, plan[i])
    end
    return reverse

  end

  return planner
end

return Planner
```

## Usage
Pass a list of actions to the planner then call the `findPlan()` method.

```lua
local state = { playerInSight = false, withinRange = false, playerIsDead = false }
local goal = { playerIsDead = true }
local actions = {
  { name = 'patrolling', conditions = { playerInSight = false }, effects = { playerInSight = true }, weight = 2 },
  { name = 'reachPlayer', conditions = { playerInSight = true }, effects = { withinRange = true }, weight = 1 },
  { name = 'attack', conditions = { withinRange = true }, effects = { playerIsDead = true }, weight = 1 },
  ...
}
local planner = Planner(unpack(actions))
local plan = planner:findPlan(state, goal)
```
