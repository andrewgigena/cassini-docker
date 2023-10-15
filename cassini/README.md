# Cassini - Raspberry Pi 2

## Installing Arch Linux

1. Start fdisk to partition the SD card:

    ```bash
    fdisk /dev/mmcblk1
    ```

2. At the fdisk prompt, delete old partitions and create a new one:
    1. Type o. This will clear out any partitions on the drive.
    2. Type p to list partitions. There should be no partitions left.
    3. Type n, then p for primary, 1 for the first partition on the drive, press ENTER to accept the default first sector, then type +200M for the last sector.
    4. Type t, then c to set the first partition to type W95 FAT32 (LBA).
    5. Type n, then p for primary, 2 for the second partition on the drive, and then press ENTER twice to accept the default first and last sector.
    6. Write the partition table and exit by typing w.

3. Create and mount the FAT filesystem:

    ```bash
    mkfs.vfat /dev/mmcblk1p1
    mkdir boot
    mount /dev/mmcblk1p1 boot
    ```

4. Create and mount the ext4 filesystem:

    ```bash
    mkfs.ext4 /dev/mmcblk1p2
    mkdir root
    mount /dev/mmcblk1p2 root
    ```

5. Download and extract the root filesystem (as root, not via sudo):

    ```bash
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-armv7-latest.tar.gz
    bsdtar -xpf ArchLinuxARM-rpi-armv7-latest.tar.gz -C root
    sync
    ```

6. Move boot files to the first partition:

    ```bash
    mv root/boot/* boot
    ```

7. Unmount the two partitions:

    ```bash
    umount boot root
    ```

8. Insert the SD card into the Raspberry Pi, connect ethernet, and apply 5V power.

## Post installation

### Doing base configuration

1. Use the serial console or SSH to the IP address given to the board by your router.
    - Login as the default user alarm with the password alarm.
    - The default root password is root.

2. Log in as _alarm_ on ssh:

    ```bash
    ssh alarm@cassini_ip
    ```

3. Log in as _root_:

    ```bash
    su
    ```

4. Change the root password:

    ```bash
    passwd
    ```

5. Initialize the pacman keyring and populate the Arch Linux ARM package signing keys:

    ```bash
    pacman-key --init
    pacman-key --populate archlinuxarm
    ```

6. Update the keyring

    ```bash
    pacman -Sy --noconfirm archlinux-keyring
    ```

7. Update the system:

    ```bash
    pacman -Syu
    ```

8. Install sudo:

    ```bash
    pacman -S sudo
    ```

9. Edit the sudoers file:

    ```bash
    EDITOR=nano visudo
    ```

10. Uncomment the line and save:

    ```bash
    %wheel ALL=(ALL) ALL
    ```

11. Add the shadows user:

    ```bash
    useradd -m shadows
    passwd shadows
    usermod -aG wheel shadows
    ```

12. Log out from root and log in as shadows:

    ```bash
    exit
    exit
    ssh shadows@cassini_ip
    ```

13. If you can log in as shadows, delete the alarm user:

    ```bash
    sudo userdel -r alarm
    ```

### Installing basic tools and docker

1. Install basic tools:

    ```bash
    sudo pacman -Sy python rsync wget curl git neovim zsh
    sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
    ```

2. Install Avahi for mDNS:

    ```bash
    sudo pacman -Sy avahi nss-mdns
    ```

3. Configure Avahi and systemd-resolved:

    ```bash
    sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf
    sudo sed -r -i.orig 's/#?MulticastDNS=yes/MulticastDNS=no/g' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved systemd-networkd
    sudo systemctl enable --now avahi-daemon 
    ```

### Installing docker

1. Install docker:

    ```bash
    sudo pacman -Sy docker docker-compose docker-buildx docker-machine docker-scan
    ```

2. Enable docker:

    ```bash
    sudo systemctl enable --now docker.service containerd.service
    ```

3. Add shadows to the docker group:

    ```bash
    sudo usermod -aG docker shadows
    ```

4. Log out and log in again:

    ```bash
    exit
    ssh shadows@cassini_ip
    ```

5. Test docker:

    ```bash
    docker run hello-world
    ```

6. Install portainer:

    ```bash
    docker run -d \
        -p 8000:8000 \
        -p 9443:9443 \
        --name portainer \
        --restart=always \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /home/shadows/portainer_data:/data \
        portainer/portainer-ce:latest
    ```

7. Install watchtower:

    ```bash
    docker run -d \
        --name watchtower \
        --restart=always \
        -v /var/run/docker.sock:/var/run/docker.sock \
        containrrr/watchtower:latest
    ```

### Installing the services on portainer

1. Configure portainer:
    - Go to https://cassini_ip:9443
    - Create an admin user
    - Select local
    - Select the local docker environment
    - Go to Stacks

2. Create the stack cloudflare-ddns:
    - Name: cloudflare-ddns
    - Repository:
        - URL: <https://github.com/andrewgigena/homelab-dockers>
        - Reference: _default_
        - Path in the repository: cassini/cloudflare-ddns.yml
    - GitOps: True
    - Deploy the stack

3. Create the stack pihole:
    - Name: pi-hole
    - Repository:
        - URL: <https://github.com/andrewgigena/homelab-dockers>
        - Reference: _default_  
        - Path in the repository: cassini/pi-hole.yml
    - GitOps: True
    - Deploy the stack

4. Create the stack nginx-proxy-manager:
    - Name: nginx-proxy-manager
    - Repository:
        - URL: <https://github.com/andrewgigena/homelab-dockers>>
        - Reference: _default_
        - Path in the repository: cassini/nginx-proxy-manager.yml
    - GitOps: True
    - Deploy the stack

5. Setup everything and enjoy!
