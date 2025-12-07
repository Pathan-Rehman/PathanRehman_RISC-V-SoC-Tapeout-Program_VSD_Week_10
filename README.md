# Caravel Functional-vs-GLS Verification Task

# Objective:
Verify that Caravel’s RTL simulation and Gate-Level Simulation (GLS) produce identical functional results for the hkspi test and subsequently for all other DV tests inside efabless/caravel/verilog/dv.

# Tools Allowed:
Icarus Verilog or Verilator, OpenLane + OpenROAD, Magic/KLayout, Sky130 PDK, GitHub Codespaces or local system.

# Setting Up vsdip/vsd-rtl github codespace

## RTL Design & Synthesis on Cloud (GitHub Codespace)

This repository provides a ready-to-use **cloud-based lab** for RTL Design and Synthesis using open-source tools such as **Yosys**, **Icarus Verilog**, and **GTKWave**.
All tools run inside a **GitHub Codespace** with **noVNC desktop access**, requiring no local installation.

---

### Step 1 – Launch Codespace

Click **“Code → Codespaces → Create codespace on main”** to start your workspace on the cloud.
GitHub will automatically build and set up your environment.

<img width="592" height="547" alt="image" src="https://github.com/user-attachments/assets/f27d6058-1c11-4298-9b69-129d818a0a66" />

---

### Step 2 – Codespace Setup and Logs

During setup, the Codespace installs all required tools.
Wait for the setup logs to complete (approximately 7–10 minutes).

<img width="801" height="180" alt="image" src="https://github.com/user-attachments/assets/9c291840-ed01-42d9-bb4e-905d166a20d0" />

After successful configuration, your container environment is ready.

<img width="1906" height="922" alt="image" src="https://github.com/user-attachments/assets/3a25234e-877c-4dae-b3ee-d39c7c558d98" />

---

### Step 3 – Open a Terminal

Use **Terminal → New Terminal** inside VS Code to begin executing synthesis and simulation commands.

<img width="1744" height="738" alt="image" src="https://github.com/user-attachments/assets/a867028c-aee4-4e62-970e-6174950da887" />

---

### Step 4 – Verify Tool Installation

Run the following commands to verify tool installation:

```bash
yosys
iverilog
```

Both should display their version information, confirming correct setup.

<img width="1160" height="1400" alt="image" src="https://github.com/user-attachments/assets/c2198c2f-0263-4e1d-b58b-bb60b48cd544" />

---

# Caravel hkspi Functional vs GLS Verification

## Objective

In this work, I verified that Caravel’s RTL simulation and gate‑level simulation (GLS) produce identical functional results for the `hkspi` test using the Sky130A PDK and open‑source tools (Icarus Verilog, volare, git, etc.). I documented all commands I used, the errors I faced, and how I resolved them, ending with a register‑by‑register comparison between RTL and GLS that matched. 

***

## Environment Setup

### Creating workspace and cloning Caravel

```bash
mkdir -p caravel_vsd
cd ./caravel_vsd/
git clone https://github.com/efabless/caravel
cd caravel
```

<img width="887" height="271" alt="image" src="https://github.com/user-attachments/assets/2be0516a-6e42-4727-836a-8c7ec8fc9146" />


- `mkdir -p caravel_vsd`: I created a working directory `caravel_vsd` to keep all Caravel‑related files organized; `-p` ensures no error if the directory already exists. 
- `cd ./caravel_vsd/`: I moved into the workspace folder so all subsequent operations stay local to this project. 
- `git clone https://github.com/efabless/caravel`: I cloned the official efabless Caravel repository that contains RTL, GL netlists, DV tests, and scripts needed for the hkspi verification task. 
- `cd caravel`: I entered the cloned Caravel repository root to run all project‑specific commands from there. 

***

### Initializing git submodules

```bash
git submodule update --init --recursive
```

- I initialized and updated all git submodules (for example, some IPs and dependencies which are brought in as submodules) to ensure the repository was completely populated. 
- `--recursive` ensured nested submodules, if any, were also fetched, avoiding missing‑directory errors later in the flow. 

***

### Installing and enabling Sky130 PDK with volare

```bash
pip install volare --user --upgrade
export PATH=$PATH:$HOME/.local/bin
export CARAVEL_ROOT=$(pwd)
export PDK_ROOT=$(pwd)/pdk
export PDK=sky130A
volare enable 12df12e2e74145e31c5a13de02f9a1e176b56e67
```

- `pip install volare --user --upgrade`: I installed (or upgraded) the `volare` PDK management tool in my user environment; it manages Sky130 PDK versions and caches. 
- `export PATH=$PATH:$HOME/.local/bin`: I added the local Python bin directory to `PATH` so that the `volare` executable is found in the shell. 
- `export CARAVEL_ROOT=$(pwd)`: I defined `CARAVEL_ROOT` as the absolute path of the Caravel repo root for use in subsequent commands and include paths. 
- `export PDK_ROOT=$(pwd)/pdk`: I selected `./pdk` under the Caravel root as the installation/cache location for the Sky130 PDK. 
- `export PDK=sky130A`: I specified that the active PDK is `sky130A` so scripts and Makefiles know which PDK flavour to use. 
- `volare enable 12df12e2e74145e31c5a13de02f9a1e176b56e67`: I enabled a specific Sky130A PDK commit/version; volare downloaded the required archives (standard cells, IOs, SRAM, etc.) and made them accessible under `$PDK_ROOT`. 

