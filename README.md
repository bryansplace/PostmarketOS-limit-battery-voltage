# Reducing Battery Charge Voltage on PostmarketOS

## Purpose
When using an old smartphone as a permanent server that's always plugged into power, reducing the maximum battery charge voltage significantly extends battery lifespan.

Lithium-ion batteries degrade faster when kept at high voltages. By limiting the maximum charge voltage to 3.8V instead of the standard 4.4V,  the battery is kept at approximately 40-50% charge, which dramatically reduces stress and extends its useful life - essential for a device that's permanently connected to power.

## Background
This guide shows how to modify the 'device tree' to lower the maximum charging voltage from 4.4V (100% charge) to 3.8V (~40-50% charge), which is much healthier for a battery that's permanently connected to power.

The device tree is a data structure that describes the hardware configuration to the Linux kernel. Modifying the battery's `voltage-max-design-microvolt` parameter in the device tree tells the charging system to stop charging at a lower, safer voltage level.

## Prerequisites
- PostmarketOS installed on your device
- SSH or terminal access to the device
- Root/sudo access
- Basic familiarity with command line operations

## Important: Battery Voltage Preparation

**Critical Step:** Before modifying the device tree, ensure the battery voltage is below your target maximum voltage. 

**Why this matters:** If the current voltage is higher than your new maximum (e.g., battery at 4.2V when setting max to 3.8V), the charging system will see the battery as "overcharged" and enter a confused state. This can prevent proper charging behavior even after the voltage naturally drops.

**What to do:** Check current voltage first:

```bash
cat /sys/class/power_supply/qcom-battery/voltage_now
```

The value is in microvolts (e.g., 3700000 = 3.7V). Only proceed when this value is below your target maximum voltage. If it's higher, either:
- Wait for the battery to discharge naturally below your target
- Set an intermediate voltage first (e.g., 4.0V), reboot, wait for discharge, then lower to final target (3.8V)

## Steps

### 1. Install Device Tree Compiler

The device tree compiler (`dtc`) is needed to convert the binary device tree blob (.dtb) into human-readable text (.dts) that we can edit, and then compile it back.

```bash
sudo apk add dtc
```

### 2. Backup Original Device Tree

Always create a backup before modifying system files. This allows you to easily restore the original configuration if needed.

```bash
# Replace 'tissot' with your device name from /boot/*.dtb
sudo cp /boot/msm8953-xiaomi-tissot.dtb /boot/msm8953-xiaomi-tissot.dtb.backup
```

### 3. Decompile Device Tree

Convert the binary device tree blob into a text file we can edit:

```bash
dtc -I dtb -O dts /boot/msm8953-xiaomi-tissot.dtb -o ~/tissot.dts
```

Note: Warnings during decompilation are normal and can be ignored. These are usually about minor formatting issues that don't affect functionality.

### 4. Find Current Battery Voltage Setting

Search for the line containing the voltage setting and note its line number:

```bash
grep -n "voltage-max-design-microvolt" ~/tissot.dts
```

This will output something like:
```
3959:           voltage-max-design-microvolt = <0x432380>;
```

The number before the colon (3959) is the line number you'll need in the next step.

### 5. Modify the Voltage

Use `sed` to automatically replace the voltage value on the line you found:

```bash
# Change from 4.4V (0x432380) to 3.8V (0x39f880)
# Adjust line number (3959) based on your grep output from step 4
sed -i '3959s/0x432380/0x39f880/' ~/tissot.dts
```

**Voltage recommendations:**
- **3.8V (0x39f880)** - ~40-50% charge, best for permanent server use (recommended)
- **4.0V (0x3d0900)** - ~60% charge, good balance
- **4.1V (0x3e9380)** - ~80% charge, still beneficial but less dramatic improvement

**Converting voltage to hex:**

The voltage is specified in microvolts (μV) and must be in hexadecimal format. To convert:
1. Multiply your target voltage by 1,000,000 (e.g., 3.8V × 1,000,000 = 3,800,000 μV)
2. Convert the decimal number to hexadecimal
3. Prefix with `0x` to indicate hexadecimal notation

Example conversion command:

```bash
printf "0x%x\n" 3800000
```

Output: `0x39f880`

### 6. Verify the Change

Double-check that the modification was applied correctly:

