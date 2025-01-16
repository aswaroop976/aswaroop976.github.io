
# Table of Contents

1.  [CHIP-8 emulator in Rust ported to the STM32f411 microcontroller](#org94201a5)
    1.  [Backend(CHIP-8 emulator)](#org58e78fc)
        1.  [Chip8 struct](#org87dc63f)
        2.  [Opcodes](#org12af38b)
        3.  [Emulation cycle](#orge5b003f)



<a id="org94201a5"></a>

# CHIP-8 emulator in Rust ported to the STM32f411 microcontroller

-   This project was inspired by this blog post: <https://dhole.github.io/post/chip8_emu_2/>
-   Here I will explain how I wrote a CHIP-8 emulator and ported it to an embedded ARM microcontroller, if you are curious about all the code here is the repository linked(ignore the name of the repository): <https://github.com/aswaroop976/ssd1306_test>
-   I will split this post into two parts first talking about the backend, which consists of the CHIP-8 emulator, and then the frontend, which is where I will explain how I ported this emulator to a micrcontroller and interfaced with a display and buttons
-   Additionally this post assumes pre-requisite knowledge of Rust(nothing too complex), and a basic understanding of computer architecture and embedded systems(although I will try to make this as beginner friendly as possible, linking to helpful resources whereever I can)


<a id="org58e78fc"></a>

## Backend(CHIP-8 emulator)

-   This part of the project was largely inspired from this blog post(seriously this post probably explains things way better than I ever can): <https://austinmorlan.com/posts/chip8_emulator/>


<a id="org87dc63f"></a>

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

    -   return<sub>stack</sub>: This essentially serves as the return address stack. A stack depth of 16 limits the number or recursive function calls to 16. The stack<sub>pointer</sub> field points to the top of the return address stack.
    -   memory: the CHIP-8 contains a 4kb memory address space, which requires 16 bits to fully address. However the general purpose registers are only 8 bits wide, to account for this there is a special 16 bit index register. The index register is used specifically to store memory addresses
    -   screen: This represents a seperate memory buffer to drive the 64x32 CHIP-8 display. The display is monochrome(black and white), so the pixel values are either 1(on), or 0(off).
    -   keys: CHIP-8 has 16 input keys, the first 16 hexadecimal values: 0 through F, each key is pressed or not
    -   jump<sub>table</sub>: This is where we will process opcodes and use function pointers to jump to appropriate opcode handler fuction for every instruction that we read from ROM files


<a id="org12af38b"></a>

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


<a id="orge5b003f"></a>

### Emulation cycle

