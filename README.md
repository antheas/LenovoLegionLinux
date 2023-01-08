# LenovoLegionLinux
Tools for controlling Lenovo Legion laptop in Linux like fan control and power mode.


## Windows
This is a pure Linux tool. If you need tools for Windows, check out:
* LenovoLegionToolkit
* LegionFanControl

## Instructions
Please do the following: 
    - Step: **the installation instructions**
    - Step: **then make the test**
    - Step: **If tests are succesful, install permantely.**
    - Step: **create your fan curve**

## Installation Instruction
### Requirements
You will need to install the following to download and build it. If there is an error of any package find the alternative name in your distro install them.

**Ubuntu/Debian**
```bash
sudo apt-get update
sudo apt-get install make gcc linux-headers-$(uname -r) build-essential git lm-sensors
```

**RHEL/CentOS**
```bash
sudo yum update
sudo yum install kernel-headers kernel-devel lm-sensors
sudo yum groupinstall "Development Tools"
sudo yum group install "C Development Tools and Libraries"
```

**Fedora**
```bash
sudo dnf install kernel-headers kernel-devel lm-sensors
sudo dnf groupinstall "Development Tools"
sudo dnf group install "C Development Tools and Libraries"
```

**openSUSE**
```bash
sudo zypper install make gcc kernel-devel kernel-default-devel git libopenssl-devel lm-sensors
```

### Build and Test Instruction
```bash
git clone https://github.com/johnfanv2/LenovoLegionLinux.git
cd LenovoLegionLinux/kernel_module
make
sudo make reloadmodule
```
For tests see `Initial Usage Testing` below. Do them first.

### Permanent Install Instruction
After succesful bulding and testing (see below) run from the folder `LenovoLegionLinux/kernel_module`
```bash
make
sudo make install
```

### Uninstall Instruction
Go to the folder `LenovoLegionLinux/kernel_module`
```bash
make
sudo make uninstall
```

## Initial Usage Testing
Please note:
    - Please test in this order and try to fix a failed text before going to the next. 
    - These tests are manual and in the terminal because this is an early version of this tool
    - Using it if test are succesful is much easier
    - You can copy-and-paste the commands. Paste with `Ctrl+Shift+V` inside the terminal.

### Quick Test: Module is properly loaded
```bash
# After you have run from folder LenovoLegionLinux/kernel_module
sudo make reloadmodule

# Check the kernel log
sudo dmesg
```
Expected result : 
    - You should see a line like the following like `legion PNP0C09:00: legion_laptop loaded for this device`. "PNP0C09" might be replaced by other text.

Unexpected result:
    - `insmod: ERROR: could not insert module legion-laptop.ko: Invalid module format` after running `make reloadmodule`
    - `legion PNP0C09:00: legion_laptop not loaded for this device`
        - Kernel module was not loaded properly



### Quick Test: Reading Current Fancurve from Hardware
```bash
# Read the current fancurve and other debug output
sudo cat /sys/kernel/debug/legion/fancurve
```

Expected output:
    - EC Chip ID should be 8227
    - fan curve points size must NOT be 0
    - the table that shows the current fan curve must NOT be only zeros, the actual values might change
    - fan curve current point id and EC Chip Version might differ
    
Example:
```text
EC Chip ID: 8227 
EC Chip Version: 2a4 
fan curve current point id: 0 
fan curve points size: 8 
Current fan curve in UEFI
rpm1|rpm2|acceleration|deceleration|cpu_min_temp|cpu_max_temp|gpu_min_temp|gpu_max_temp|ic_min_temp|ic_max_temp
0 0 2 2 0 48 0 59 0 41
1700 1900 2 2 45 54 55 59 39 44
1900 2000 2 2 51 58 55 59 42 50
2200 2100 2 2 55 62 55 59 46 127
2300 2400 2 2 59 71 55 59 127 127
2600 2700 2 2 68 76 55 64 127 127
2900 3000 2 2 72 81 60 68 127 127
3500 3500 2 2 78 127 65 127 127 127
```

The fan curve is displayed as a table with the following columns:
```text
rpm1: speed in rpm for fan1 at this point
rpm2: speed in rpm for fan1 at this point
acceleration: accelleration time (higher = slower)
deceleration: deceleration time (higher = slower)
cpu_min_temp: CPU temperatue must go below this before leaving this point
cpu_max_temp: if CPU temperature is above this value, go to next point 
gpu_min_temp: GPU temp must go below this before leaving this level
gpu_max_temp: if GPU temperature is above this value, go to next point 
ic_min_temp: IC temp must go below this before leaving this level
ic_max_temp: if IC temperature above this value, go to next point 

All temperatures are in degree Celsius.
```
**Note**: This is just a debug output. The fan curve is configured as usual using the standard `hwmon` interface.