<img width="1093" height="643" alt="image" src="https://github.com/user-attachments/assets/aa82546c-74e1-4e64-96bf-faf031d848c0" />
<img width="1107" height="315" alt="image" src="https://github.com/user-attachments/assets/1611a0d0-eb0a-4385-9cef-a91fdf507f17" />

***

## hkspi Test RTL and GLS Context

According to the task document, the hkspi test verifies the housekeeping SPI interface between the management SoC and a SPI flash model, reading back registers and checking expected values. The RTL side uses `hkspi_tb.v`, `housekeeping_spi.v` and `hkspi.hex` to drive the management SoC and validate the SPI protocol and register map. In GLS, the same testbench is reused, but the DUT is replaced with gate‑level netlists plus standard‑cell libraries from the Sky130A PDK, and the expected behavior must remain identical. 

***

## First attempt to patch VexRiscv (failed)

```bash
export CARAVEL_ROOT=$(pwd)
export PDK_ROOT=$(pwd)/pdk
export VEX_FILE=$(find verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)
sed -i 's/module VexRiscv (/module VexRiscv ( inout vccd1, inout vssd1, /' $VEX_FILE
```

- `export VEX_FILE=...find...VexRiscv*.v`: I tried to auto‑locate the VexRiscv CPU RTL file inside `verilog/rtl/mgmt_core_wrapper` to later add explicit power pins if needed. 
- `sed -i 's/module VexRiscv.../' $VEX_FILE`: I attempted to edit the VexRiscv module declaration to add `vccd1` and `vssd1` power pins inline. 

**Error faced**

```text
find: 'verilog/rtl/mgmt_core_wrapper': No such file or directory
sed: no input files
```

- The directory `verilog/rtl/mgmt_core_wrapper` did not exist in the initial Caravel checkout, so `find` failed to locate `VexRiscv*.v`, and consequently `sed` had no input file.

<img width="1109" height="113" alt="image" src="https://github.com/user-attachments/assets/2e025045-ba62-4f17-97b7-bdc0edacc8e3" />


**How I solved it**

Later, I cloned the management SoC wrapper from the `caravel_mgmt_soc_litex` repository into `verilog/rtl/mgmt_core_wrapper`, which created the required directory and VexRiscv sources. 

***

## RTL hkspi simulation (caravel_netlists include problem)

### Moving into hkspi test directory

```bash
cd verilog/dv/caravel/mgmt_soc/hkspi
```

- I navigated to the hkspi DV test directory where `hkspi_tb.v` and associated DV infrastructure reside. 

### RTL iverilog compilation (first attempt)

```bash
iverilog -Ttyp \
-DFUNCTIONAL -DSIM \
-D USE_POWER_PINS \
-D UNIT_DELAY=#1 \
-I $CARAVEL_ROOT/verilog/rtl \
-I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
-I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
-I $CARAVEL_ROOT/verilog/dv/caravel \
-I $PDK_ROOT/sky130A \
-y $CARAVEL_ROOT/verilog/rtl \
-y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
hkspi_tb.v -o hkspi.vvp
```

- `iverilog -Ttyp`: I compiled the design using Icarus Verilog, targeting a typical timing model. 
- `-DFUNCTIONAL -DSIM`: I defined `FUNCTIONAL` and `SIM` macros so that the RTL and libraries compile in functional, simulation‑friendly mode. 
- `-D USE_POWER_PINS`: I enabled sections of RTL that expect explicit power pins, matching the Sky130 cell models. 
- `-D UNIT_DELAY=#1`: I set a unit delay macro to `#1` used in some cell models and wrappers to model minimal delays. 
- `-I ...`: I added multiple include directories for RTL, management core wrapper, and DV files so that `\`include` directives resolve.[file:1]  
- `-y ...`: I pointed Icarus to library directories for module auto‑search, including main RTL and management core wrapper RTL. 
- I explicitly compiled primitive and standard‑cell Verilog models (`primitives.v`, `sky130_fd_sc_hd.v`) plus IO library (`sky130_fd_io.v`) from the Sky130A PDK. 
- `hkspi_tb.v -o hkspi.vvp`: I compiled the hkspi testbench and produced an output simulation file `hkspi.vvp`.

<img width="1093" height="423" alt="image" src="https://github.com/user-attachments/assets/cf504246-d05c-4d36-9a5d-3e766c9afc50" />


**Error faced**

```text
/workspaces/.../verilog/rtl/caravel_netlists.v:95: Include file mgmt_core_wrapper.v not found
```

- The file `caravel_netlists.v` referenced `mgmt_core_wrapper.v` directly in the same directory, but no such file existed at that location yet. 

**How I mitigated it (later)**

After bringing in `mgmt_core_wrapper` from the `caravel_mgmt_soc_litex` repo and adjusting include paths and include lines, I resolved the missing file issue. 

### Running vvp (no output because compilation failed)

```bash
vvp hkspi.vvp | tee rtl_hkspi.log
```

- `vvp hkspi.vvp`: I attempted to run the compiled RTL simulation. 
- `| tee rtl_hkspi.log`: I piped the console output into a log file `rtl_hkspi.log` for later analysis and comparison.   

Due to the compilation error, this early run was not the final, correct RTL simulation yet. 

***

## Fixing mgmt_core_wrapper: cloning caravel_mgmt_soc_litex

