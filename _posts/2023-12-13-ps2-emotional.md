## Writing a PS2 emulator, the guide: Part 2, getting started!

Well, here we are again. It's been several months since my last blog post, and I've been busy working on a PS3 emulator among other things.
But I finally found the time (and the motivation) to write another post. By the end of this part, we should execute our first EE instruction! So, let's get into it!

## Foreword

We'll be developing this emulator in C++, but it could easily be adapted to a variety of other platforms. We'll also be using SDL2 to blit a framebuffer to the screen, and most of the instructions will assume a Linux development environment, although they could be adapted to Windows with little to no hassle.

You'll also need at least a passing experience with both emulation development and C++ programming, as this blog will become far too long if I explain all the basics.

## The BIOS ROM
Before we can begin, you'll need a copy of the 4MB BIOS used in the PlayStation 2. There are several different revisions, but I've found the easiest to boot initially is a strange Japanese prototype BIOS called 'SCPH10000.BIN'. Of course, I can't legally share this ROM, so you'll have to dump it for yourself from a console or do a bit of digging on the internet.

Now, let's set up our directory structure:

![A capture of the tree command, showing a folder named src with a subfolder named mem, a folder named build, and a file named CMakeLists.txt](/images/tut_code_layout_initial.png)

The `mem` folder will contain our `Bus` namespace. This namespace will contain all of our read and write functions for the console. So, let's create that now.

## The Code
Let's define the interface for our Bus. Create `src/mem/Bus.h` and open it.

```c++
#pragma once

#include <stdint.h>
#include <string>

namespace Bus
{

void Init(std::string biosName);

}
```

So, first we begin with our standard include functions. No surprised there.

Next, we define `Bus` as a namespace. I believe that too much OOP is a bad thing, so we'll be defining a decent chunk of our codebase in namespaces.

The `Init()` function will be responsible for initializing our memory arrays and loading the BIOS file into memory.

Let's implement it in `src/mem/Bus.cpp` now.

```c++
#include <mem/Bus.h>
#include <fstream>

namespace Bus
{

	void Init(std::string biosName)
	{
		std::ifstream biosFile(biosName, std::ios::binary | std::ios::ate);
		size_t size = biosFile.tellg();
		biosFile.seekg(0, std::ios::beg);
	}

}
```

So, now we've got the file opened and we've got the size. Let's add an assert to make sure our file is the proper size:

```c++
...
#include <cassert>
...
	// Inside Init()
	assert(size == 4*1024*1024); // Make sure the file is 4MB exactly
```

Next, we'll need an array to store the BIOS in:

```c++
...
uint8_t* bios;
...
	// At  the top of Init()
	bios = new uint8_t[4*1024*1024];
```

Now we can read the file into this array:

```c++
file.read((char*)bios, size);
file.close();
```

Now, let's set up `src/main.cpp` to do some basic `argv` checking and have it initialize our Bus namespace.

```c++
#include <stdio.h>
#include <mem/Bus.h>

int main(int argc, char** argv)
{
	if (argc < 2)
	{
		printf("Usage: %s <bios>\n", argv[0]);
		return 1;
	}

	Bus::Init(argv[1]);
	return 0;
}
```

Next, we'll setup CMake to build our project for us:

```CMake
project(PS2)

set(CMAKE_MINIMUM_REQUIRED_VERSION 3.16.3)

set(CXX_STANDARD c++23)

set(SOURCES src/main.cpp
            src/mem/Bus.cpp)

set(CMAKE_BUILD_TYPE Debug)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_executable(ps2 ${SOURCES})
set(TARGET_NAME ps2)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)
```

Now, on Linux, the project can be built by typing    
```sh
cd build
cmake ..
make -j$(nproc -a)
```

Try it out by typing `./ps2` followed by your BIOS file name. It should work!

Now we can start defining our Emotion Engine. First, we'll need to add a `Read32()` function to `Bus`, as instructions on the PS2 are 32-bits long.

```c++
// In Bus.h
...
uint32_t Read32(uint32_t addr);
...
// In Bus.cpp
...
#include <stdio.h>
#include <stdlib.h>
...
uint32_t Read32(uint32_t addr)
{
	printf("[EE]: Read32 from unknown address 0x%08x\n", addr);
	exit(1);
}
...
```

So, let's discuss the Emotion Engine's memory map. On MIPS processors, addresses can be virtual, translated to physical addresses using this table:

![A list of MIPS segments, followed by their corresponding physical addresses](/images/memory_translation_mips.png)

