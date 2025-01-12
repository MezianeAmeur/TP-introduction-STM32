
# ***Project: Bouncing ENSEA Logo***

***Objective:***

The goal of this project Implement a system that makes the ENSEA logo bounce continuously on the HDMI output, mimicking the effect seen in DVD players.

<img width="1448" alt="Capture d’écran 2024-12-02 à 13 02 07" src="https://github.com/user-attachments/assets/6e2e5f8d-1111-4521-9372-da1308f49e7c">


## ***1. HDMI Controller***

***Objective:***

The HDMI controller generates the necessary signals for the HDMI transmitter. Additionally, it will produce signals required by the circuitry responsible for image generation.

***1. Resources and Initial Setup:***

Starting by retrieving all the required resources from Moodle. These resources included:

 - A Quartus project with the pinout already configured.
 - `DE10_Nano_HDMI_TX.vhd`: The top-level module of the project that defines the inputs and outputs.
 - `I2C_HDMI_Config.v` and `I2C_Controller.v`: These files handle the configuration of the HDMI transmitter and are already instantiated in the top module.
 - `hdmi_generator.vhd`: This file is partially implemented and needs to be completed. It is also partially instantiated in the top module.
 - 
***2. Analysis of the "hdmi_generator" Entity***
   
   **- Role of the Different Parameters & Signals :**

   
```vhdl
entity hdmi_generator is
   generic (
              -- Resolution
       h_res : natural := 720; -- Horizontal Resolution = number of pixels per line (width of the image). Unit : Pixels
       v_res : natural := 480; -- Vertical Resolution = number of lines (height of the image). Unit : Pixels

              -- Timings magic values (480p)

       h_sync : natural := 61; -- Horizontal Sync: the time the display electronics wait for the next line of image data.. Unit : Number of pixels
       h_fp : natural := 58; -- Horizontal Front Porch: duration of the space between the end of the active image line and the start of the horizontal sync. Unit: Number of pixels
       h_bp : natural := 18; -- Horizontal Back Porch: space between the horizontal sync and the beginning of the active image line. Unit: Number of pixels
       v_sync : natural := 5; -- Vertical Sync: the time needed to switch from one image to the next. Unit: Number of lines
       v_fp : natural := 30; -- Vertical Front Porch : the vertical space between the end of the active image and the start of the vertical sync. Unit: Number of lines
       v_bp : natural := 9 -- Vertical Back Porch: the space between the vertical sync and the beginning of the active image. Unit: Number of lines
    );
    port (
       i_clk : in std_logic; -- Input clock signal: synchronizes the other signals
       i_reset_n : in std_logic; -- Input reset signal: re-initializes the entity when needed
       o_hdmi_hs : out std_logic; -- Horizontal sync signal for HDMI: defines the horizontal sync period
       o_hdmi_vs : out std_logic; -- Vertical sync signal for HDMI: defines the vertical sync period
       o_hdmi_de : out std_logic; -- Data enable: indicates whether the pixel data is valid/active during the transmission of each pixel
       o_pixel_en : out std_logic;  -- Pixel Enable: to enable or disable the display of pixels
       o_pixel_address : out natural range 0 to (h_res * v_res - 1); -- contains the pixel address calculated using: o_pixel_address = o_pixel_pos_x + (h_res × o_pixel_pos_y)
       o_x_counter : out natural range 0 to (h_res - 1); -- measures the horizontal position of the pixels during the video signal generation
       o_y_counter : out natural range 0 to (v_res - 1); -- measures the vertical position of the pixels during the video signal generation
       o_pixel_pos_x : out natural range 0 to (h_res - 1); -- Positions of pixels in X (horizontal) within the active display area
       o_pixel_pos_y : out natural range 0 to (v_res - 1); -- Positions of pixels in Y (vertical) within the active display area
       o_new_frame : out std_logic -- goes high during 1 clk cycle when a new image is ready to be transmitted
    );
end hdmi_generator;
```