I used a long, chained export/clone/sed command to fix the missing management core wrapper and GL paths:

```bash
export CARAVEL_ROOT=$(pwd | sed 's|/verilog/dv/caravel/mgmt_soc/hkspi||') && \
cd $CARAVEL_ROOT && \
rm -rf verilog/rtl/mgmt_core_wrapper && \
git clone https://github.com/efabless/caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper && \
export PDK_ROOT=$CARAVEL_ROOT/pdk && \
export MGMT_GL_DIR=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "mgmt_core_wrapper.v" | grep "/gl" | head -n 1 | xargs dirname) && \
cd $CARAVEL_ROOT/verilog/rtl && \
sed -i 's|"gl/digital_pll.v"|"digital_pll.v"|g' caravel_netlists.v && \
sed -i 's|"gl/gpio_control_block.v"|"gpio_control_block.v"|g' caravel_netlists.v && \
sed -i 's|"gl/gpio_signal_buffering.v"|"gpio_signal_buffering.v"|g' caravel_netlists.v && \
echo "module sky130_ef_sc_hd__fill_4(inout VPWR, inout VGND, inout VPB, inout VNB); endmodule" > $CARAVEL_ROOT/verilog/dv/dummy_fill.v && \
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
```

- `export CARAVEL_ROOT=...sed...`: From inside the hkspi directory, I derived the repository root by stripping the DV path suffix using `sed`. 
- `rm -rf verilog/rtl/mgmt_core_wrapper`: I removed any old or partial `mgmt_core_wrapper` directory to avoid conflicts. 
- `git clone ...caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper`: I cloned the `caravel_mgmt_soc_litex` project directly into `verilog/rtl/mgmt_core_wrapper`, bringing in the complete LiteX management SoC including `mgmt_core_wrapper.v` and VexRiscv. 
- `export MGMT_GL_DIR=$(find ...mgmt_core_wrapper.v | grep "/gl" ... xargs dirname)`: I located the directory that contains the gate‑level `mgmt_core_wrapper.v` (under some `gl` subdirectory) so I could include it during GLS compilation.   
- `sed -i 's|"gl/digital_pll.v"|"digital_pll.v"|g' caravel_netlists.v`: I edited `caravel_netlists.v` so that it referenced non‑`gl/` versions of some modules, making the include paths match the actual files I intended to use.   
- I repeated similar `sed` replacements for `gpio_control_block` and `gpio_signal_buffering` to avoid broken `gl/` paths. 
- `echo "module sky130_ef_sc_hd__fill_4(...)" > ...dummy_fill.v`: I created a dummy definition for a fill cell (`sky130_ef_sc_hd__fill_4`) used in gate‑level netlists but not required functionally, so that compilation would not fail due to missing module definitions. 

***

## First GLS compilation attempts and errors

### 1st GLS compile with iverilog

```bash
iverilog -Ttyp -DFUNCTIONAL -DSIM -DGL -D USE_POWER_PINS -D UNIT_DELAY=#1 \
-I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
-I $CARAVEL_ROOT/verilog/dv/caravel \
-I $CARAVEL_ROOT/verilog/rtl \
-I $CARAVEL_ROOT/verilog \
-I $PDK_ROOT/sky130A \
-I $MGMT_GL_DIR \
-y $CARAVEL_ROOT/verilog/gl \
-y $MGMT_GL_DIR \
-y $CARAVEL_ROOT/verilog/rtl \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
$CARAVEL_ROOT/verilog/dv/dummy_fill.v \
hkspi_tb.v -o hkspi_gl.vvp && \
vvp hkspi_gl.vvp | tee gls_hkspi.log
```

- `-DGL`: I defined `GL` so that GL‑specific sections (e.g., gate‑level netlists, SDF‑related conditionals) would be active. 
- I included both RTL and GL directories as search paths and compiled the standard cell and IO models similarly to the RTL flow. 

**Error faced**

```text
caravel_netlists.v:70: Include file gl/mgmt_defines.v not found
```

- `caravel_netlists.v` still referenced `gl/mgmt_defines.v`, which did not exist in my current GL directory structure, causing GLS compilation to stop. 

### Fixing missing gl/mgmt_defines.v

```bash
export CARAVEL_ROOT=$(pwd | sed 's|/verilog/dv/caravel/mgmt_soc/hkspi||') && \
export PDK_ROOT=$CARAVEL_ROOT/pdk && \
export MGMT_GL_DIR=$(find $CARAVEL_ROOT -name "mgmt_core_wrapper.v" | grep "/gl" | head -n 1 | xargs dirname) && \
cd $CARAVEL_ROOT/verilog/rtl && \
sed -i 's|"gl/digital_pll.v"|"digital_pll.v"|g' caravel_netlists.v && \
sed -i 's|"gl/gpio_control_block.v"|"gpio_control_block.v"|g' caravel_netlists.v && \
sed -i 's|"gl/gpio_signal_buffering.v"|"gpio_signal_buffering.v"|g' caravel_netlists.v && \
sed -i 's|"gl/mgmt_defines.v"|"defines.v"|g' caravel_netlists.v && \
echo "module sky130_ef_sc_hd__fill_4(inout VPWR, inout VGND, inout VPB, inout VNB); endmodule" > $CARAVEL_ROOT/verilog/dv/dummy_fill.v && \
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi && \
iverilog ...
```

