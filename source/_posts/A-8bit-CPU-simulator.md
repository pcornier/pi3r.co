---
title: A 8bit CPU simulator
date: 2018-10-22 17:14:39
tags:
---

Here is a short introduction to emulation. I will try to explain how to emulate a 8bit CPU.

<!-- more -->

### Memory Emulation

Firstly, we need to emulate the memory. I decided to create a 64k, little endian memory.

Create a `memory.lua` module:
```lua
local memory = {
  ram = {}
}

return memory
```

We need to initialize memory by zeroing it. So we add an initialization function that will fill the entire memory with zeros:

```lua
function memory:initialize()
  for i=0,0xffff do
    self.ram[i] = 0
  end
end
```

Now we can implement a write function in `memory.lua`:
```lua
function memory:write(address, value)
  address = address & 0xffff
  self.ram[address] = value & 0xff
end
```

The function receives two parameters: a location and a byte value. We need to perform a bitwise "and" operation with a 16bit mask to get a safe memory address: if the address exceeds 0xffff, it will wrap to zero e.g. 0x10000 = 0x0000, 0x10001 = 0x0001 etc. Same thing with the byte value, we "and" the value with a 8bit mask.

The read function is really basic:
```lua
function  memory:read(address)
  address = address & 0xffff
  return self.ram[address]
end
```

Now we need a function to read 16bit values from memory. We need one because our memory is a "little endian" system and therefore the order of the bytes is inverted.

```lua
function memory:read16(address)
  return self.ram[(address+1)&0xffff] << 8 |  self.ram[address&0xffff]
end
```

Nothing really complex here, if memory contains $aa $bb at locations 00 and 01 respectively, then $bb is shifted so it becomes $bb00 and $aa is added (OR) to that value. The final 16bit value is $bbaa.


The final function that we can add is a small utility which loads a program into memory.
```lua
function memory:loadPRG(prg, address)
  address = address & 0Xffff
  for i=1,prg:len() do
    self.ram[address+i-1] = prg:byte(i)
  end
end
```

We subtract one to the memory address `self.ram[address+i-1]` because bytes are not zero indexed in Lua. We need to remove one in order to load at the correct location.

Let's create a test file so we can use "Busted" to test our memory: https://github.com/Olivine-Labs/busted

Create a `spec` directory and a `spec/memory_spec.lua`:
```lua
local memory = require 'memory'
memory:initialize()

describe('read/write to memory', function()

  it('can read/write memory', function()
    memory:write(0x1234, 0xff)
    assert.equal(0xff, memory:read(0x1234))
  end)

  it('cannot write more than 8bit', function()
    memory:write(0x4321, 0x105)
    assert.equal(0x05, memory:read(0x4321))
  end)

  it('can read little endian', function()
    memory:write(0x100, 0x01)
    memory:write(0x101, 0xff)
    assert.equal(0xff01, memory:read16(0x100))
  end)

  it('can memory wrap', function()
    memory:write(0xffff, 0x01)
    memory:write(0x0000, 0xff)
    assert.equal(0xff01, memory:read16(0xffff))
  end)

  it('can load PRG in memory', function()
    memory:loadPRG('\01\02\03\04', 0x100)
    assert.equal(0x01, memory:read(0x100))
    assert.equal(0x201, memory:read16(0x100))
    assert.equal(0x403, memory:read16(0x102))
  end)

end)
```

Run busted at the root level of the project and you should see something like:
```
●●●●●
5 successes / 0 failures / 0 errors / 0 pending : 0.012428 seconds
```

### CPU Emulation

Now, the interesting part: we will emulate a basic 8bit CPU.

The Lua file: cpu.lua
```lua
local memory

local cpu = {}

function cpu:initialize(mem)
  memory = mem
end

return cpu
```
We give the memory component to the CPU during initialization.

This 8bit CPU needs internal variables:

- The PC register, which is a 16bit register that points to the next instruction in memory (a read head).
- Three working registers: `A`, `X` and `Y`.
- A stack pointer `SP`.
- And at least four flags: Z (zero), C (carry), N (negative), V (overflow). These flags are either active(1) or inactive(0)

We can initialize these variables in the CPU table:
```lua
local cpu = {

  -- Program Counter aka. Instruction Pointer (16 bit register)
  PC = 0,

  -- 8 bit registers
  A = 0, -- accumulator
  X = 0,
  Y = 0,

  -- stack pointer
  SP = 0xff,

  -- flags
  C = 0, -- carry
  Z = 0, -- zero
  N = 0, -- negative
  V = 0, -- overflow
}
```