### ***1.1. Writing the Component***

**1. Horizontal Counter (h_count) and Horizontal Sync Signal:** the objective is to implement of a horizontal counter (h_count) that loops from 0 to h_total and generates the horizontal sync signal (o_hdmi_hs).

**2. Testing the Horizontal Counter:** Simulation of the horizontal counter using a testbench to verify functionality.


**3. Vertical Counter (v_count) and Vertical Sync Signal:** the objective is to add a vertical counter (v_count) that increments on each complete cycle of the horizontal counter (h_count) and loops from 0 to v_total and generates the vertical sync signal (o_hdmi_vs).


**4. Testing Both Counters:** simulatation of both horizontal and vertical counters together.


**5. Defining the Active Display Zone:** the objective is to identify the ranges of h_count and v_count where the pixels are visible (active zone) and implement signals h_act and v_act to indicate active regions.


**6. Generating the Data Enable Signal (o_hdmi_de):** the objective is to combine h_act and v_act to produce a signal (o_hdmi_de) that indicates when the generator is in an active display zone.


**7. Pixel Address Generation:** the objective is to add an active pixel counter (r_pixel_counter) to generate a pixel address (o_pixel_address) that increments only in active zones and reset the counter at the start of a new frame.


**8. Active Pixel Counters for X and Y Positions:** the objective is to add two additional counters (o_x_counter and o_y_counter) to track the X and Y positions of active pixels.


**9. Final Testing of the Component:** simulation of the complete HDMI generator to verify all functionalities, including synchronization signals, active zones, and pixel addressing.

***The Final Code:***

```vhdl

library ieee;
use ieee.std_logic_1164.all;

entity projet_fpga is
    generic (
        -- Resolution
        h_res : natural := 720;
        v_res : natural := 480;
        -- Timings magic values (480p)
        h_sync : natural := 61;
        h_fp : natural := 58;
        h_bp : natural := 18;
        v_sync : natural := 5;
        v_fp : natural := 30;
        v_bp : natural := 9
    );
    port (
        i_clk : in std_logic;
        i_reset_n : in std_logic;
        o_hdmi_hs : out std_logic;
        o_hdmi_vs : out std_logic;
        o_hdmi_de : out std_logic;  -- PAS DE NEGOCIATION 
        o_pixel_en : out std_logic;
        o_pixel_address : out natural range 0 to (h_res * v_res -1);
        o_x_counter : out natural range 0 to (h_res - 1);
        o_y_counter : out natural range 0 to (v_res - 1);
        o_pixel_pos_x : out natural range 0 to (h_res - 1);
        o_pixel_pos_y : out natural range 0 to (v_res - 1);
        o_new_frame : out std_logic
    );
end projet_fpga;

architecture rtl of projet_fpga is
signal h_count : natural range 0 to (h_res + h_sync + h_fp + h_bp - 1) := 0;
constant h_total : natural := h_res + h_sync + h_fp + h_bp;

signal v_count : natural range 0 to (v_res + v_sync + v_fp + v_bp - 1) := 0;
constant v_total : natural := v_res + v_sync + v_fp + v_bp;

signal h_act : std_logic;
signal v_act : std_logic;

signal r_pixel_counter : natural range 0 to (h_res*v_res - 1) := 0;

signal x_counter : natural range 0 to (h_res - 1) := 0;
signal y_counter : natural range 0 to (v_res - 1) := 0;
begin

process(i_clk, i_reset_n)
begin
	if i_reset_n = '0' then
		h_count <= 0;
		o_hdmi_hs <= '1';
		
		v_count <= 0;
		o_hdmi_vs <= '1';
		
		h_act <= '0';
		v_act <= '0';
		
		o_hdmi_de <= '0';
		r_pixel_counter <= 0;
		o_pixel_address <= 0;
		
		
		x_counter <= 0;
		y_counter <= 0;
		
	elsif rising_edge(i_clk) then
			if h_count = h_total - 1  then
				h_count <= 0;
				o_hdmi_hs <= '1';
				
				if v_count = v_total - 1 then
					v_count <= 0;
					o_hdmi_vs <= '1';
					r_pixel_counter <= 0;
					x_counter <= 0;
		         y_counter <= 0;
					
				else 
					v_count <= v_count + 1;
					o_hdmi_vs <= '0';
					
				
				end if;
				
				
				
			else 
				h_count <= h_count + 1;
				o_hdmi_hs <= '0';
				o_hdmi_vs <= '0';
				
			
			if ( h_count >= h_sync + h_bp ) and ( h_count < h_res + h_sync + h_fp ) then
				h_act <= '1';
				if x_counter < h_res - 1 then
					x_counter <= x_counter + 1;
					
			else
				h_act <= '0';
				x_counter <= 0;
			end if;
			end if;
			
			if (v_count >= v_sync + v_bp) and (v_count < v_res + v_sync + v_fp) then
				v_act <= '1';
				if y_counter < v_res - 1 then
					y_counter <= y_counter + 1;
			else
				v_act <= '0';
				y_counter <= 0;
			end if;
			end if;
		
			o_hdmi_de <= h_act and v_act;
			if (h_act = '1') and (v_act = '1') then 
				r_pixel_counter <= r_pixel_counter + 1; 
				
				end if;
				
			o_pixel_address <= r_pixel_counter;
			o_x_counter <= x_counter;
			o_y_counter <= y_counter;
			end if;	
		end if;	
	end process;
end rtl;


```