- I updated `caravel_netlists.v` again to replace `gl/mgmt_defines.v` with `defines.v`, which exists in the source tree and provides the necessary macro definitions for the management SoC. 

**Next error**

```text
caravel_netlists.v:71: Include file gl/mgmt_core_wrapper.v not found
```

<img width="1096" height="500" alt="image" src="https://github.com/user-attachments/assets/69f3ea63-68c3-4c31-a6b0-08aed75f769d" />


- The netlists still referenced `gl/mgmt_core_wrapper.v`, but my GL directory structure placed `mgmt_core_wrapper.v` under the cloned `mgmt_core_wrapper` tree, not under `verilog/gl/gl/...` as originally expected. 

### Fixing gl/mgmt_core_wrapper.v

```bash
export CARAVEL_ROOT=$(pwd | sed 's|/verilog/dv/caravel/mgmt_soc/hkspi||') && \
export PDK_ROOT=$CARAVEL_ROOT/pdk && \
export MGMT_GL_DIR=$(find $CARAVEL_ROOT -name "mgmt_core_wrapper.v" | grep "/gl" | head -n 1 | xargs dirname) && \
cd $CARAVEL_ROOT/verilog/rtl && \
sed -i 's|"gl/digital_pll.v"|"digital_pll.v"|g' caravel_netlists.v && \
sed -i 's|"gl/gpio_control_block.v"|"gpio_control_block.v"|g' caravel_netlists.v && \
sed -i 's|"gl/gpio_signal_buffering.v"|"gpio_signal_buffering.v"|g' caravel_netlists.v && \
sed -i 's|"gl/mgmt_defines.v"|"defines.v"|g' caravel_netlists.v && \
sed -i 's|"gl/mgmt_core_wrapper.v"|"mgmt_core_wrapper.v"|g' caravel_netlists.v && \
echo "module sky130_ef_sc_hd__fill_4(inout VPWR, inout VGND, inout VPB, inout VNB); endmodule" > $CARAVEL_ROOT/verilog/dv/dummy_fill.v && \
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi && \
iverilog ...
```

- I changed the include for the management core wrapper from `gl/mgmt_core_wrapper.v` to `mgmt_core_wrapper.v` so that it resolved to the correct location within the cloned LiteX management RTL/GL tree. 

These GLS attempts were later terminated manually (`^C` / `Terminated`) as I tuned the setup and runtime behavior. 

***

## Managing runtime and waveform dumping

To keep GLS runs lighter and avoid oversized VCD dumps, I disabled waveform dumping in the testbench:

```bash
sed -i 's/$dumpfile/\/\/ $dumpfile/g' hkspi_tb.v
sed -i 's/$dumpvars/\/\/ $dumpvars/g' hkspi_tb.v
```

<img width="1107" height="69" alt="image" src="https://github.com/user-attachments/assets/1a53bf4f-1324-42e2-bd58-e3d7a5172775" />


- These `sed` commands comment out the `$dumpfile` and `$dumpvars` lines in `hkspi_tb.v` by prefixing them with `//`, effectively disabling waveform generation. 
- This saves disk space and memory during long GLS runs, making simulations more stable. 

I also tried different `iverilog` options:

```bash
iverilog -Ttyp -DFUNCTIONAL -DSIM -DGL -D USE_POWER_PINS -pfileline=1 ...
vvp -N hkspi_gl.vvp | tee gls_hkspi.log
```
<img width="1092" height="140" alt="image" src="https://github.com/user-attachments/assets/a26e5626-73d6-4bd5-9a51-49c95118c6dc" />


- `-pfileline=1`: I enabled file/line printing in runtime errors to help trace issues back to source lines. 
- `vvp -N`: I used `-N` to disable certain interactive features and help keep the run straightforward with logging to stdout. 

Even though some of these GLS runs were terminated, they were part of narrowing down the correct files and configuration. 

***

## Mixed RTL + GL attempt and duplicated module error

At one point, I attempted a mixed compilation including GL `housekeeping.v` manually:

```bash
iverilog -Ttyp -DFUNCTIONAL -DSIM -D USE_POWER_PINS -D UNIT_DELAY=#1 \
-I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
-I $CARAVEL_ROOT/verilog/dv/caravel \
-I $CARAVEL_ROOT/verilog/rtl \
-I $CARAVEL_ROOT/verilog \
-I $PDK_ROOT/sky130A \
-y $CARAVEL_ROOT/verilog/rtl \
-y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
$CARAVEL_ROOT/verilog/gl/housekeeping.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
$CARAVEL_ROOT/verilog/dv/dummy_fill.v \
hkspi_tb.v -o hkspi_mixed.vvp && \
vvp hkspi_mixed.vvp | tee gls_hkspi.log
```

<img width="1095" height="491" alt="image" src="https://github.com/user-attachments/assets/ea433f8e-2321-4bc4-8515-7328722fe9a8" />


**Error faced**

```text
verilog/rtl/housekeeping.v:57: error: 'housekeeping' has already been declared in this scope.
verilog/gl/housekeeping.v:1: : It was declared here as a module.
```

- Both RTL and GL versions of `housekeeping` were compiled simultaneously, causing a duplicate module declaration error. 

**Fix**

I prevented the RTL version from being pulled in via `caravel_netlists.v` and used only the GL `housekeeping.v`:

