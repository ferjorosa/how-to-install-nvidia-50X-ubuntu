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