***The Final Test Bench Code:***

```vhd

library ieee;
use ieee.std_logic_1164.all;

entity projet_fpga_tb is
end projet_fpga_tb;

architecture tb of projet_fpga_tb is
    -- Signaux internes
    signal clk : std_logic := '0';
    signal reset_n : std_logic := '0';
    signal o_hdmi_hs : std_logic;
	 signal o_hdmi_vs : std_logic;
	signal o_hdmi_de : std_logic;
	
		signal o_pixel_address : natural;
		signal o_x_counter : natural;
		signal o_y_counter : natural;
    -- Constantes pour les tests
    constant clk_period : time := 20 ns; 

begin

    uut: entity work.projet_fpga
        generic map (
            h_res => 8,
            v_res => 6,
            h_sync => 1,
            h_fp => 1,
            h_bp => 1,
            v_sync => 1,
            v_fp => 1,
            v_bp => 1
        )
        port map (
            i_clk => clk,
            i_reset_n => reset_n,
            o_hdmi_hs => o_hdmi_hs,
            o_hdmi_vs => o_hdmi_vs,
            o_hdmi_de => o_hdmi_de,
            o_pixel_en => open,
            o_pixel_address => o_pixel_address,
            o_x_counter => o_x_counter,
            o_y_counter => o_y_counter,
            o_pixel_pos_x => open,
            o_pixel_pos_y => open,
            o_new_frame => open
        );

    -- Génération de l'horloge
    clk_gen : process
    begin
        while true loop
            clk <= not clk;
            wait for 10 ns;
        end loop;
    end process;

    -- Génération de la réinitialisation
    reset_gen : process
    begin
        reset_n <= '0';
        wait for 0 ns;
        reset_n <= '1';
        wait;
    end process;
end tb;

```

To simulate the code we used ModelSim to run directly the .do File : 

```tcl

vcom projet_fpga.vhd
vcom projet_fpga_tb.vhd

vsim -c work.projet_fpga_tb

# Ajouter les signaux d'entrée et de sortie
add wave -divider Inputs
add wave -color yellow /projet_fpga_tb/clk
add wave -color yellow /projet_fpga_tb/reset_n
add wave -color green uut/h_count
add wave -color green uut/v_count

add wave -divider Outputs
add wave -color cyan /projet_fpga_tb/o_hdmi_hs
add wave -color cyan /projet_fpga_tb/o_hdmi_vs
add wave -color cyan /projet_fpga_tb/o_hdmi_de
add wave -color cyan /projet_fpga_tb/o_pixel_address
add wave -color cyan /projet_fpga_tb/o_x_counter
add wave -color cyan /projet_fpga_tb/o_y_counter

add wave -divider Internals
add wave uut/h_act
add wave uut/v_act
add wave uut/r_pixel_counter
add wave uut/x_counter
add wave uut/y_counter
run 10 us

```

