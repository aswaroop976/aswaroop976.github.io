
# Table of Contents

1.  [CHIP-8 emulator in Rust ported to the STM32f411 microcontroller](#orgd6af987)
    1.  [Backend(CHIP-8 emulator)](#orgf923472)



<a id="orgd6af987"></a>

# CHIP-8 emulator in Rust ported to the STM32f411 microcontroller

-   This project was inspired by this blog post: <https://dhole.github.io/post/chip8_emu_2/>
-   Here I will explain how I wrote a CHIP-8 emulator and ported it to an embedded ARM microcontroller, if you are curious about all the code here is the repository linked(ignore the name of the repository): <https://github.com/aswaroop976/ssd1306_test>
-   I will split this post into two parts first talking about the backend, which consists of the CHIP-8 emulator, and then the frontend, which is where I will explain how I ported this emulator to a micrcontroller and interfaced with a display and buttons
-   Additionally this post assumes pre-requisite knowledge of Rust(nothing too complex), and a basic understanding of computer architecture and embedded systems(although I will try to make this as beginner friendly as possible, linking to helpful resources whereever I can)


<a id="orgf923472"></a>

## Backend(CHIP-8 emulator)

-   This part of the project was largely inspired from this blog post(seriously this post probably explains things way better than I ever can): <https://austinmorlan.com/posts/chip8_emulator/>
-   First I defined a Chip8 struct where I store the state of the emulator at any point in time:
    
        pub struct Chip8 {
            pub memory: [u8; MEMORY_SIZE],       // 4kb memory
            pub registers: [u8; REGISTER_COUNT], // 16 general purpose registers
            pub index_register: u16,
            pub program_counter: u16,
            pub screen: [u8; SCREEN_WIDTH * SCREEN_HEIGHT], // 64x32 pixel display
            pub delay_timer: u8,
            pub sound_timer: u8,
            pub stack: [u16; STACK_SIZE], // stack with 16 levels
            pub stack_pointer: u8,        // stack pointer
            pub keys: [u8; REGISTER_COUNT],
            pub jump_table: [OpcodeHandler; 16],
        }