Unexpected:
    - ` /sys/kernel/debug/legion/fancurve: No such file or directory`: Kernel module was not loaded properly
    - `cat: /sys/kernel/debug/legion/fancurve: Permission denied` you have forgot sudo

### Quick Test: Read Sensor Values from Hardware
- display sensor values and check that it contains lines with "Fan 1", "Fan 2", "CPU Temperature", "GPU Temperature":
```bash
sensors
```
- display sensor values
- increase the CPU load and check if 
    - displayed CPU temperature increases
    - displayed fan speed increases 
- display sensor values
- increase the GPU load and check if GPU temperature changes
    - displayed CPU temperature increases
    - displayed fan speed increases 

Expected output:
    - Output of `sensors` contains something like
        ```text
        legion_hwmon-isa-0000
        Adapter: ISA adapter
        Fan 1:           1737 RPM
        Fan 2:           1921 RPM
        CPU Temperature:  +42.0°C  
        GPU Temperature:  +30.0°C  
        IC Temperature:   +41.0°C  
        ```
    - if GPU is in deep sleep, its reported temperature is 0; run something on the GPU to test it
    - temperatures are valid: in particular not 0 (except GPU see above)
    - fan speeds are valid: if fan is off it is 0, otherwise greater than 1000 rpm
    - temperatures and fan speeds increase as expected

Unexpected output:
    - `Command 'sensors' not found`: Install `sensors` from the package `lm-sensors`     
    - no entries for "Fan 1", "Fan 2" etc. are shown
    

### Quick Test: Change Current Fancurve from Hardware with hwmon
```bash
# Change the RPM of fans at the second and third point of the fan curve to 1500 rpm, 1600 rpm, 1700 rpm, 1800 rpm.
# Get root
sudo su
# As root enter
# 2. point, 1. fan
echo 1500 > /sys/module/legion_laptop/drivers/platform:legion/PNP0C09:00/hwmon/hwmon*/pwm1_auto_point2_pwm
# 2. point, 2.fan
echo 1600 > /sys/module/legion_laptop/drivers/platform:legion/PNP0C09:00/hwmon/hwmon*/pwm2_auto_point2_pwm
# 3. point, 1. fan
echo 1700 > /sys/module/legion_laptop/drivers/platform:legion/PNP0C09:00/hwmon/hwmon*/pwm1_auto_point3_pwm
# 3. point, 2.fan
echo 1800 > /sys/module/legion_laptop/drivers/platform:legion/PNP0C09:00/hwmon/hwmon*/pwm2_auto_point3_pwm


# Read the current fancurve and other debug output
cat /sys/kernel/debug/legion/fancurve
```
Expected: 
    - The entries in the fan curve are set to their values. The other values are not relevant (marked with XXXX)
    - the controller might have loaded default values if you pressed Ctr+Q to change the powermode or waited too long; try again
```
rpm1|rpm2|acceleration|deceleration|cpu_min_temp|cpu_max_temp|gpu_min_temp|gpu_max_temp|ic_min_temp|ic_max_temp
XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
1500 1600 XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
1700 1800 XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
```
Unexpected: 
    - `file not found`
    - the values have not changed
    - there are different values


### Quick Test: Set your custom fan curve
Set a custom fan curve with the provided script.


### Test: Finish
- If you are satisfied with the test results, then you can install the kernel module permanently (see above).
- Create a GitHub Issue and report if the test work or fail. 
    - Please include exact laptop model
    - If errors occured, include output of commands.

# Normal Usage
- If you are satisfied with the test results, then install the kernel module permanently (see above), otherwise you must reload it manually after each restart.

## Temperatuer and Fan Monitoring
The tempereratues and fan speeds should be displayed in any graphical tool that monitors them, e.g. psensor. You have to install it first before running:
```bash
psensor
```
    
## Creating and Setting your own Fan Curve
Just run the script to set the fan curve. It is in the folder `LenovoLegionLinux`.
```bash
# Go to folder LenovoLegionLinux and run it. It should output "fancurve set"
sudo ./setmyfancurve.sh
# And check new fan curve
sudo cat /sys/kernel/debug/legion/fancurve
```
Open the file `setmyfancurve.sh` and edit it to adapt the values in the script to create your own fan curve. Follow the description in the file.

Unexpected output:
    - `bash: ./setmyfancurve.sh: Permission denied`: You have to make the script executable with `chmod +x ./setmyfancurve` first
    - script does not end with "fancurve set": maybe path to hwmon changed; Please report this

Note: 
    - Currently, there is no GUI available. 
    - Currenlty, the hardware resets the fan curve randomly or if you change powermode, suspend, or restart. Just run the script again. 
    - You might want to create different scritps for different usages. Just copy it and adapt the values.

