---
title: Vector Field path finding
date: 2018-05-08 10:40:34
tags:
- gamedev
- lua
- pathfinding
---

A vector field path finding for LUA. This demonstrates the importance of relocation of intensive calculations: entity based pathfinding vs global grid pathfinding.

![zombies!](/images/zombies.gif)
<center>*Hungry zombies will try to avoid bushes*</center>

<!-- more -->

```lua
local FlowVF = {}

-- getWeight: callback (x, y) and should return a weight between 0 and 1
-- setvector: callback (x, y, angle) called on each cell
-- the function returns the weight table
function FlowVF:createVectors(px, py, w, h, getWeight, setVector)

  local neighbors = {{0, -1}, {-1, 0}, {1, 0}, {0, 1}}
  local done = {}
  local todo = {}
  local indexes = {}
  local idx = py * w + px
  todo[idx] = { x = px, y = py, c = 0, i = getWeight(px, py) }
  indexes[#indexes+1] = idx

  repeat
    idx = table.remove(indexes, 1)
    if idx then
      local pt = todo[idx]

      for i=1, #neighbors do

        local nx = pt.x + neighbors[i][1]
        local ny = pt.y + neighbors[i][2]
        local j = ny * w + nx
        local cost = done[j] and done[j].c or math.huge
        local itg = getWeight(nx, ny)

        if done[j] == nil and
           todo[j] == nil and
           pt.c < cost and
           itg < 1 and
           nx >= 0 and ny >= 0 and
           nx < w and ny < h then

          indexes[#indexes+1] = j
          todo[j] = { x = nx, y = ny, c = pt.c + 1, i = itg }

        end

      end

      done[idx] = pt
      todo[idx] = nil

    end
  until idx == nil

  neighbors = { -w-1, -w, -w+1, -1, 1, w-1, w, w+1 }
  angles = { -math.pi*0.75, -math.pi/2, -math.pi/4, -math.pi, 0, math.pi*0.75, math.pi/2, math.pi/4 }

  for i, pt in pairs(done) do
    local idx = pt.y * w + pt.x
    local min = math.huge
    local dir = 1
    for i=1, #neighbors do
      local p = done[idx+neighbors[i]]
      if p and p.c + p.i < min then
        min = p.c + p.i
        dir = i
      end
    end
    setVector(pt.x, pt.y, angles[dir])
  end

  return done

end

return FlowVF
```