```bash
grep -n -A 2 -B 2 "voltage-max-design-microvolt" ~/tissot.dts
```

You should see your new hex value (e.g., `0x39f880`) instead of the original `0x432380`.

### 7. Recompile Device Tree

Convert the modified text file back into a binary device tree blob:

```bash
dtc -I dts -O dtb ~/tissot.dts -o ~/tissot-modified.dtb
```

Note: Warnings during compilation are normal and can be ignored, similar to the decompilation step.

### 8. Install Modified Device Tree

Replace the original device tree file with your modified version:

```bash
sudo cp ~/tissot-modified.dtb /boot/msm8953-xiaomi-tissot.dtb
```

### 9. Create Automatic Charging Reset Script

**Why this is needed:** After modifying the voltage settings and rebooting, the charger driver enters a state where it won't automatically start charging even though the battery is below the new maximum voltage. This appears to be a state machine issue where the charger needs to be explicitly told to reinitialize.

**The solution:** Create a startup script that automatically resets the charger by toggling the USB connection state after each boot.

Create the directory and script file:

```bash
sudo mkdir -p /etc/local.d
sudo nano /etc/local.d/reset-charging.start
```

Add this content:

```bash
#!/bin/sh
# Reset USB charging after boot
# This fixes the issue where the charger stays in "Discharging" state
# after a reboot with modified voltage settings

# Wait for system to fully initialize
sleep 10

# Toggle USB online to reset charger state machine
# This simulates unplugging and replugging the USB cable
echo 0 | tee /sys/class/power_supply/qcom-smbchg-usb/online > /dev/null
sleep 2
echo 1 | tee /sys/class/power_supply/qcom-smbchg-usb/online > /dev/null

logger "USB charging reset completed"
```

Make the script executable and enable it to run at boot:

```bash
sudo chmod +x /etc/local.d/reset-charging.start
sudo rc-update add local default
```

**What this does:** The `local` service runs all executable `.start` scripts in `/etc/local.d/` during system startup. This script waits 10 seconds for the system to stabilize, then toggles the charger's online state (0=off, 1=on), which forces the charging hardware to reinitialize and start charging properly.

### 10. Reboot

Apply all changes by rebooting the system:

```bash
sudo reboot
```

### 11. Verify the Change

After reboot, wait approximately 15 seconds for the startup script to complete, then verify everything is working:

```bash
# Check that the new voltage maximum is active
cat /sys/class/power_supply/qcom-battery/voltage_max_design

# Check current battery voltage
cat /sys/class/power_supply/qcom-battery/voltage_now

# Check charging status (should show "Charging" or "Not charging" if at voltage limit)
cat /sys/class/power_supply/qcom-smbchg-usb/status
cat /sys/class/power_supply/qcom-battery/status
```

**Expected results:**
- `voltage_max_design`: Should show 3800000 (3.8V) instead of 4400000 (4.4V)
- `voltage_now`: Current voltage in microvolts (will vary based on charge level)
- `status`: Should show "Charging" if below 3.8V, or "Not charging" if at the voltage limit (not "Discharging")

If the status shows "Discharging" even though USB is connected, the startup script may not have run yet. Wait a bit longer or manually run:

```bash
echo 0 | sudo tee /sys/class/power_supply/qcom-smbchg-usb/online
sleep 2
echo 1 | sudo tee /sys/class/power_supply/qcom-smbchg-usb/online
```

## Reverting Changes

If you need to restore the original settings:

```bash
# Restore the original device tree
sudo cp /boot/msm8953-xiaomi-tissot.dtb.backup /boot/msm8953-xiaomi-tissot.dtb

# Optionally remove the charging reset script
sudo rm /etc/local.d/reset-charging.start

# Reboot to apply
sudo reboot
```

## Notes
- **Minimum safe voltage is 3.4V** - Never set the voltage below this, as it's the battery's minimum design voltage
- **For permanent server use, 3.8V seems an excellent balance** between longevity and having enough capacity for brief power outages. Batteries are normally initially shipped with about this voltage.
- **Changes persist across reboots** and most system updates, but may be overwritten if the kernel package updates the device tree files
- **The automatic charging reset script is essential** - without it, you'll need to manually unplug and replug USB after each reboot
- **Battery voltage should be below target before applying changes** to avoid confusing the charging system
- **Monitor the first few charge/discharge cycles** to ensure the system is working as expected

