---
title: Signal Pattern
date: 2018-08-04 20:19:14
tags:
- gamedev
- observer pattern
- signal pattern
- lua
---

![Signal pattern](/images/signal.png)
<center>*One signal to rule them all*</center>

The Signal (aka. Observer) pattern is one of the most powerful tool for decoupling game objects. The main difference with the classic observer pattern is that the signal is sent to a global event system before it is broadcasted to all listeners, iow., there's only one subject. <!-- more --> Game objects are completely decoupled, if one sends a signal and no one listen because of missing objects, that's not a problem anymore, and if there are some listeners but no signal is emitted, that's okay too. Here is a small Lua implementation.

```lua
------------------------------
-- Global event system
------------------------------

Bus = {
 reg = {}
}

function Bus:emit(event, ...)
 local res = {}
 if self.reg[event] then
  for _, cb in pairs(self.reg[event]) do
   local r = cb(...)
   if r ~= nil then table.insert(res, r) end
  end
 end
 return res
end

function Bus:register(event, cb)
 self.reg[event] = self.reg[event] or {}
 table.insert(self.reg[event], cb)
end

function Bus:clear()
 self.reg = {}
end
```

A bullet system could emit a signal when it collides a monster:
```lua
-- bullet system
Bus:emit('collideMonster', monsterId)
```

Then the monster system can react accordingly:
```lua
-- monster system
Bus:register('collideMonster', function(monsterId)
  -- kill this monster
end)
```