This can be simplified down to this code, where the top 

```c++
uint32_t TranslateAddress(uint32_t addr)
{
	return addr & 0x1FFFFFFF;
}
```

However, there is an exception to this rule: scratchpad RAM, which is CPU-side, can only be accessed via virtual addressing at `0x70000000`. Let's add this now:

```c++
uint32_t TranslateAddress(uint32_t addr)
{
	if ((addr & 0xF0000000) == 0x70000000)
		return addr;
	return addr & 0x1FFFFFFF;
}
```

Next, let's fill out our `Read32()` function to allow reading from the BIOS, which is mapped starting at `0x1fc00000 - 0x20000000 (4MB)`:

```c++
uint32_t Read32(uint32_t addr)
{
	addr = TranslateAddr(addr);

	if (addr >= 0x1fc00000 && addr < 0x20000000)
		return *(uint32_t*)&bios[addr & 0x3FFFFF];

	printf("[EE]: Read32 from unknown address 0x%08x\n", addr);
	exit(1);
}
```

We do a bitwise `and` on the address with 4MB-1 to truncate the address to the proper offset within the BIOS.

Now, let's make our EmotionEngine namespace. Make a new folder under `src` named `cpu`, and then make a new folder under that named `ee`. Create `EmotionEngine.h` and `EmotionEngine.cpp`.

In `EmotionEngine.h`, we'll set up our `Reset()` and `Tick()` functions:

```c++
#pragma once

namespace EmotionEngine
{

	void Reset();
	void Tick(int cycles);

}
```

Don't worry about the `cycles` parameter for now. That will come in handy when we implement a scheduler later. For now, let's move on to `EmotionEngine.cpp`:

```c++
#include <stdint.h>
#include <cpu/ee/EmotionEngine.h>

namespace EmotionEngine
{


	void Reset()
	{

	}

}
```

So, let's take a look at how a MIPS processor is laid out:
- First, there are 32 128-bit general purpose register
- There is a 32-bit PC that keeps track of the current instruction
- There are three coprocessors, which are accessed through the `mtc0` and `mfc0` instructions. 
- Branches take one extra instruction to take effect, leading to an extra instruction being executed after a branch. This is called a "branch delay slot"

So, let's implement some of this:
```c++
...
// Inside the EmotionEngine namespace
uint32_t pc, next_pc; // next_pc will be used to handle branch delay slots
```

Hold on, 128-bit GPRs? How will we emulate those? On GCC, the answer is a builtin type named `__int128`, which we can use for native 128-bit integers. So, let's create another file named `src/types.h`:
```c++
#pragma once

#include <stdint.h>

union uint128_t
{
	unsigned __int128 u128;
	uint64_t u64[2];
	uint32_t u32[4];
	uint16_t u16[8];
	uint8_t u8[16];
};
```

We use a `union` so we can easily access the different register sizes with minimal hassle.

Now, include it in `EmotionEngine.cpp` and define our GPRs, along with COP0's registers:

```c++
uint128_t regs[32];
uint32_t cop0_regs[32];
```

We'll explain some of the functions of COP0's registers as they become relevant. Now, let's implement our `Reset()` function.

```c++
#include <cstring>

void Reset()
{
	memset(regs, 0, sizeof(regs));
	pc = 0xbfc00000;
	next_pc = pc + 4;
}
```

The CPU starts executing at `0xbfc00000`, which is translated to `0x1fc00000`, which maps to the BIOS.

Now, let's implement the `Tick()` function:

```c++

void Tick(int cycles)
{
	for (int cycle = 0; cycle < cycles; cycle++)
	{
		uint32_t instr = Bus::Read32(pc);
		pc = next_pc;
		next_pc += 4;

		uint32_t op = (instr >> 26) & 0x3F;
		switch (op)
		{
		default:
			printf("[EE]: Unknown opcode 0x%02x (0x%08x)\n", op, instr);
			exit(1);
		}
	}
}
```

Now, in `src/main.cpp`:

```c++
EmotionEngine::Reset();

while (true)
{
	EmotionEngine::Tick(1);
}
```

Add the source files to `CMakeLists.txt`:

```CMake
set(SOURCES src/main.cpp
            src/mem/Bus.cpp
	    src/cpu/ee/EmotionEngine.cpp)

```

Build and run, and you should get this:
```sh
[EE]: Unknown opcode 0x10 (0x401a7800)
```

And that concludes this part! In the next part, we'll get into the nitty-gritty of implementing EE opcodes, and try to get our first message in the TTY console!