Now we can start emulating. The steps are:

1. Load the instruction id (opcode) located at $(PC) value in memory
2. Increment PC so it points to the parameter location
3. Emulate the instruction
4. Return the total number of cycles

```lua
function cpu:emulate()

  -- clock cycles
  local cycles = 0

  -- read operation code in memory at $(PC) location (1)
  local opcode = memory:read(self.PC)

  -- move PC to the next memory address (2)
  self.PC = self.PC + 1

  -- emulate instruction (3)

  -- 0xa9 = LDA IMM : load an immediate value into the accumulator
  if opcode == 0xa9 then
    self.A = memory:read(self.PC)           -- load memory value in accumulator
    self.N = self.A >= 0x80 and 1 or 0      -- update negative flag
    self.Z = self.A == 0 and 1 or 0         -- update zero flag
    self.PC = self.PC + 1                   -- go to next instruction
    cycles = cycles + 2                     -- count cycles
  end

  -- return cycles for synchronizing with other components (4)
  return cycles

end
```

The first emulated instruction is LDA IMM. It means **L**oa**D** *into* **A**ccumulator an immediate value. e.g. if memory contains $A9 $FF (LDA #$FF), the CPU will load $FF into the accumulator and it will update two internal CPU flags. The N flag activates if accumulator is negative (>= 0x80), the Z flag activates if accumulator equals zero. Then the PC is incremented so it points to the next instruction and cycles are added to the cycle counter.

We can create a test file for the CPU: `spec/cpu_spec.lua`:
```lua
local memory = require 'memory'
local cpu = require 'cpu'

memory:initialize()
cpu:initialize(memory)

describe('emulate instructions', function()

  it('can load a value in accumulator', function()
    -- lda $1a : a9 1a
    cpu.PC = 0x100
    memory:loadPRG('\xa9\xa5', cpu.PC)
    cpu:emulate()
    assert.equal(0xa5, cpu.A)
    assert.equal(1, cpu.N)
    assert.equal(0, cpu.Z)
  end)

end)
```

Test it with `Busted`.
We are going to emulate these 8 instructions:

| OP  | INS | ADR | CC  |
|-----|-----|-----|-----|
| $8D | STA | ABS | 4   |
| $9D | STA | ABX | 5   |
| $A2 | LDX | IMM | 2   |
| $A9 | LDA | IMM | 2   |
| $BD | LDA | ABX | 4   |
| $D0 | BNE | REL | 2   |
| $E0 | CPX | IMM | 2   |
| $E8 | INX | IMP | 2   |

This is a very small subset of all the MOS6502 instructions.

First column **OP** is the operation code value, this is the ID of the operation. Second column **INS** is the name of the instruction. Third column **ADR** is the addressing mode: how and where to get the instruction parameters. Last column **CC** is the number of internal CPU clock cycles required to execute the instruction.

We will implement only 5 addressing modes:

- IMM: (immediate) parameter follows the instruction
- ABS: (absolute) load/save from the specified 16bit memory location
- REL: (relative) used with branching instructions, it's a signed byte (vector) that is added to the current instruction address.
- ABX: (absolute X) load/save from the specified 16bit memory location + X
- IMP: (implicit) parameter is implicit

*The original 6502 has **12** addressing modes.*

We can easily emulate the LDX IMM instruction. It's quite similar to the LDA instruction:
```lua
  -- emulate instruction

  -- 0xa9 = LDA IMM : load an immediate value into the accumulator
  if opcode == 0xa9 then
    self.A = memory:read(self.PC)           -- load memory value in accumulator
    self.N = self.A >= 0x80 and 1 or 0      -- update negative flag
    self.Z = self.A == 0 and 1 or 0         -- update zero flag
    self.PC = self.PC + 1                   -- go to next instruction
    cycles = cycles + 2                     -- count cycles

  -- 0xa2 = LDX IMM : load an immediate value into X
  elseif opcode == 0xa2 then
    self.X = memory:read(self.PC)           -- load memory value in X register
    self.N = self.X >= 0x80 and 1 or 0      -- update negative flag
    self.Z = self.X == 0 and 1 or 0         -- update zero flag
    self.PC = self.PC + 1                   -- go to next instruction
    cycles = cycles + 2                     -- count cycles

  end

```

And we can add a test in our spec/cpu_spec.lua
```lua
  it('can load an immediate value in X register', function()
    -- ldx $1b : a2 00
    cpu.PC = 0x100
    cpu.X = 0xff
    memory:loadPRG('\xa2\x00', cpu.PC)
    cpu:emulate()
    assert.equal(0, cpu.X)
    assert.equal(0, cpu.N)
    assert.equal(1, cpu.Z)
  end)
```

Now we can implement STA ABS
```lua
  -- 0x8d = STA ABS : store accumulator at absolute address
  elseif opcode == 0x8d then
    local address = memory:read16(self.PC)  -- read address from memory
    memory:write(address, self.A)           -- save accumulator
    self.PC = self.PC + 2                   -- increment PC (absolute address is 16bit)
    cycles = cycles + 4                     -- count cycles
```

This time, we need to increment the PC register by two because the parameter is two bytes long.

And the test:
```lua
  it('can store accumulator at absolute address', function()
    -- lda $ff   : a9 ff
    -- sta $1000 : 8d 00 10
    cpu.PC = 0x100
    memory:loadPRG('\xa9\xff\x8d\x00\x10', cpu.PC)
    cpu:emulate() --> lda
    cpu:emulate() --> sta
    assert.equal(0xff, memory:read16(0x1000))
  end)
```

The STA ABX instruction is mostly the same except that we need to sum X to the address value before storing:
```lua
  -- 0x9d = STA ABX : store accumulator at absolute + x
  elseif opcode == 0x9d then
    local address = memory:read16(self.PC)  -- read address from memory
    memory:write(address + self.X, self.A)  -- store accumulator at addr+X
    self.PC = self.PC + 2                   -- go to next instruction
    cycles = cycles + 5                     -- count cycles
```

The STA ABX test:
```lua
  it('can load accumulator value at absolute + X', function()
    -- lda $12       : a9 12
    -- sta $0025     : 8d 25 00
    -- ldx $05       : a2 05
    -- lda ($0020)+X : bd 20 00
    cpu.PC = 0x100
    memory:loadPRG('\xa9\x12\x8d\x25\x00\xa2\x05\xbd\x20\x00', cpu.PC)
    for i=1,4 do cpu:emulate() end
    assert.equal(0x12, cpu.A)
  end)
```

Now, the CPX IMM instruction which compares X with an immediate value:
```lua
  -- 0xe0 = CPX IMM : compare with immediate
  elseif opcode == 0xe0 then
    local value = memory:read(self.PC)      -- read immediate value
    self.C = self.X >= value and 1 or 0     -- update carry flag
    value = (self.X - value) & 0xff         -- subtract from X for comparison
    self.Z = value == 0 and 1 or 0          -- update zero flag
    self.N = value >= 0x80 and 1 or 0       -- update negative flag
    self.PC = self.PC + 1                   -- increment PC
    cycles = cycles + 2                     -- count cycles
```
Here we need to update three flags.

Finally, in order to create a small basic program, we need to implement a branching instruction **BNE** and the increment X instruction **INX**:
```lua
  -- 0xd0 = BNE REL : branch to relative address if not equal
  elseif opcode == 0xd0 then
    if self.Z == 0 then                           -- zero flag status of last comparison
      local vect = memory:read(self.PC)           -- read relative vector value
      vect = vect < 0x80 and vect or vect - 0x100 -- convert to signed integer
      self.PC = self.PC + vect                    -- jump to the relative address
    else
      self.PC = self.PC + 1                       -- otherwise, go to next instruction
    end
    cycles = cycles + 2                           -- count cycles

  -- 0xe8 = INX IMP : increment X with implicit value
  elseif opcode == 0xe8 then
    self.X = (self.X+1) & 0xff
    cycles = cycles + 2
```

And the final test:
```lua
  it('can run a small program!', function()
    --- copy memory from $010b to $0000
    -- ldx $00       - $100 : a2 00
    -- lda ($010d)+X - $102 : bd 0d 01
    -- sta ($0000)+X - $105 : 9d 00 00
    -- inx           - $108 : e8
    -- cmp $0a       - $109 : e0 0a
    -- bne -10       - $10b : d0 f6
    -- data          - $10d : 0a 09 08 07 06 05 04 03 02 01
    cpu.PC = 0x100
    memory:loadPRG('\xa2\x00\xbd\x0d\x01\x9d\x00\x00\xe8\xe0\x0a\xd0\xf6\x0a\x09\x08\x07\x06\x05\x04\x03\x02\x01', cpu.PC)
    for i=0,50 do
      cpu:emulate()
    end
    assert.equal(0x090a, memory:read16(0x0))
    assert.equal(0x0102, memory:read16(0x8))
  end)
```

Now, try to implement all the official opcodes from the 6502. Just google for "6502 opcodes" and you will get the full list.

*cpu.lua*
```lua

local memory

local cpu = {

  -- Program Counter aka. Instruction Pointer (16 bit register)
  PC = 0,

  -- 8 bit registers
  A = 0, -- A (accumulator)
  X = 0,
  Y = 0,

  -- stack pointer
  SP = 0xff,

  -- flags
  C = 0, -- carry
  Z = 0, -- zero
  N = 0, -- negative
  V = 0, -- overflow
}

function cpu:initialize(mem)
  memory = mem
end

function cpu:emulate()

  -- store the number of CPU cycles to execute the instruction
  local cycles = 0

  -- read operation code in memory at PC address
  local opcode = memory:read(self.PC)

  -- move PC to the next memory address
  self.PC = self.PC + 1

  -- emulate instruction

  -- 0xa9 = LDA IMM : load an immediate value into the accumulator
  if opcode == 0xa9 then
    self.A = memory:read(self.PC)           -- load memory value in accumulator
    self.N = self.A >= 0x80 and 1 or 0      -- update negative flag
    self.Z = self.A == 0 and 1 or 0         -- update zero flag
    self.PC = self.PC + 1                   -- go to next instruction
    cycles = cycles + 2                     -- count cycles

  -- 0xa2 = LDX IMM : load an immediate value into X
  elseif opcode == 0xa2 then
    self.X = memory:read(self.PC)           -- load memory value in X register
    self.N = self.X >= 0x80 and 1 or 0      -- update negative flag
    self.Z = self.X == 0 and 1 or 0         -- update zero flag
    self.PC = self.PC + 1                   -- go to next instruction
    cycles = cycles + 2                     -- count cycles

  -- 0x8d = STA ABS : store accumulator at absolute address
  elseif opcode == 0x8d then
    local address = memory:read16(self.PC)  -- read address from memory
    memory:write(address, self.A)           -- save accumulator
    self.PC = self.PC + 2                   -- increment PC (absolute address is 16bit)
    cycles = cycles + 4                     -- count cycles

  -- 0xbd = LDA ABX : load accumulator value at absolute + X
  elseif opcode == 0xbd then
    local address = memory:read16(self.PC)  -- read address from memory
    self.A = memory:read(address + self.X)  -- load accumulator from memory at addr+X
    self.N = self.A >= 0x80 and 1 or 0      -- update negative flag
    self.Z = self.A == 0 and 1 or 0         -- update zero flag
    self.PC = self.PC + 2                   -- go to next instruction
    cycles = cycles + 4                     -- count cycles

  -- 0x9d = STA ABX : store accumulator at absolute + x
  elseif opcode == 0x9d then
    local address = memory:read16(self.PC)  -- read address from memory
    memory:write(address + self.X, self.A)  -- save accumulator at addr+X
    self.PC = self.PC + 2                   -- go to next instruction
    cycles = cycles + 5                     -- count cycles

  -- 0xe0 = CPX IMM : compare with immediate
  elseif opcode == 0xe0 then
    local value = memory:read(self.PC)      -- read immediate value
    self.C = self.X >= value and 1 or 0     -- update carry flag
    value = (self.X - value) & 0xff         -- subtract from X for comparison
    self.Z = value == 0 and 1 or 0          -- update zero flag
    self.N = value >= 0x80 and 1 or 0       -- update negative flag
    self.PC = self.PC + 1                   -- increment PC
    cycles = cycles + 2                     -- count cycles

  -- 0xd0 = BNE REL : branch to relative address if not equal
  elseif opcode == 0xd0 then
    if self.Z == 0 then                           -- zero flag status of last comparison
      local vect = memory:read(self.PC)           -- read relative vector value
      vect = vect < 0x80 and vect or vect - 0x100 -- convert to signed integer
      self.PC = self.PC + vect                    -- jump to the relative address
    else
      self.PC = self.PC + 1                       -- otherwise, go to next instruction
    end
    cycles = cycles + 2                           -- count cycles

  -- 0xe8 = INX IMP : increment X with implicit value
  elseif opcode == 0xe8 then
    self.X = (self.X+1) & 0xff
    cycles = cycles + 2

  end

  -- return cycles for synchronizing with other components
  return cycles

end


return cpu
```

*memory.lua*
```lua
local memory = {
  ram = {}
}

function memory:initialize()
  for i=0,0xffff do
    self.ram[i] = 0
  end
end

function memory:write(address, value)
  address = address & 0xffff
  self.ram[address] = value & 0xff
end

function  memory:read(address)
  address = address & 0xffff
  return self.ram[address]
end

function memory:read16(address)
  return self.ram[(address+1)&0xffff] << 8 |  self.ram[address&0xffff]
end

function memory:loadPRG(prg, address)
  address = address & 0Xffff
  for i=1,prg:len() do
    self.ram[address+i-1] = prg:byte(i)
  end
end

return memory
```

*spec/cpu_spec.lua*
```lua
local memory = require 'memory'
local cpu = require 'cpu'

memory:initialize()
cpu:initialize(memory)

describe('emulate instructions', function()

  it('can load a value in accumulator', function()
    -- lda $1a : a9 1a
    cpu.PC = 0x100
    memory:loadPRG('\xa9\xa5', cpu.PC)
    cpu:emulate()
    assert.equal(0xa5, cpu.A)
    assert.equal(1, cpu.N)
    assert.equal(0, cpu.Z)
  end)

  it('can load an immediate value in X register', function()
    -- ldx $1b : a2 00
    cpu.PC = 0x100
    cpu.X = 0xff
    memory:loadPRG('\xa2\x00', cpu.PC)
    cpu:emulate()
    assert.equal(0, cpu.X)
    assert.equal(0, cpu.N)
    assert.equal(1, cpu.Z)
  end)

  it('can store accumulator at absolute address', function()
    -- lda $ff   : a9 ff
    -- sta $1000 : 8d 00 10
    cpu.PC = 0x100
    memory:loadPRG('\xa9\xff\x8d\x00\x10', cpu.PC)
    cpu:emulate() --> lda
    cpu:emulate() --> sta
    assert.equal(0xff, memory:read16(0x1000))
  end)

  it('can load accumulator value at absolute + X', function()
    -- lda $12       : a9 12
    -- sta $0025     : 8d 25 00
    -- ldx $05       : a2 05
    -- lda ($0020)+X : bd 20 00
    cpu.PC = 0x100
    memory:loadPRG('\xa9\x12\x8d\x25\x00\xa2\x05\xbd\x20\x00', cpu.PC)
    for i=1,4 do cpu:emulate() end
    assert.equal(0x12, cpu.A)
  end)

  it('can run a small program!', function()
    --- copy memory from $010b to $0000
    -- ldx $00       - $100 : a2 00
    -- lda ($010d)+X - $102 : bd 0d 01
    -- sta ($0000)+X - $105 : 9d 00 00
    -- inx           - $108 : e8
    -- cmp $0a       - $109 : e0 0a
    -- bne -10       - $10b : d0 f6
    -- data          - $10d : 0a 09 08 07 06 05 04 03 02 01
    cpu.PC = 0x100
    memory:loadPRG('\xa2\x00\xbd\x0d\x01\x9d\x00\x00\xe8\xe0\x0a\xd0\xf6\x0a\x09\x08\x07\x06\x05\x04\x03\x02\x01', cpu.PC)
    for i=0,50 do
      cpu:emulate()
    end
    assert.equal(0x090a, memory:read16(0x0))
    assert.equal(0x0102, memory:read16(0x8))
  end)

end)
```

*spec/memory_spec.lua*
```lua
local memory = require 'memory'

memory:initialize()

describe('read/write to memory', function()

  it('can read/write memory', function()
    memory:write(0x1234, 0xff)
    assert.equal(0xff, memory:read(0x1234))
  end)

  it('cannot write more than 8bit', function()
    memory:write(0x4321, 0x105)
    assert.equal(0x05, memory:read(0x4321))
  end)

  it('can read little endian', function()
    memory:write(0x100, 0x01)
    memory:write(0x101, 0xff)
    assert.equal(0xff01, memory:read16(0x100))
  end)

  it('can memory wrap', function()
    memory:write(0xffff, 0x01)
    memory:write(0x0000, 0xff)
    assert.equal(0xff01, memory:read16(0xffff))
  end)

  it('can load PRG in memory', function()
    memory:loadPRG('\01\02\03\04', 0x100)
    assert.equal(0x01, memory:read(0x100))
    assert.equal(0x201, memory:read16(0x100))
    assert.equal(0x403, memory:read16(0x102))
  end)

end)
```
