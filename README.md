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

These steps are written for **Ubuntu / Ubuntu-based distributions**.

### 1️⃣ Download the new CUDA `.run` installer

Example:

```bash
~/Downloads/cuda_13.3.0_610.43.02_linux.run
```

### 2️⃣ Boot Ubuntu without the GUI

In my case, just switching to a text console was **not enough**. What worked was booting directly from **GRUB** without the GUI.

1. Reboot
2. In GRUB, highlight your Ubuntu entry and press `e`
3. Find the line that starts with `linux`
4. Add this at the end of that line:

```text
systemd.unit=multi-user.target
```

5. Boot with `Ctrl+X` or `F10`

This starts Ubuntu without the graphical interface. In my case, using `text` did **not** work, but `systemd.unit=multi-user.target` did.

### 3️⃣ Run the full installer from that non-GUI boot

```bash
cd ~/Downloads
sudo sh ./cuda_13.3.0_610.43.02_linux.run
```

### 4️⃣ Reboot and check whether the driver updated

```bash
sudo reboot
nvidia-smi
```

If `nvidia-smi` shows the new version, then the driver update worked.

### 5️⃣ If `nvcc` is still old, install only the toolkit

In my case, the full installation had an issue, but the driver ended up installed correctly. The remaining fix was to install only the toolkit afterward, and this did **not** require starting without the GUI again.

```bash
cd ~/Downloads
sudo sh ./cuda_13.3.0_610.43.02_linux.run --toolkit
```

### 6️⃣ Verify the toolkit and switch `nvcc` to it

```bash
nvcc --version
ls -l /usr/local | grep cuda
ls -l /usr/local/cuda
```

If the new toolkit exists but `/usr/local/cuda` still points to the old one, update the symlink:

```bash
sudo ln -sfn /usr/local/cuda-13.3 /usr/local/cuda
```

### 7️⃣ Final verification

```bash
nvidia-smi
nvcc --version
```

Expected result:

- `nvidia-smi` shows the new driver version
- `nvcc --version` shows the new toolkit version
- both toolkit folders may exist side by side until you remove the old one

## Optional: Remove the Old Toolkit

Only do this **after** confirming that:

- `nvidia-smi` works
- `nvcc --version` shows the new version
- `/usr/local/cuda` points to the new toolkit
- PyTorch and/or vLLM work correctly

Check the current CUDA folders:

```bash
ls -l /usr/local | grep cuda
ls -l /usr/local/cuda
```

If the old toolkit is still there and is no longer in use, you can remove it directly:

```bash
sudo rm -rf /usr/local/cuda-12.8
```

Only remove the old versioned folder, **not** `/usr/local/cuda`.
