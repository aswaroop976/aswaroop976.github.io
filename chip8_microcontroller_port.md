
# Table of Contents

1.  [CHIP-8 emulator in Rust ported to the STM32f411 microcontroller](#org73f4c5c)
    1.  [Backend(CHIP-8 emulator)](#org4b867ef)
        1.  [Chip8 struct](#orgd5a1bcb)
        2.  [Opcodes](#orgbb3395c)
        3.  [Emulation cycle](#orgebf9881)
    2.  [Frontend(Microcontroller port)](#org90567d3)
        1.  [Hardware](#orgbf9cfdd)



<a id="org73f4c5c"></a>

# CHIP-8 emulator in Rust ported to the STM32f411 microcontroller

-   This project was inspired by this blog post: <https://dhole.github.io/post/chip8_emu_2/>
-   Here I will explain how I wrote a CHIP-8 emulator and ported it to an embedded ARM microcontroller, if you are curious about all the code here is the repository linked(ignore the name of the repository): <https://github.com/aswaroop976/ssd1306_test>
-   I will split this post into two parts first talking about the backend, which consists of the CHIP-8 emulator, and then the frontend, which is where I will explain how I ported this emulator to a micrcontroller and interfaced with a display and buttons
-   Additionally this post assumes pre-requisite knowledge of Rust(nothing too complex), and a basic understanding of computer architecture and embedded systems(although I will try to make this as beginner friendly as possible, linking to helpful resources whereever I can)


<a id="org4b867ef"></a>

## Backend(CHIP-8 emulator)

-   This part of the project was largely inspired from this blog post(seriously this post probably explains things way better than I ever can): <https://austinmorlan.com/posts/chip8_emulator/>


<a id="orgd5a1bcb"></a>

### Chip8 struct

-   First I defined a Chip8 struct where I store the state of the emulator at any point in time:
    
        pub struct Chip8 {
            pub memory: [u8; MEMORY_SIZE],       // 4kb memory
            pub registers: [u8; REGISTER_COUNT], // 16 general purpose registers
            pub index_register: u16,
            pub program_counter: u16,
            pub screen: [u8; SCREEN_WIDTH * SCREEN_HEIGHT], // 64x32 pixel display
            pub delay_timer: u8,
            pub sound_timer: u8,
            pub return_stack: [u16; STACK_SIZE], // return_stack with 16 levels
            pub stack_pointer: u8,        // return_stack pointer
            pub keys: [u8; REGISTER_COUNT],
            pub jump_table: [OpcodeHandler; 16],
        }

1.  Important fields

    -   return stack: This essentially serves as the return address stack. A stack depth of 16 limits the number or recursive function calls to 16. The stack pointer field points to the top of the return address stack.
    -   memory: the CHIP-8 contains a 4kb memory address space, which requires 16 bits to fully address. However the general purpose registers are only 8 bits wide, to account for this there is a special 16 bit index register. The index register is used specifically to store memory addresses
    -   screen: This represents a seperate memory buffer to drive the 64x32 CHIP-8 display. The display is monochrome(black and white), so the pixel values are either 1(on), or 0(off).
    -   keys: CHIP-8 has 16 input keys, the first 16 hexadecimal values: 0 through F, each key is pressed or not
    -   jump table: This is where we will process opcodes and use function pointers to jump to appropriate opcode handler fuction for every instruction that we read from ROM files


<a id="orgbb3395c"></a>

### Opcodes

-   For every opcode present in the Chip8 ISA(instruction set architecture), I created an opcode handler function, eg:
    
        // DRW Vx, Vy, nibble
        // Instruction: display n-byte sprite starting at memory location I at (Vx, Vy), set VF =
        // collision
        fn drw_vx_vy_nibble(&mut self, opcode: u16) {
            let x = ((opcode & 0x0F00) >> 8) as usize;
            let y = ((opcode & 0x00F0) >> 4) as usize;
            let height = (opcode & 0x000F) as usize;
        
            let vx = self.registers[x] as usize;
            let vy = self.registers[y] as usize;
        
            self.registers[0xF] = 0;
        
            for row in 0..height {
                let sprite_byte = self.memory[self.index_register as usize + row];
                for col in 0..8 {
                    let sprite_pixel = sprite_byte & (0x80 >> col);
                    let screen_index = (vy + row) * SCREEN_WIDTH + (vx + col);
        
                    if screen_index < self.screen.len() {
                        let screen_pixel = &mut self.screen[screen_index];
                        if sprite_pixel != 0 {
                            if *screen_pixel == 1 {
                                self.registers[0xF] = 1;
                            }
                            *screen_pixel ^= 1;
                        }
                    }
                }
            }
        }
    
    -   In this instruction I have to modify the screen field by drawing a sprite at the specified location, this is the main way that CHIP-8 programs interact with the display


<a id="orgebf9881"></a>

### Emulation cycle

-   Here I present my emulate cycle function where I perform the fetch/decode/execute stages of the CHIP-8 emulator(My opcode handlers deal with memory instructions, as well as writing to registers). The fetch opcode instruction is a helper function to fetch instructions
    
        pub fn fetch_opcode(&self) -> u16 {
            let high_byte = self.memory[self.program_counter as usize] as u16;
            let low_byte = self.memory[(self.program_counter + 1) as usize] as u16;
            (high_byte << 8) | low_byte
        }
        
        pub fn emulate_cycle(&mut self) {
            let opcode = self.fetch_opcode();
            self.program_counter += 2;
            let index = (opcode & 0xF000) >> 12;
            let handler = self.jump_table[index as usize];
            handler(self, opcode);
        
            if self.delay_timer > 0 {
                self.delay_timer -= 1;
            }
            if self.sound_timer > 0 {
                self.sound_timer -= 1;
            }
        }

-   After loading the ROM into the Chip8&rsquo;s memory(I will expand more on how I do this in my frontend section), I fetch the current instruction using the program counter(this points to the instruction to be fetched). Then I increment the program counter to point to the next instruction to be fetched(for control/branch instructions the opcode handler will set the program counter accordingly). Then using the opcode from the instruction fetched I can get the relevant opcode handler from the jump table.


<a id="org90567d3"></a>

## Frontend(Microcontroller port)


<a id="orgbf9cfdd"></a>

### Hardware

1.  STM32F411

    ![img](/black_pill.jpg)
    
    -   I chose the STM32F411, successor to the popular &ldquo;blue-pill&rdquo; STM32 micrcontroller, because of it&rsquo;s robust Rust support and powerful internals(for a microcontroller)
    -   The specs include:
        1.  512 Kb of flash memory, 128 Kb of SRAM
        2.  32 bit ARM Cortex-M4 CPU that can be clocked up to 100Mhz
        3.  Many peripherals
    
    **\*\***

