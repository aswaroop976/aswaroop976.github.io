
# Table of Contents

1.  [CHIP-8 emulator in Rust ported to the STM32f411 microcontroller](#orgd96146c)
    1.  [Backend(CHIP-8 emulator)](#orgc07405e)
        1.  [Chip8 struct](#org552e0a7)
        2.  [Opcodes](#org41a8dbe)
        3.  [Emulation cycle](#org2caf71a)
    2.  [Frontend(Microcontroller port)](#orgc73ffeb)
        1.  [Hardware](#org7a96427)
        2.  [Software](#orgab6d1b5)



<a id="orgd96146c"></a>

# CHIP-8 emulator in Rust ported to the STM32f411 microcontroller

-   This project was inspired by this blog post: <https://dhole.github.io/post/chip8_emu_2/>
-   Here I will explain how I wrote a CHIP-8 emulator and ported it to an embedded ARM microcontroller, if you are curious about all the code here is the repository linked(ignore the name of the repository): <https://github.com/aswaroop976/ssd1306_test>
-   I will split this post into two parts first talking about the backend, which consists of the CHIP-8 emulator, and then the frontend, which is where I will explain how I ported this emulator to a micrcontroller and interfaced with a display and buttons
-   Additionally this post assumes pre-requisite knowledge of Rust(nothing too complex), and a basic understanding of computer architecture and embedded systems(although I will try to make this as beginner friendly as possible, linking to helpful resources whereever I can)


<a id="orgc07405e"></a>

## Backend(CHIP-8 emulator)

-   This part of the project was largely inspired from this blog post(seriously this post probably explains things way better than I ever can): <https://austinmorlan.com/posts/chip8_emulator/>


<a id="org552e0a7"></a>

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


<a id="org41a8dbe"></a>

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


<a id="org2caf71a"></a>

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

-   After loading the ROM into the Chip8&rsquo;s memory(expanded on below), I fetch the current instruction using the program counter(this points to the instruction to be fetched). Then I increment the program counter to point to the next instruction to be fetched(for control/branch instructions the opcode handler will set the program counter accordingly). Then using the opcode from the instruction fetched I can get the relevant opcode handler from the jump table.
-   Below I show the load program function which is how I load a buffer containing the ROM instructions into the chip8&rsquo;s memory(instructions from the ROM shoud be stored in the range 0x200 to 0xFFF in the chip8 memory):
    
        pub fn load_program(&mut self, program: &[u8]) {
            for (i, &byte) in program.iter().enumerate() {
                self.memory[0x200 + i] = byte;
            }
        }


<a id="orgc73ffeb"></a>

## Frontend(Microcontroller port)


<a id="org7a96427"></a>

### Hardware

1.  STM32F411

    ![img](/black_pill.jpg)
    
    -   I chose the STM32F411, successor to the popular &ldquo;blue-pill&rdquo; STM32 micrcontroller, because of it&rsquo;s robust Rust support and powerful internals(for a microcontroller)
    -   The specs include:
        1.  512 Kb of flash memory, 128 Kb of SRAM
        2.  32 bit ARM Cortex-M4 CPU that can be clocked up to 100Mhz
        3.  Many peripherals

2.  SSD1306 display

    ![img](/ssd1306.jpg)
    
    -   I chose this display because of its cheap price and robust Rust support


<a id="orgab6d1b5"></a>

### Software

-   This was a learning experience with using Rust for an embedded system, as such I will document the libraries/crates I used, as well discoveries I made about writing bare-metal Rust

1.  Crates used

    -   stm32f4xx-hal: This is the main crate I used to interact with the stm32f411, as it is a multi-device hardware abstraction layer for all STM32F4 series microcontrollers. This crate provided an API to interface with the peripherals on the microcontroller, such as the GPIO pins, timers, clock, SPI, etc.
    -   cortex-m-rt: This crate contains all the required parts to build an application containing no standard library, that targets a Cortex-M microcontroller. I used it to define an entry point of the program(the main function)
    -   ssd1306: This crate provided a driver interface with the ssd1306 display, it supports both I2C and SPI. I used I2C for this project
    -   embedded-graphics: This crate is a 2D graphics library for memory constrained embedded devices. I used this crate(alongside the ssd1306 crate) as a helpful abstraction to draw individual points on the screen

2.  Code overview

    -   This is how I setup the clock, GPIO pins, and the display initially in the main function:
        
            fn main() -> ! {
                rtt_init_print!();
                let dp = pac::Peripherals::take().unwrap();
                // Set up the system clock.
                let rcc = dp.RCC.constrain();
                let clocks = rcc.cfgr.sysclk(100.MHz()).freeze();
            
                // Set up I2C - SCL is PB6 and SDA is PB7; they are set to Alternate Function 4
                let gpiob = dp.GPIOB.split();
                let scl = gpiob.pb6.internal_pull_up(true);
                let sda = gpiob.pb7.internal_pull_up(true);
                // Configure PB0 as an output pin (connected to one side of the button)
                let mut output_pin: PB0<Output<PushPull>> = gpiob.pb0.into_push_pull_output();
                // Configure PB1 as an input pin as pull down input
                let input_pin: PB1<Input> = gpiob.pb1.into_pull_down_input();
            
                output_pin.set_high();
                let i2c = dp.I2C1.i2c((scl, sda), 400.kHz(), &clocks);
            
                // Set up the display
                let interface = I2CDisplayInterface::new(i2c);
                let mut disp = Ssd1306::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
                    .into_buffered_graphics_mode();
                disp.init().unwrap();
                disp.flush().unwrap();
            
                let mut chip8 = Chip8::new();
            
                // Load ROM ================================================================
                const CHIP8_ROM: &[u8] = include_bytes!("../Chip8 Picture.ch8");
                // Load the program into the CHIP-8 emulator
                chip8.load_program(&CHIP8_ROM);
        
        -   I set the clock frequency to about 100Mhz here, and use pins pb6 and pb7 to drive the I2C output. pb6 serves as the serial clock line(SCL) and pb7 serves as serial data line(SDA).
        -   The I2C speed mode I set the pins to is Fast mode considering I am driving the I2C display at 400kHZ(the max rate for this microcontroller). This means the transmission speed is approximately 400kbps
            -   Using the ssd1306 crate I am able to setup my display, using the I2C interface I setup earlier
        -   I am currently using a singular button(even though CHIP-8 has 16 buttons, I didn&rsquo;t have enough wires to connect all 16 buttons😭)
            -   Here pressing the button closes a circuit between pin pb0 and pb1, where pin pb0 is outputting a voltage(3.3v). Therefore whenever pin pb0(set to be a pull-down input) detects a voltage, we detect a button press. Very useful if I had more wires however to connect all 16 buttons on my makeshift keypad.
        -   I took .ch8 binaries that I found online, and using include bytes was able to write the instructions into a buffer, which I then load into the chip8&rsquo;s memory using the load program function(more info on this on the chip8 section)
        -   Below I present the main loop that runs after the system is setup and initialized:
            
                loop {
                    // Emulate cycle:
                    chip8.emulate_cycle();
                    // Draw logic here =====================================================
                    disp.flush().unwrap();
                    for (i, &pixel) in chip8.screen.iter().enumerate() {
                        if pixel == 1 {
                            let x = 32 + ((i % SCREEN_WIDTH) as i32);
                            let y = 17 + (i / SCREEN_WIDTH) as i32;
                            Pixel(Point::new(x, y), BinaryColor::On)
                                .draw(&mut disp)
                                .unwrap();
                        }
                    }
                }
            
            -   Here I emulate the chip8 clock cycle by calling the emulate cycle and draw to the display
            -   To draw to the screen I iterate through the screen buffer in the chip8 struct and using the embedded-graphics crate turn each pixel on the ssd1306 on/off based on the value in the screen buffer.