***The simulation result*** 

![IMG_9366](https://github.com/user-attachments/assets/ac703404-874a-44b6-b303-b4864423a7cf)

**Description:** 


### 1. Clock and Reset Signals
- The design is driven by an external clock input (`i_clk`), which synchronizes all operations, including counters and state transitions.
- The reset signal (`i_reset_n`) ensures all internal signals and counters are initialized to a known state. It is active low and must be set to high (`'1'`) to enable normal operation.

### 2. Horizontal and Vertical Counters
- **Horizontal Counter (`h_count`)**:
  - Increments with each clock cycle.
  - Resets to zero after reaching the total horizontal timing (`h_total`), which includes the active resolution (`h_res`), sync pulse (`h_sync`), front porch (`h_fp`), and back porch (`h_bp`).

- **Vertical Counter (`v_count`)**:
  - Increments after the horizontal counter completes a line.
  - Resets to zero after reaching the total vertical timing (`v_total`), which includes the active resolution (`v_res`), sync pulse (`v_sync`), front porch (`v_fp`), and back porch (`v_bp`).

### 3. Synchronization Signals
- **Horizontal Sync (`o_hdmi_hs`)**:
  - Generated based on `h_count`.
  - Active low (`'0'`) during the horizontal sync pulse and high (`'1'`) otherwise.

- **Vertical Sync (`o_hdmi_vs`)**:
  - Generated based on `v_count`.
  - Active low (`'0'`) during the vertical sync pulse and high (`'1'`) otherwise.

### 4. Active Video Region
- The active display region is determined by the horizontal and vertical counters:
  - **Horizontal Active (`h_act`)**: Active (`'1'`) when `h_count` is within the range of the active horizontal resolution (`h_sync + h_bp <= h_count < h_sync + h_bp + h_res`).
  - **Vertical Active (`v_act`)**: Active (`'1'`) when `v_count` is within the range of the active vertical resolution (`v_sync + v_bp <= v_count < v_sync + v_bp + v_res`).

- The **data enable signal (`o_hdmi_de`)** is asserted (`'1'`) during the active video region, combining `h_act` and `v_act`.

### 5. Pixel Address and Counters
- **Pixel Address (`o_pixel_address`)**:
  - A linear index of the active pixels, incremented only during the active video region.

- **Pixel Position Counters**:
  - **`o_x_counter`**: Represents the horizontal pixel position. It increments within each line and resets at the end of the line.
  - **`o_y_counter`**: Represents the vertical pixel position. It increments after each line and resets at the end of the frame.

### 6. Frame Generation
- At the end of each frame (when both `h_count` and `v_count` reset), the **new frame signal (`o_new_frame`)** is asserted, indicating the start of a new frame.

![IMG_9365](https://github.com/user-attachments/assets/25c2cfee-0242-4c2e-bf51-ba9eb609f84c)

---


### ***1.2. FPGA Implementation***

**1. Connecting Signals to Color Channels:** assigning the counters to color outputs:

 - Green Channel: Connect o_x_counter
 - Blue Channel: Connect o_y_counter


**2. Testing the HDMI Output:**

The HDMI output was tested and reviewed by the professor!

## ***2. Bouncing ENSEA Logo***


### 2. Image Display Implementation
- VHDL code was written to render the logo at the top-left corner of the screen : 

![WhatsApp Image 2024-12-22 à 13 11 52_8782d056](https://github.com/user-attachments/assets/55480e98-1470-49fd-accc-8c6c329a8a42)





