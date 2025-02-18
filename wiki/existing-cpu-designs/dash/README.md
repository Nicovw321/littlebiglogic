Dash is a 24-bit, 30hz, 8-core CPU made by Mazzetip in LittleBigPlanet 3.
This CPU is equipped with 2 address buses (Code address bus and Variables address bus), a 32-word stack, two cache systems (Code cache and Variables cache) and two interrupt signals (Maskable and non-maskable).

## Execution
As said before, Dash has 8 cores at its disposal. These cores, instead of running in parallel, they run in series, meaning every core is executed in sequence, one after the other, taking advantage of how LittleBigPlanet 3 updates logic; if you run your logic in sequence everything can be executed in a single frame. Effectively, the CPU is running at 240hz instead of 30. But there is a catch! The first 7 cores of the CPU can do addition, substraction, division, multiplication, everything *except* memory and stack accesses, since those can only be accessed by the last core. In practice, if it ever needs to access memory and it's at the first 7 cores, it'll skip cores until it reaches the last one and then do the execution. This slows down the CPU (depending on how many cores were skipped, it could slow down to 210hz, 180hz, 150hz, 120hz, 90hz, 60hz, 30hz) however once it executes the next frame, it'll speed back up to 240hz (unless there is another memory/stack access).

## Interrupts
The CPU has 2 interrupts. A non-maskable interrupt (NMI) and a maskable interrupt (IRQ). Both have its respective vectors.
Once an interrupt is set, the CPU *interrupts* its execution, or wakes back up if it was halted, and it begins its Interrupt Preparation Sequence™.
First, it saves all its flags to the return stack. Then, it reads from the address at the vector from the interrupt. If NMI was triggered, it reads from the NMI vector, if IRQ was triggered, it reads from the IRQ vector (if both are triggered, the NMI vector is used). Then it sends a heartbeat signal to the cores that kickstart the execution. (There is a heartbeat signal that runs through the cores while they are executing. Once an interrupt happens, that heartbeat signal is intercepted and delayed until it finishes the Interrupt Preparation Sequence™)

## Bitwise math
To remove complexity on cores, this CPU is unable to execute bitwise math. It can only do addition, substraction, division, modulo, and multiplication. It's possible to emulate in software, though it's better to avoid bitwise math as much as possible on this CPU.
To see if any bit is set to 1 for example, you can do division and modulo: Divide the value to ``2^bit`` and reduce it with ``modulo 2`` to get a 1 or 0.

## Code Cache
Code cache is what the CPU uses to access instructions, on the Cache address bus. At the start of a frame, a memory chip on the Code address bus returns 16 24-bit values to the CPU, which are from ``address+0`` to ``address+15`` (like getting the value at the address requested, then also returning the next 15 after it. The CPU takes note of the address of each value creating a list. Then, inside a core, before executing, it searches on that list of 16 values to see if the current program counter matches any of the address associated with those 16 values. If there is a match it'll fetch the value from this volatile cache and execute the instruction, if there is a miss, it'll skip to the last core to try to fetch the insturction at that address. Again, this slows down execution and it depends on how many cores it skips.

## Variable Cache
Variable cache is a more complicated system than the Code cache, it also has 16 slots but those 16 slots not only hold the value, they hold the address associated with that value and its usage number. These 3 numbers make it so the CPU can read and write cache addresses at random (unlike the Code cache that it's limited to 16 values one after the other, and also unwriteable!).
- **Value:** As the name suggests, it's the value held from the address at the Variable address bus. It can be changed, and that sets the the dirty flag meaning the value is different from the RAM at the Variable address bus.
- **Address/Unused flag:** It's the address that the cache line is referencing. It's a 24-bit value, and when the cache address is unused it's set to 16777216 (100% if the output is read with a note or pressing L2) which is an impossible address to set normally.
- **Usage number/dirty flag:** It's a 24-bit value, when a non-dirty cache line is read, the usage number is incremented by 1. As mentioned before in Value, the CPU can write to a cache value and when that happens, the cache line is "unsynced" from the RAM address it's referencing (since it's not writing to RAM yet). So just like the Address, the Usage Number is set to 16777216 to set it as dirty and it's practically impossible to increment the usage number to reach 16 million accesses (it would take around 600 hours ingame if all cores write to the cache line non-stop).
### Addding cache lines
When the CPU accesses a memory address that isn't cached, it'll go to the last core and read from the actual address. Then it'll add it to the cache. There are 3 steps/attempts the CPU does internally for adding a cache line.
1. **It tries to find an empty cache line.** It'll try to find a cache line where the address bit is set to 100%. When it finds one, it'll add it at that location. If it doesn't find any empty cache line, it'll go to the 2nd step.
2. **It sees if all of the cache lines are dirty.** If all of them are dirty, give up trying to add the cache line, to avoid corruption. If there are some that aren't dirty, it goes to the 3rd and last step.
3. **Overwrite least used cache line.** Ah, see? That's what the usage number was for. It looks at every single cache line and determines which one is the least used to throw out. It'll skip cache lines with the dirty flag set, since well, cache lines with the dirty flag set technically has 16 million uses (the usage number is at 100% for the dirty flag), so the CPU determines it's hella used and never select it.
### Other Stuff
- Cache reads and writes can be disabled/enabled. You can have cache writes disabled but reads enabled to improve efficiency and not clutter the cache with dirty cache lines.
- When a cache line is written at the last core, it doesn't make it dirty since the last core can also access memory. What the CPU does is write to the actual memory address and then remove the dirty flag from the cache value, if it was previously dirty.
- When you want to write back the dirty cache lines to the Variables address bus, you can execute the instruction Flush Cache which will write all dirty cache lines and wipe out the entire cache.
### Hardware error
When the CPU is first ran, the cache lines are all set to the address 0 and value 0. This can potentially slow down execution and/or cause corruption on the address 0, so to combat this the CPU has to have the instruction set to Flush Cache, because once you write to address 0 it could set the dirty flag to true on all the cache lines and cause 16 writes to the same address once you execute Flush Cache. Also if you read to address 0 the cache would get a hit and return the value 0, which it isn't what's in the Variables address bus since it never read from there. This bug can be ignored in the software but if so, the software has to evade writes to address 0 until it hits the first Flush Cache.
This error wasn't fixed since it required many more objects to do so, it's cheaper to do in software.

## Requirements
The CPU needs two address buses populated.
On the Code address bus, it needs 24-bit RAM/ROM with 0 frame delay and interleaved 16 times so it can read a block from ``address+0`` to ``address+15`` in a single frame. Reading will reach speeds up to 30hz, and writing will reach up to 15.
On the Variables address bus, it needs 24-bit RAM/ROM (RAM recommended) with 0 frame delay. Reading at 30hz and writing at 30hz.
To connect the memory chips, a tag design has to be made around the CPU. The CPU's memory outputs go to the corresponding memory chip and the outputs of both memory chips will go to a set of tags. 16 tags for the Code address bus memory chip and 1 tag for the Variables memory chip. Then on the CPU's inputs, tag sensors are connected that detect the tags and send it to the CPU. This effectively does a 1 frame delay on the memory (due to the funky way tags are made), but the CPU is prepared for such delay and works as a pipeline instead.

## Footage of Dash
TODO

## ISA
TODO