```bash
cd $CARAVEL_ROOT/verilog/rtl
sed -i 's|`include "housekeeping.v"|// `include "housekeeping.v"|g' caravel_netlists.v
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
```

<img width="1087" height="98" alt="image" src="https://github.com/user-attachments/assets/e98eeb10-d9d5-4e4e-98bc-9ec97c82f96f" />


- This change commented out the `housekeeping.v` RTL include, avoiding the double definition while keeping the explicitly provided GL `housekeeping.v` in the compilation. 

***

## VexRiscv unknown module error and fix

After disabling the RTL `housekeeping` include and compiling again, I hit:

```text
mgmt_core.v:8426: error: Unknown module type: VexRiscv
*** These modules were missing:
        VexRiscv referenced 2 times.
```

- The management core RTL (`mgmt_core.v`) instantiated `VexRiscv`, but the VexRiscv Verilog file itself wasn’t part of the compile command yet. 

**Fix**

I located the VexRiscv file and added it explicitly:

```bash
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)

iverilog -Ttyp -DFUNCTIONAL -DSIM -D USE_POWER_PINS -D UNIT_DELAY=#1 \
-I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
-I $CARAVEL_ROOT/verilog/dv/caravel \
-I $CARAVEL_ROOT/verilog/rtl \
-I $CARAVEL_ROOT/verilog \
-I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
-I $PDK_ROOT/sky130A \
-y $CARAVEL_ROOT/verilog/rtl \
-y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
$CARAVEL_ROOT/verilog/gl/housekeeping.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
$CARAVEL_ROOT/verilog/dv/dummy_fill.v \
$VEX_FILE \
hkspi_tb.v -o hkspi_mixed.vvp && \
vvp hkspi_mixed.vvp | tee gls_hkspi.log
```

<img width="1090" height="448" alt="image" src="https://github.com/user-attachments/assets/50641167-c84b-48f9-8dfe-d0cd83490f81" />


- `export VEX_FILE=...`: I automatically discovered the VexRiscv RTL file path. 
- I appended `$VEX_FILE` to the iverilog command, ensuring the CPU module is compiled and elaborated correctly. 

***

## Handling hkspi.hex readmemh warning

During both RTL and GLS runs, I saw:

```text
ERROR: spiflash.v:114: $readmemh: Unable to open hkspi.hex for reading.
hkspi.hex loaded into memory
```

- The testbench or SPI flash model attempted to use `$readmemh("hkspi.hex", ...)`, but depending on the working directory, the file wasn’t initially found, triggering a warning/error. 
- Despite this message, the test proceeded and indicated that register reads and expected values matched, so functionally the test still passed in my final runs. 

The resolution here was to ensure I ran simulations from the hkspi directory where `hkspi.hex` is available, which allowed the memory to be loaded and the test to proceed correctly. 

***

## Final RTL run and log capture

For the final RTL reference run, I used a command of the form:

```bash
export CARAVEL_ROOT=$(pwd | sed 's|/verilog/dv/caravel/mgmt_soc/hkspi||')
export PDK_ROOT=$CARAVEL_ROOT/pdk
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)

