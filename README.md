# How to Install NVIDIA 50X on Ubuntu

📺 Video: [YouTube Guide](https://www.youtube.com/watch?v=o5deOXLDpZw&t=182s)

Basically, you need to install `build-essential` for GCC and then manually download the `.run` file of version **570 or upwards**.

⚠️ If a different version that you manually installed gets replaced later, you need to do a **full cleanup** and then repeat the installation steps above.  

To fully remove NVIDIA, follow the steps from this ChatGPT conversation:  
[Full Cleanup Instructions](https://chatgpt.com/share/68ab3a18-90c8-800d-9c45-0f464c7c2b4a)

---

## Complete NVIDIA Cleanup Steps (Ubuntu)

### 1️⃣ Switch to a text console
```bash
Ctrl+Alt+F3
````

### 2️⃣ Stop the display manager (GUI)

For GNOME (GDM3):

```bash
sudo systemctl stop gdm3
```

### 3️⃣ Unload NVIDIA kernel modules

Order matters:

```bash
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia 2>/dev/null
```

💡 Note: `nvidia` usually freezes my monitor even after deactivating `gdm3`.
Ideally, start Ubuntu in **text-only mode** from GRUB. If that fails, run them one by one; when the screen freezes after unloading `nvidia`, wait a bit, then type blindly:

```bash
sudo reboot
```

Check with:

```bash
lsmod | grep nvidia
```

After rebooting, you should see nothing.

---

### 4️⃣ Remove leftover NVIDIA kernel modules

```bash
sudo find /lib/modules/$(uname -r) -type f -name "*nvidia*.ko" -exec rm -f {} \;
sudo depmod -a
```

*(I have not seen anything here, probably not needed.)*

### 5️⃣ Remove NVIDIA configuration files

```bash
sudo rm -f /etc/modprobe.d/nvidia-*.conf
sudo rm -f /etc/X11/xorg.conf
sudo rm -f /etc/X11/xorg.conf.d/10-nvidia.conf
```

*(Same here, nothing found in my case.)*

---

### 6️⃣ Purge all NVIDIA packages

```bash
sudo apt-get purge '^nvidia-.*'
sudo apt-get autoremove
sudo apt-get autoclean
```

⚠️ This is important, especially if the drivers (that don’t work) were installed via `apt-get`.

Sometimes extra packages remain, check with:

```bash
dpkg -l | grep nvidia
```

Then remove manually:

```bash
sudo apt-get purge <package-name>
```

---

### 7️⃣ (Optional) Disable Nouveau (to avoid conflicts)

```bash
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
```

*(Not sure if needed.)*

---

### 8️⃣ Reboot

```bash
sudo reboot
```
## CUDA Version Mismatch Fix (When nvcc Fails)

To find the appropriate nvcc version you go to:

https://developer.nvidia.com/cuda-12-8-1-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=runfile_local

### Problem: nvcc version > nvidia-smi CUDA version

If `nvcc --version` shows a higher CUDA version than `nvidia-smi` (e.g., nvcc shows 13.0 but nvidia-smi shows 12.8), this creates compatibility issues with PyTorch and other CUDA applications.

**Common symptoms:**
- PyTorch shows `CUDA available: False` despite working GPU
- CUDA initialization errors in applications
- Version mismatch between toolkit and driver

### Quick Fix (No Driver Reinstall Needed):
```bash
# 1. Remove mismatched CUDA toolkit packages
sudo apt purge cuda-*-13-0 cuda-toolkit-13-*
sudo apt autoremove
sudo rm -rf /usr/local/cuda-13.0 /usr/local/cuda-13

# 2. Download correct CUDA toolkit version (match nvidia-smi version)

# For CUDA 12.8:
wget https://developer.download.nvidia.com/compute/cuda/12.8.1/local_installers/cuda_12.8.1_570.124.06_linux.run

# 3. Install ONLY the toolkit (preserve your working driver!)
sudo sh cuda_12.8.1_570.124.06_linux.run --toolkit --silent --override

# 4. Verify versions now match
nvcc --version  # Should show 12.8
nvidia-smi      # Should show CUDA 12.8

# 5. Reboot to initialize CUDA context properly
sudo reboot
```

**⚠️ Critical:** Always use `--toolkit` flag to avoid touching your working RTX 5090 driver!

**✅ Success indicators:**
- `nvcc --version` matches `nvidia-smi` CUDA version
- PyTorch shows `CUDA available: True`
- GPU is detected: `torch.cuda.get_device_name(0)`

---

## Update Driver + Toolkit with a New `.run` File

If you already installed NVIDIA/CUDA using a `.run` file and now want to move to a newer version, you do **not** need to remove the old toolkit first.

These steps are written for **Ubuntu / Ubuntu-based distributions** using `systemd`, `apt`, and a desktop display manager such as `gdm3`.

What happens during the upgrade:

- The **driver** gets replaced by the newer one.
- The **new toolkit** gets installed in a new folder such as `/usr/local/cuda-13.3`.
- The **old toolkit** usually remains on disk, for example `/usr/local/cuda-12.8`.

This is usually the safest way to upgrade because you can confirm the new version works before removing the older toolkit.

### 1️⃣ Download the new CUDA `.run` installer

Example:

```bash
~/Downloads/cuda_13.3.0_610.43.02_linux.run
```

### 2️⃣ Boot Ubuntu without the GUI

The most reliable method is to boot directly into **text mode from GRUB**, not just switch to `Ctrl+Alt+F3` after the desktop already started.

1. Reboot
2. In GRUB, highlight your Ubuntu entry and press `e`
3. Find the line that starts with `linux`
4. Add this at the end of that line:

```text
systemd.unit=multi-user.target
```

5. Boot with `Ctrl+X` or `F10`

This starts Ubuntu without the graphical interface, which makes it much easier to replace the NVIDIA driver.

### 3️⃣ Check that the GUI stack is really stopped

After logging in to the text console, run:

```bash
systemctl is-active gdm3
systemctl is-active nvidia-persistenced
ps -eo pid,comm,args | grep -E 'Xorg|gdm|gnome-shell|wayland|nvidia-persistenced'
lsmod | grep nvidia
```

What you want:

- `gdm3` should be `inactive`
- `nvidia-persistenced` should be `inactive`
- no `Xorg` / `gnome-shell` processes should remain
- ideally `lsmod | grep nvidia` shows nothing, or at least the modules can be unloaded

> A very common failure mode is: `gdm3` is inactive, but NVIDIA modules such as `nvidia_uvm` are still loaded. In that case the installer aborts and tells you to look in `/var/log/nvidia-installer.log`.

### 4️⃣ Unload NVIDIA modules if they are still present

```bash
sudo systemctl stop nvidia-persistenced
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia
```

Then confirm again:

```bash
lsmod | grep nvidia
```

If this still shows loaded modules, do **not** continue with the installer yet.

### 5️⃣ Run the installer

```bash
cd ~/Downloads
sudo sh ./cuda_13.3.0_610.43.02_linux.run
```

In the installer:

- Select `Driver`
- Select `Toolkit`
- Keep the default install path unless you have a reason to change it

The `MIT/GPL` kernel module choice is **not** the problem when running in text mode. The installer can answer that automatically. The usual blocker is that an old NVIDIA kernel module is still loaded.

### 6️⃣ Reboot

```bash
sudo reboot
```

### 7️⃣ Verify the new install

```bash
nvidia-smi
nvcc --version
ls -l /usr/local | grep cuda
```

Expected result:

- `nvidia-smi` shows the new driver version
- `nvcc --version` shows the new toolkit version
- both the old and new toolkit folders may exist side by side

### 8️⃣ If needed, point `/usr/local/cuda` to the new toolkit

Sometimes the installer updates the symlink automatically, but it is worth checking:

```bash
ls -l /usr/local/cuda
```

If needed:

```bash
sudo ln -sfn /usr/local/cuda-13.3 /usr/local/cuda
```

## Uninstall the Old Toolkit After the Upgrade

Only do this **after** confirming that:

- the new driver works
- `nvcc --version` points to the new version
- PyTorch and/or vLLM work correctly

### 1️⃣ Check which CUDA folders are installed

```bash
ls -l /usr/local | grep cuda
```

Example:

- `/usr/local/cuda-12.8`
- `/usr/local/cuda-13.3`

### 2️⃣ Look for the old toolkit uninstaller

Run:

```bash
ls /usr/local/cuda-12.8/bin | grep uninstall
```

If present, use it:

```bash
sudo /usr/local/cuda-12.8/bin/uninstall_cuda_12.8.pl
```

The exact filename may vary slightly by version.

### 3️⃣ If there is no uninstaller, remove the old toolkit folder manually

Only remove the **old toolkit directory**, not the currently active one:

```bash
sudo rm -rf /usr/local/cuda-12.8
```

### 4️⃣ Recheck the active symlink

```bash
ls -l /usr/local/cuda
```

If it still points to the correct new version, you are done. Otherwise:

```bash
sudo ln -sfn /usr/local/cuda-13.3 /usr/local/cuda
```

### 5️⃣ Final verification

```bash
nvcc --version
nvidia-smi
```

The driver should stay on the new version, and `nvcc` should point to the new toolkit.
