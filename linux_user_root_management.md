# Setting Up a User and Root Management on Linux

## 1. Creating a New User
Start by creating a non-root user. Replace `<username>` with your choice.
```bash
useradd -m -G wheel -s /bin/bash <username>
passwd <username>
```
- `-m` → creates a home directory.  
- `-G wheel` → adds the user to the `wheel` group (Arch uses `wheel` for sudo privileges).  
- `-s /bin/bash` → sets Bash as the default shell.  

## 2. Granting Sudo Rights
On Arch Linux (and many distros), sudo access is controlled by `/etc/sudoers`.  

Enable sudo for the `wheel` group:
```bash
EDITOR=nano visudo
```
Uncomment this line:
```
%wheel ALL=(ALL:ALL) ALL
```
Now your user can run commands with `sudo`.

**Test it:**
```bash
su - <username>
sudo whoami
```
Expected output: `root`.

## 3. Disabling Root Password Login
Lock the root account:
```bash
sudo passwd -l root
```
This prevents root from authenticating with a password.  

### Implications:
- `su -` will no longer work, because `su` needs the root password.  
- `sudo -i` *will* work (if your user has sudo rights). It gives you an interactive root shell using **your** password, not root’s.  

Check status:
```bash
sudo passwd -S root
```
Look for `L` (locked).  

## 4. Re-enabling Root Password Login
If you change your mind and want root login back:
```bash
sudo passwd root
```
This sets (or resets) the password and unlocks root.  
Or explicitly unlock:
```bash
sudo passwd -u root
```

## 5. Disaster Scenario: Losing Sudo Rights
If you lock root **and** remove your user’s sudo rights, you’ve cut the ladder from under yourself. The system is still running, but you can’t climb back into admin mode.  

### Recovery Steps:
1. Boot from an Arch ISO or any live Linux USB.  
2. Mount your root partition and chroot:
   ```bash
   mount /dev/nvme0n1pX /mnt   # replace with your root partition
   arch-chroot /mnt
   ```
3. Reset root password:
   ```bash
   passwd root
   ```
4. Or re-add your user to the `wheel` group:
   ```bash
   usermod -aG wheel <username>
   ```
5. Exit and reboot:
   ```bash
   exit
   reboot
   ```

Now you’re back in control.

---

## Summary of Behaviors
- `su -` → requires root’s password. Breaks if root login is disabled.  
- `sudo -i` → requires *your* password, works as long as you’re in `wheel` with sudo rights.  
- Disabling root login = more secure (attackers can’t brute-force root).  
- Losing sudo rights after disabling root login = self-lockout → requires live USB rescue.  