iverilog -Ttyp -DFUNCTIONAL -DSIM -D USE_POWER_PINS -D UNIT_DELAY=#1 \
-I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
-I $CARAVEL_ROOT/verilog/dv/caravel \
-I $CARAVEL_ROOT/verilog/rtl \
-I $CARAVEL_ROOT/verilog \
-I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
-I $PDK_ROOT/sky130A \
-y $CARAVEL_ROOT/verilog/rtl \
-y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
$PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
$CARAVEL_ROOT/verilog/dv/dummy_fill.v \
$VEX_FILE \
hkspi_tb.v -o hkspi.vvp && \
vvp hkspi.vvp | tee rtl_hkspi.log
```

<img width="1082" height="707" alt="image" src="https://github.com/user-attachments/assets/80dbd253-0115-4cb1-a602-538f687fd219" />


- This command compiled the RTL hierarchy together with the VexRiscv CPU and standard‑cell models and then ran `vvp`, saving the output in `rtl_hkspi.log`. 

**Result**

```text
Monitor: Test HK SPI (RTL) Passed
```

- The RTL test completed successfully and printed that the hkspi test passed. 

***

## Final GLS run (mixed flow) and log capture

After fixing `caravel_netlists.v`, cloning `mgmt_core_wrapper`, avoiding duplicate housekeeping modules, and including VexRiscv, I obtained a GLS log `gls_hkspi.log` where the register prints matched the RTL run. The final GLS command reused a similar iverilog invocation but with `-DGL` and appropriate GL/RTL mixtures where needed, and the runtime log included the register reads used for comparison. 

***

## RTL vs GLS comparison

### Extracting register reads

```bash
grep "Read register" rtl_hkspi.log > rtl_reads.txt
grep "Read register" gls_hkspi.log > gls_reads.txt
head rtl_reads.txt gls_reads.txt
```

<img width="1096" height="599" alt="image" src="https://github.com/user-attachments/assets/4a8e97fb-b487-424f-87b6-0e7c3342e679" />


- `grep "Read register"`: I filtered the logs to only keep lines that print register reads, which are the functional checkpoints of the hkspi test. 
- I saved the RTL and GLS subsets into `rtl_reads.txt` and `gls_reads.txt` respectively for a clean diff. 
- `head rtl_reads.txt gls_reads.txt`: I quickly inspected the first lines to confirm that the GLS log contained the expected register read sequence. 

### Running diff

```bash
diff -s rtl_reads.txt gls_reads.txt
```

**Intermediate diff**

```text
0a1,19
> Read register 0 = 0x00 (should be 0x00)
...
> Read register 18 = 0x04 (should be 0x04)
```

<img width="1095" height="90" alt="image" src="https://github.com/user-attachments/assets/bbbcf960-4623-49b4-b976-8c203448a5c9" />


- At first, the GLS file contained the register reads while `rtl_reads.txt` was empty or incomplete, indicating that I needed a clean RTL run with the same logging format. 

After rerunning RTL with the correct configuration and comparison:

```bash
... && vvp hkspi.vvp | tee rtl_hkspi.log && \
grep "Read register" rtl_hkspi.log > rtl_reads.txt && \
grep "Read register" gls_hkspi.log > gls_reads.txt && \
echo "----------------------------------------" && \
echo "COMPARING RESULTS..." && \
diff -s rtl_reads.txt gls_reads.txt
```

**Final result**

```text
Files rtl_reads.txt and gls_reads.txt are identical
```

- This shows that, for every register index printed in the hkspi testbench, the GLS run produced the same value as the RTL run. 

***

## Summary of major errors and fixes

| Error / Symptom                                   | Cause                                                        | Fix                                                                                 |
|---------------------------------------------------|--------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `mgmt_core_wrapper.v not found`                   | Missing `verilog/rtl/mgmt_core_wrapper` directory            | Clone `caravel_mgmt_soc_litex` into `verilog/rtl/mgmt_core_wrapper`.         |
| `gl/mgmt_defines.v not found`                     | Mismatched include path in `caravel_netlists.v`              | Replace `gl/mgmt_defines.v` with `defines.v` via `sed`.                      |
| `gl/mgmt_core_wrapper.v not found`                | Netlist expecting `gl/` prefix not matching actual path      | Replace `gl/mgmt_core_wrapper.v` with `mgmt_core_wrapper.v` in `caravel_netlists.v`.  |
| Duplicate `housekeeping` module declaration       | Both RTL and GL versions compiled together                  | Comment out RTL `housekeeping.v` include and compile GL `housekeeping.v` only.  |
| `Unknown module type: VexRiscv`                   | VexRiscv module file not compiled                           | Find `VexRiscv*.v` and add `$VEX_FILE` to iverilog command line.             |
| `$readmemh: Unable to open hkspi.hex for reading` | Running from wrong directory or path mismatch for hex file   | Run simulations from hkspi DV directory where `hkspi.hex` is located.        |

***
***

# Caravel hkspi RTL Simulation Guide

Complete step-by-step guide for performing RTL simulation of the housekeeping SPI (hkspi) module in Caravel on Ubuntu/Linux systems.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [RTL Simulation Steps](#rtl-simulation-steps)
- [Common Errors and Solutions](#common-errors-and-solutions)
- [Verification](#verification)
- [Understanding the Commands](#understanding-the-commands)

***

## Prerequisites

Before starting, ensure you have the following tools installed:

```bash
iverilog --version   # Icarus Verilog (any recent version)
vvp --version        # Verilog runtime
git --version        # Git version control
python3 --version    # Python 3.x
pip3 --version       # pip package manager
```

If any tool is missing, install using:
```bash
sudo apt-get update
sudo apt-get install iverilog git python3 python3-pip
```

***

## Environment Setup

### Step 1: Navigate to hkspi Directory

```bash
cd ~/Desktop/VLSI/caravel_vsd/caravel/verilog/dv/caravel/mgmt_soc/hkspi
```

**Explanation:** This changes your working directory to the hkspi test location where the testbench (`hkspi_tb.v`) and test files (`hkspi.hex`) are located.

***

### Step 2: Set CARAVEL_ROOT Environment Variable

```bash
export CARAVEL_ROOT=$(pwd | sed 's|/verilog/dv/caravel/mgmt_soc/hkspi||')
```

**Explanation:** 
- `pwd` prints the current working directory
- `sed` removes the hkspi subdirectory path to get the Caravel repository root
- This variable is used by compilation commands to locate source files

***

### Step 3: Set PDK_ROOT and PDK Variables

```bash
export PDK_ROOT=$CARAVEL_ROOT/pdk
```

```bash
export PDK=sky130A
```

**Explanation:**
- `PDK_ROOT` points to the Process Design Kit directory containing Sky130A libraries
- `PDK` specifies which PDK variant to use (sky130A for this project)

***

### Step 4: Verify Management Core Wrapper Exists

```bash
ls $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper
```

**Expected output:** Directory listing showing folders like `def`, `gds`, `verilog`, etc.

**If error "No such file or directory" appears:**

```bash
cd $CARAVEL_ROOT && git clone https://github.com/efabless/caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper && cd -
```

**Explanation:** The management core wrapper contains the VexRiscv CPU and management SoC RTL required for simulation. This step clones it if missing.

***

### Step 5: Fix caravel_netlists.v Include Paths

Navigate to RTL directory:
```bash
cd $CARAVEL_ROOT/verilog/rtl
```

Fix digital_pll include:
```bash
sed -i 's|"gl/digital_pll.v"|"digital_pll.v"|g' caravel_netlists.v
```

Fix GPIO control block include:
```bash
sed -i 's|"gl/gpio_control_block.v"|"gpio_control_block.v"|g' caravel_netlists.v
```

Fix GPIO signal buffering include:
```bash
sed -i 's|"gl/gpio_signal_buffering.v"|"gpio_signal_buffering.v"|g' caravel_netlists.v
```

Fix management defines include:
```bash
sed -i 's|"gl/mgmt_defines.v"|"defines.v"|g' caravel_netlists.v
```

**Explanation:** 
- `sed -i` performs in-place file editing
- These commands remove incorrect `gl/` prefixes from include paths
- The `caravel_netlists.v` file references various modules that need correct paths for RTL simulation

***

### Step 6: Create Dummy Fill Cell Module

```bash
echo "module sky130_ef_sc_hd__fill_4(inout VPWR, inout VGND, inout VPB, inout VNB); endmodule" > $CARAVEL_ROOT/verilog/dv/dummy_fill.v
```

**Explanation:** Creates a dummy (empty) module for the Sky130 fill cell that appears in netlists but isn't functionally required for RTL simulation.

***

### Step 7: Return to hkspi Directory

```bash
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
```

***

### Step 8: Locate VexRiscv CPU File

```bash
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)
```

**Explanation:** 
- `find` searches for the VexRiscv CPU Verilog file
- `head -n 1` takes the first match
- The VexRiscv is the RISC-V CPU used in Caravel's management SoC

Verify it was found:
```bash
echo $VEX_FILE
```

**Expected output:** Path ending with something like `/VexRiscv.v` or `/VexRiscv_MinDebugCache.v`

***

## RTL Simulation Steps

### Step 9: Compile RTL with Icarus Verilog

```bash
iverilog -Ttyp -DFUNCTIONAL -DSIM -D USE_POWER_PINS -D UNIT_DELAY=#1 -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc -I $CARAVEL_ROOT/verilog/dv/caravel -I $CARAVEL_ROOT/verilog/rtl -I $CARAVEL_ROOT/verilog -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl -I $PDK_ROOT/sky130A -y $CARAVEL_ROOT/verilog/rtl -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v $CARAVEL_ROOT/verilog/dv/dummy_fill.v $VEX_FILE hkspi_tb.v -o hkspi.vvp
```

**Explanation of flags:**
- `-Ttyp`: Select typical timing model
- `-DFUNCTIONAL`: Define FUNCTIONAL macro for functional (non-timing) simulation
- `-DSIM`: Define SIM macro to enable simulation-specific code
- `-D USE_POWER_PINS`: Enable power pin connections required by Sky130 cells
- `-D UNIT_DELAY=#1`: Set unit delay to 1 time unit
- `-I <path>`: Add include search paths for `include` directives
- `-y <path>`: Add library search paths for module auto-discovery
- `primitives.v`: Sky130 primitive cells (basic gates)
- `sky130_fd_sc_hd.v`: Sky130 standard cell library (high-density)
- `sky130_fd_io.v`: Sky130 I/O pad library
- `-o hkspi.vvp`: Output compiled simulation file

**Expected output:** Minimal or no output means successful compilation.

***

### Step 10: Run the Simulation

```bash
vvp hkspi.vvp | tee rtl_hkspi.log
```

**Explanation:**
- `vvp` executes the compiled Verilog simulation
- `| tee rtl_hkspi.log` displays output to terminal AND saves it to `rtl_hkspi.log`
- The simulation reads `hkspi.hex` to initialize memory for the SPI flash model

**Expected output:** Should show simulation progress and register read operations.

***

### Step 11: Verify Success

```bash
grep "Monitor: Test HK SPI (RTL) Passed" rtl_hkspi.log
```

**Expected output:**
```
Monitor: Test HK SPI (RTL) Passed
```

***

## Common Errors and Solutions

### Error 1: Empty Simulation Output

**Symptom:**
```bash
vvp hkspi.vvp | tee rtl_hkspi.log
# Returns immediately with no output
```

**Possible Causes:**

1. **hkspi.hex file missing or in wrong location**

**Check:**
```bash
ls -lh hkspi.hex
```

**Solution:** Ensure you're running from the hkspi directory:
```bash
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
```

2. **Testbench missing $dumpfile or $finish statements**

**Check:**
```bash
grep -n "\$finish" hkspi_tb.v
grep -n "initial" hkspi_tb.v
```

**Solution:** Verify testbench has proper simulation control statements.

3. **Waveform dumping commented out (reduces output)**

**Check:**
```bash
grep "dumpfile" hkspi_tb.v
```

If lines start with `//`, waveform dumping is disabled (this is OK for lightweight simulation).

***

### Error 2: mgmt_core_wrapper.v Not Found

**Error message:**
```
Include file mgmt_core_wrapper.v not found
```

**Cause:** Management SoC wrapper repository not cloned.

**Solution:**
```bash
cd $CARAVEL_ROOT
git clone https://github.com/efabless/caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
```

***

### Error 3: VexRiscv Module Not Found

**Error message:**
```
error: Unknown module type: VexRiscv
```

**Cause:** VexRiscv CPU file not included in compilation.

**Solution:** Ensure Step 8 was completed and `$VEX_FILE` is set:
```bash
echo $VEX_FILE
# Should show path to VexRiscv*.v file
```

If empty, re-run:
```bash
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)
```

Then recompile (Step 9).

***

### Error 4: Include File gl/xxx.v Not Found

**Error message:**
```
Include file gl/digital_pll.v not found
```

**Cause:** `caravel_netlists.v` has incorrect GL (gate-level) path prefixes for RTL simulation.

**Solution:** Run Step 5 completely to fix all include paths using `sed` commands.

***

### Error 5: $readmemh Cannot Open hkspi.hex

**Error message:**
```
ERROR: $readmemh: Unable to open hkspi.hex for reading
```

**Cause:** Simulation running from wrong directory or hex file missing.

**Solution 1 - Verify location:**
```bash
pwd
# Should end with .../mgmt_soc/hkspi
```

**Solution 2 - Check hex file exists:**
```bash
ls -lh hkspi.hex
```

**Solution 3 - Generate hex file if missing:**
Check if `hkspi.c` exists, then compile it (requires RISC-V toolchain):
```bash
ls -lh hkspi.c
# If you see hkspi.c, you may need to compile it first
```

***

### Error 6: Simulation Hangs (Never Finishes)

**Symptom:** `vvp` command runs but never terminates.

**Cause:** Testbench may be waiting for signals or lacks $finish statement.

**Solution 1 - Add timeout:**
Kill the process (Ctrl+C) and check testbench has timeout:
```bash
grep -A5 "initial" hkspi_tb.v | grep "finish"
```

**Solution 2 - Run with timeout:**
```bash
timeout 60 vvp hkspi.vvp | tee rtl_hkspi.log
```
This kills simulation after 60 seconds.

***

### Error 7: Compilation Warnings About Undefined Modules

**Warning message:**
```
warning: macro USE_POWER_PINS is undefined
```

**Cause:** Normal for some library cells that conditionally use power pins.

**Solution:** These warnings are typically safe to ignore if compilation completes.

***

## Verification

### Check Log File Content

```bash
cat rtl_hkspi.log
```

Should show:
- Memory loading messages
- Register read/write operations
- Test progress indicators
- Final "Passed" message[1]

***

### Extract Register Reads

```bash
grep "Read register" rtl_hkspi.log > rtl_reads.txt
cat rtl_reads.txt
```

**Expected output:** Multiple lines showing register reads with expected vs actual values:
```
Read register 0 = 0x00 (should be 0x00)
Read register 1 = 0x04 (should be 0x04)
...
```

***

### Verify All Tests Passed

```bash
grep -i "passed\|failed" rtl_hkspi.log
```

Should show "Passed" and no "Failed" messages.

***

## Understanding the Commands

### What is iverilog?

Icarus Verilog (`iverilog`) is an open-source Verilog compiler that converts Verilog source code into an executable simulation format.

### What is vvp?

`vvp` (Verilog Simulation Runtime) executes the compiled `.vvp` file produced by iverilog.

### What is hkspi?

The housekeeping SPI (hkspi) is a 4-pin SPI interface in Caravel that allows external access to configuration registers, CPU control, and system monitoring.

### What does the hkspi test verify?

The test verifies:
- SPI communication protocol (Mode 0)
- Register read/write operations through housekeeping SPI
- Proper data transfer between host and management SoC
- Register values match expected configuration

***

## Quick Reference - Complete Workflow

Copy and paste these commands for a complete RTL simulation (assuming setup is done):

```bash
# Set environment
export CARAVEL_ROOT=$(pwd | sed 's|/verilog/dv/caravel/mgmt_soc/hkspi||')
export PDK_ROOT=$CARAVEL_ROOT/pdk
export PDK=sky130A
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi

# Set VexRiscv path
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)

# Compile
iverilog -Ttyp -DFUNCTIONAL -DSIM -D USE_POWER_PINS -D UNIT_DELAY=#1 \
  -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
  -I $CARAVEL_ROOT/verilog/dv/caravel \
  -I $CARAVEL_ROOT/verilog/rtl \
  -I $CARAVEL_ROOT/verilog \
  -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  -I $PDK_ROOT/sky130A \
  -y $CARAVEL_ROOT/verilog/rtl \
  -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
  $CARAVEL_ROOT/verilog/dv/dummy_fill.v \
  $VEX_FILE \
  hkspi_tb.v -o hkspi.vvp

# Run simulation
vvp hkspi.vvp | tee rtl_hkspi.log

# Verify
grep "Monitor: Test HK SPI (RTL) Passed" rtl_hkspi.log
```

***

## Troubleshooting Checklist

If simulation fails, verify:

- [ ] You're in the correct directory (`pwd` shows `.../hkspi`)
- [ ] `hkspi.hex` exists in current directory (`ls hkspi.hex`)
- [ ] `$CARAVEL_ROOT` is set correctly (`echo $CARAVEL_ROOT`)
- [ ] `$VEX_FILE` points to VexRiscv file (`echo $VEX_FILE`)
- [ ] PDK files exist (`ls $PDK_ROOT/sky130A/libs.ref/`)
- [ ] mgmt_core_wrapper was cloned (`ls $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper`)
- [ ] `caravel_netlists.v` was fixed (check with `grep "gl/" $CARAVEL_ROOT/verilog/rtl/caravel_netlists.v`)
- [ ] Compilation completed without errors

***

## Next Steps

After successful RTL simulation:

1. **Compare with GLS (Gate-Level Simulation)** to verify functional equivalence
2. **Run other Caravel DV tests** using similar methodology
3. **Modify test parameters** in `hkspi_tb.v` for custom verification
4. **Generate waveforms** by uncommenting `$dumpfile`/`$dumpvars` in testbench

***
