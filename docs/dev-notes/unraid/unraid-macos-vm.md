# Unraid MacOS Virtual Machine

This will guide you through the setup of an OSX machine as an Unraid Virtual Machine using macinabox.
This guide will start you off with Monterey (since that is the last currently support OS from macinabox) and then give you the optional steps to upgrade to Ventura.
I'm working to see if the same steps work for upgrading to Sonoma. Currently I have GPU passthrough working with a GeForce GT-710 (Monterey), GeForce GT-730 (Monterey), and AMD Radeon 6600 XT (Monterey, Ventura, Sonoma).
I also have Intel Bluetooth and Intel Wireless passthrough working (Monterey and Ventura).

## Preliminary Steps

- Clean out all previous failed attempts
  - delete two helper scripts
  - remove old macinabox docker
  - erase appdata/macinabox folder
  - might be good to clear out old vm files too
- If you plan to passthru the 6600xt, you have to do all the install process below on VNC or a GT710/730 and then at the end switch it to the 6600xt. I'll try to point out differences below.
  - go to Tools → System Devices in unraid to bind passthru devices to VFIO at boot. 
  - I also had to tell my bios to NOT use the GPU slot i wanted to passthrough but the more and more i do this, i'm not sure how much of that mattered. Maybe a lot? Maybe not at all?
- All this worked for me in legacy mode as that was what my unraid install was already set too. Never tried it in UEFI mode.

## Create OSX Virtual Machine

### Use Macinabox to install Monterey

- Docker variables
  ```ini
  Operating System Version: Monterey
  Install Type: Auto Install
  Vdisk Size: 100G
  Vdisk Type: raw
  Opencore stock or custom: stock
  Delete and replace Opencore: no
  Override default NIC type: no
  VM Images Location: /mnt/user/domain/
  Isos Share Location: /mnt/user/isos/
  appdata: /mnt/user/appdata/
  custom ovmf location (under more settings): /mnt/user/custom_ovmf/
  inject helper scripts: Yes
  ```

- RUN!
- Note for Ventura: I don't know if this is required (or even worked correctly) but I manually downloaded the latest OpenCore and tried using the “custom” feature with Macinabox to go straight to the latest OpenCore
  but I don't think that even did anything/worked/made a difference since I upgraded OpenCore manually anyway before upgrading to Ventura… future installs might answer that.
  (update: i'm 99% sure that did nothing as future installs for both Ventura and Sonoma i did NOT do this and just used stock)

### Finish Creating VM

- Go to your user scripts under settings and change the vmready notify script `(while [ ! -d ___ ]` to match appdata location (if you changed it in step 1 above) then run in background to let you know when ready.
- Once the vmready script says you are done downloading… open the helper script and edit the NAME to Macinabox Monterey and the appdata and custom_ovmf locations to the locations that match where you actually have them.
- RUN the helper script and your VM should now appear in the VM section as “Macinabox Monterey”

### Edit VM

From here on out… only edit xml by hand\*\*\* NEVER update or save changes using the vm GUI… it will break.
It broke my gpu passthru editing the form and then running the helper script each time. I'm sure it was something i messed up… but custom xml editing worked like a charm for me and eliminated all my problems!
JUST DON'T MESS WITH IT IF YOU'VE BEEN HAVING PROBLEMS. ONLY ONLY ONLY save the vm settings using the XML editor mode!!!

- Click on the vm and select edit. Then change to XML view in the top right.

- Change memory to whatever you want in both places
  
  ```xml
  <memory unit='KiB'>8388608</memory>
  <currentMemory unit='KiB'>8388608</currentMemory>
  ```

- Change vcpu and cputune to number of cores you want to use
  
  ```xml
    <vcpu placement='static'>8</vcpu>
    <cputune>
      <vcpupin vcpu='0' cpuset='4'/>
      <vcpupin vcpu='1' cpuset='12'/>
      <vcpupin vcpu='2' cpuset='5'/>
      <vcpupin vcpu='3' cpuset='13'/>
      <vcpupin vcpu='4' cpuset='6'/>
      <vcpupin vcpu='5' cpuset='14'/>
      <vcpupin vcpu='6' cpuset='7'/>
      <vcpupin vcpu='7' cpuset='15'/>
    </cputune>
  ```
  
  - ProTip: if unsure of your setup. Create fake linux vm and edit using the gui and then see what XML looks like and copy that... i did this a few times for things like CPU and Mem size
- Change os paths to match custom ovmf location you set in your docker template
  
  ```xml
  <loader readonly='yes' type='pflash'>/mnt/disks/samsung_nvme/system/custom_ovmf/Macinabox_CODE-pure-efi.fd</loader>
  <nvram>/mnt/disks/samsung_nvme/system/custom_ovmf/Macinabox_VARS-pure-efi.fd</nvram>
  ```

- Change CPU mode line to match above layout… in my case, 8 threads = 4 cores with 2 threads each
  
  ```xml
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' cores='4' threads='2'/>
  </cpu>
  ```

- Under Devices: Three disks should look like the following making sure the paths match where those files are
  
  ```xml
  <disk type='file' device='disk'>
    <driver name='qemu' type='raw' cache='writeback'/>
    <source file='/mnt/disks/samsung_nvme/vms/Macinabox Monterey/Monterey-opencore.img'/>
    <target dev='hdc' bus='sata'/>
    <boot order='1'/>
    <address type='drive' controller='0' bus='0' target='0' unit='2'/>
  </disk>
  <disk type='file' device='disk'>
    <driver name='qemu' type='raw' cache='writeback'/>
    <source file='/mnt/user/isos/Monterey-install.img'/>
    <target dev='hdd' bus='sata'/>
    <address type='drive' controller='0' bus='0' target='0' unit='3'/>
  </disk>
  <disk type='file' device='disk'>
    <driver name='qemu' type='raw' cache='writeback'/>
    <source file='/mnt/disks/samsung_nvme/vms/Macinabox Monterey/macos_disk.img'/>
    <target dev='hde' bus='sata'/>
    <address type='drive' controller='0' bus='0' target='0' unit='4'/>
  </disk>
  ```

- Next step is to edit the graphics lines.
  
  - If using VNC or installing a 6600xt using VNC… skip this next step. Just continue and future steps will fix the 6600xt.
  - If using GT710 or GT730 (ie, a GPU supported without needing to change any boot-args):
    - Erase everything in the `<graphics>` and `<video>` tags and add the following block
      ```xml
      <hostdev mode='subsystem' type='pci' managed='yes'>
        <driver name='vfio'/>
        <source>
          <address domain='0x0000' bus='0x0b' slot='0x00' function='0x0'/>
        </source>
        <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0' multifunction='on'/>
      </hostdev>
      <hostdev mode='subsystem' type='pci' managed='yes'>
        <driver name='vfio'/>
        <source>
          <address domain='0x0000' bus='0x0b' slot='0x00' function='0x1'/>
        </source>
        <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x1'/>
      </hostdev>
      ```
    
    - change the 0x0b to match whatever unraid says your video card is on (can be found in tools → system Devices).
    - change the 0x04 to whatever bus is free… 4 was the first open one the XML hadn't used for something else and will likely by fine for you as well.
- I also added passthrough Bluetooth controller which unraid said was at 08:00.0, .1, and .3 and that I assigned to 0x05
  
  ```xml
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0' multifunction='on'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x08' slot='0x00' function='0x1'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x1'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x08' slot='0x00' function='0x3'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x3'/>
    </hostdev>
  ```

- I also added passthrough USB controller card that unraid said was at 07:00.0 and that I assigned to 0x06
  
  ```xml
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
      <address domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
    </source>
    <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
  </hostdev>
  ```

- I also added passthrough Wi-Fi that unraid said was at 05:00.0 and that I assigned to 0x07
  
  ```xml
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
      <address domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
    </source>
    <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
  </hostdev>
  ```

- If passing through an individual mouse and keyboard, use the dummy vm trick and check the boxes for the keyboard and mouse and then copy and paste the appropriate XML code making sure the two busses in the address type lines are not used elsewhere.

- SAVE XML and RUN VM!

## Install & Configure OSX

### SETUP Bootloader

- Monterey should now boot to the bootloader either on your VNC viewer or monitor if using the GT710/730
- Select macOS Base System. MacOS recovery will start.
- Select Disk Utility
- Select the correct drive to install on (should match the size of the drive you created)
- Select Erase
- Give it a Name
- Select Erase / Select Done
- Quit Disk Utility
- Select Reinstall macOS Monterey
- Select Continue / Continue / Agree / Agree
- Select the disk you named above
- Select Continue
- MacOS will now install and restart a number of times (4 for me). Select the macOS installer during the first two restarts, and then the name of the disk you created during the next two restarts.
  - Note: If using a GT710 or GT730, it will be sluggish. We will fix this at the end to either switch over to a 6600xt, or install the GT drivers if you are planning to keep using that card.

### Install OpenCore Configurator

- Follow Monterey setup steps but don't sign in with Apple ID. we will fix that later
- Install OpenCore Configurator
- Drag OpenCore Configurator to Applications folder.
- Open OpenCore Configurator
  - You will get a message saying it can't be opened… click OK
  - go to System Preferences → Security & Privacy → click Open Anyway and select Open
- Go to Tools → Mount EFI
- In the bottom half under EFI Partitions, select the drive (EFI o) and click Mount Partition, put in your password, click Open Partition
- A folder should open. Copy the contents to the desktop (Initial installs had two things: EFI and NvVARS… subsequent installs only had the EFI folder)
- Unmount that partition
- Mount your harddrive (should be the one that has the name you gave it). Open that partition
- Copy the file(s) from the desktop into the window that pops up
- Delete files on desktop.

### UPDATE Bootloader

- UPDATE bootloader: When you opened up OpenCore Configurator it may have told you you were using an older version of boot loader.
  
  - Download the new package (0.9.7 at time of this guide) from here (you may also be able to upgrade from the Configurator by going to Tools → OpenCore Downloader)
  - unzip and open new opencore files
  - copy individual files into the folder, don't replace whole folders
    ```bash
    copy X64/EFI/BOOT/BOOTx64.efi into EFI/BOOT folder on your mounted drive (replacing old one)
    copy X64/EFI/OC/Opencore.efi into EFI/OC folder on your mounted drive (replacing old one)
    copy X64/EFI/OC/Drivers/*.* into EFI/OC/Drivers folder on your mounted drive (replacing old ones)
    copy X64/EFI/OC/Tools/*.* into EFI/OC/Tools folder on your mounted drive (replacing old ones)
    ```

- While here, lets make smbios
  
  - Still in your mounted drive, open `EFI/OC/Config.plist`
  - Go to Platform info and next to the “check coverage” button, use the selection list to pick what you want your Mac to “be”
    I picked Mac Pro 7,1 (2019) for what it's worth… not sure how much that would affect the rest of this install but i believe i needed this to make my 6600xt work.
    When i was just using the GT710/730, I selected the Mac Pro Late 2013. I'll be honest, not sure about how much difference this makes but the forums that I used mentioned that the 7,1 worked best for the 6600xt.
    It also appears to be one of the supported ones for Ventura and maybe higher… but it will require a workaround for a Memory Reporting Issue (see below)
  - Click the check coverage button to make sure that the serial number it generated for you is not on file… that way Apple services will work since you didn't randomly get an existing Mac Owners number.
  - Save and close open core window

### SHUT DOWN THE VM

- SHUT DOWN THE VM

- Back on Unraid, edit the XML
  
  - change the source file= line on the first disk to be what your actual vm disk is and then erase the other two disks… leaving you with this
    ```xml
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' cache='writeback'/>
      <source file='/mnt/disks/samsung_nvme/vms/Macinabox Monterey/macos_disk.img'/>
      <target dev='hdc' bus='sata'/>
      <boot order='1'/>
      <address type='drive' controller='0' bus='0' target='0' unit='2'/>
    </disk>
    ```

- Save changes and start up the VM

- update kexts
  
  - Open OpenCore Configurator
  - mount EFI (Tools → Mount EFI → click mount partition next to your drive in bottom section)
  - open EFI/OC/config.plist
  - go to Kernel and click on Download/Update kexts
    NOTE: it seems with later versions of OpenCoreConfigurator you can use the little icon in the top right to just mount and go straight to your config.plist
- select your disk to start up (we will set up auto start to this disk next)
  
  - Open OpenCore Configurator
  - mount EFI (Tools → Mount EFI → click mount partition next to your drive in bottom section)
  - open EFI/OC/config.plist
  - go to misc and uncheck show picker
  - Save and close OpenCore Configurator

## Update for Ventura

- Once kexts are updated, you should be able to use the appstore Ventura install to update
- Open AppStore and search for Ventura and start install process.
- Once the computer resets, you want to boot from the new install media drive in the boot picker
- It will crash until we fix the xml file of the VM
- shut down the vm and edit the xml file in unraid
  - change
    ```xml
    <qemu:commandline>
      <qemu:arg value='-usb'/>
      <qemu:arg value='-device'/>
      <qemu:arg value='usb-kbd,bus=usb-bus.0'/>
      <qemu:arg value='-device'/>
      <qemu:arg value='isa-applesmc,osk=ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc'/>
      <qemu:arg value='-smbios'/>
      <qemu:arg value='type=2'/>
      <qemu:arg value='-cpu'/>
      <qemu:arg value='Penryn,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc,+pcid,+ssse3,+sse4.2,+popcnt,+avx,+avx2,+aes,+fma,+fma4,+bmi1,+bmi2,+xsave,+xsaveopt,+rdrand,check'/>
    </qemu:commandline>
    ```
    
    to
    ```xml
    <qemu:commandline>
      <qemu:arg value='-usb'/>
      <qemu:arg value='-device'/>
      <qemu:arg value='usb-kbd,bus=usb-bus.0'/>
      <qemu:arg value='-device'/>
      <qemu:arg value='isa-applesmc,osk=ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc'/>
      <qemu:arg value='-smbios'/>
      <qemu:arg value='type=2'/>
      <qemu:arg value='-cpu'/>
      <qemu:arg value='host,vendor=GenuineIntel,+invtsc,+hypervisor,kvm=on,vmware-cpuid-freq=on'/>
      <qemu:arg value='-global'/>
      <qemu:arg value='ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off'/>
      <qemu:arg value='-global'/>
      <qemu:arg value='nec-usb-xhci.msi=off'/>
    </qemu:commandline>
    ```
  
  - if you have an AMD chip like me… change
    ```xml
    <qemu:arg value='-cpu'/>
    <qemu:arg value='host,vendor=GenuineIntel,+invtsc,+hypervisor,kvm=on,vmware-cpuid-freq=on'/>
    ```
    
    to
    ```xml
    <qemu:arg value='-cpu'/>
    <qemu:arg value='Haswell-noTSX,vendor=GenuineIntel,+invtsc,+hypervisor,kvm=on,vmware-cpuid-freq=on'/>
    ```

- Now your install media in the boot loader should load. Select it, then your hard drive during each restart until it boots up!

Links: https://github.com/thenickdude/KVM-Opencore/issues/47, https://www.nicksherlock.com/2022/10/installing-macos-13-ventura-on-proxmox/

## Update for Sonoma

- After completing all the steps to get a working copy of Ventura… I updated to Sonoma using the Update feature in system preferences. (this ended up not working very well and for future attempts i did the above steps for a fresh Monterey install... and then went straight to Sonoma... worked much better!)
- You can probably follow the steps listed in the above section (Update to Ventura) and just select Sonoma and it will probably upgrade from Monterey to Sonoma… I might check that sometime in the future. (update: this is better!)
- I've been running Sonoma all of 10 minutes now and everything “SEEMS” to work. Will update more as we go!
  - Wifi seems to cause everything to crash when you select the network tab (may be the same issue that was present in Ventura with having to hard restart the server to get wifi to not radio kill or whatever.
  - SMB shares are painfully slow to connect / don't work most of the time.
  - bluetooth doesn't work.
- Installing a fresh copy and going straight from Monterey to Sonoma to see if that helps.
  - Fresh copy going straight from Monterey to Sonoma seems to have been MUCH better.
  - Wifi still has same issue as above (maybe need a newer kext when it comes out)
  - SMB shares work great now.
  - bluetooth still doesn't work but at least you can turn it on and off. Just won't find any devices. (update: i believe this is a USB port issue... working on it still)

## Fixes and Tidy Up

Some of these may not apply to you but here's what I had to do to fix certain things as well as some things I did to tidy up my install

### Connect iPhone/iPad/MacOS Guest

- [Connect iPhone/iPad to macOS guest](https://github.com/kholia/OSX-KVM/blob/master/notes.md#connect-iphone--ipad-to-macos-guest)
- [iPhone docker osx usb passthrough](https://github.com/Silfalion/Iphone_docker_osx_passthrough)

### GPU Passthrough

- [GPU Passthrough PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

- AMD Radeon RX 6600 XT (works for Monterey and Ventura and Sonoma)
  
  - use OpenCore Configurator to mount EFI as above
  - open EFI/OC/config.plist
  - go to nvram
  - go to your nvram (the bottom one in UUID usually)
  - go to boot-args and add the value agdpmod=pikera
    (It should now be agdpmod=pikera keepsyms=1)
  - Shut down VM
  - Edit your XML like above in the beginning for the GT710 but instead of using the GT710 slot, make it use the 6600xt.
  - Link: https://www.tonymacx86.com/threads/success-xfx-rx-6600-xt-graphics-card-in-monterey-12-2-1.319128/
- NVidia GeForce GT710/GT730 (only tested on Monterey and less)
  
  - Download OpenCore Legacy Patcher (https://github.com/dortania/OpenCore-Legacy-Patcher/releases)
  - Open it and click on Post install Root Patch
  - Start Root Patching
  - Yes
  - password for root
  - Restart
  - Profit!!!
  - video of this: https://www.youtube.com/watch?v=y2whNpzL42s

### Ultra Wide monitor? (works for Monterey and Ventura and Sonoma)

- download DisableMonitor from https://github.com/eun/disablemonitor based on this blog: https://medium.com/zurassic/how-to-enable-5120x1440-on-mac-cd780c41a108

### Apple Login (works for Monterey and Ventura and Sonoma)

- Download hackintool (optional but makes things easier). (https://github.com/headkaze/Hackintool)
- open it (using same steps as you first used to open OpenCore Configurator) and go to System → Peripherals
- Network interfaces should have a en0 but the built in is unchecked (mine was actually checked on future installs so i don't know if the following steps were still required but i did them anyway :D / turns out I had to do this to get Documents and Desktop sync to turn on… or just needed a restart… either way I did it and that fixed that part in Ventura)
- open opencore → mount EFI → open config.plist
- go to DeviceProperties and select ethernet controller from list of PCI Devices at bottom (if it's not listed already)
- click the + on the far bottom right of the right box to add a property to the ethernet controller.
  - For “key” use: built-in
  - For “type” use: Data
  - For “value” use: 01
- save and reboot
- log in to apple services!
- The hackintool technically isn't needed but it provides an easy graphical way to see if that “built-in” check box is checked.

### Intel Bluetooth (only follow these steps for an Intel based Bluetooth card) (works for Monterey and Ventura)

- Download IntelBluetooth-v2.2.0 from https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases
- Download BrcmPatchRAM-2.6.4-RELEASE from https://github.com/acidanthera/BrcmPatchRAM/releases
- Download Lilu-1.6.2-RELEASE from https://github.com/acidanthera/Lilu/releases
- unzip all
- open opencore → mount EFI → Mount Partition → Open Partition → Go to OC folder
- copy IntelBTPatcher.kext and IntelBluetoothFirmware.kext from IntelBluetooth zip file into Kext folder
- copy BlueToolFixup.kext from BrcmPatchRAM zip file into Kext folder
- copy Lilu.kext from Lilu-1.6.2 zip file into Kext folder
- open config.plist → go to Kernel
- drag from Kexts folder the .kext files that are NOT in the list under Kernel.
  - IntelBTPatcher and IntelBluetoothFirmware were the only two not already there for me.
- Put IntelBTPatcher above IntelBluetoothFirmware above BlueToolFixup
- Uncheck all the rest of the BCRM ones (not using them… i'm using Intel)
- Save, reboot, profit.

### Intel Wireless (works for Monterey and Ventura until the VM is rebooted… then requires server reboot and will work again until VM rebooted… still working on this one. Sonoma seems to suffer similar issues as Ventura but instead of just not working, it crashes the computer when clicking on “network” in settings.)

- Download Airportitlwm_v2.1.0_stable_Monterey.kext.zip from https://github.com/OpenIntelWireless/itlwm/releases/tag/v2.1.0
- Unzip folder
- open opencore → mount EFI → Mount Partition → Open Partition → Go to OC folder
- copy Airportitlwm.kext from zip into Kext folder
- open config.plist → go to Kernel
- drag from Kexts folder the airportitlwm.kext file in the list under Kernel.
- I put it all the way to the bottom.
- Go to Misc → Security
- Change SecureBootModel to Default
- Save → Shutdown
- Update VM xml file to passthru the wifi card
  ```xml
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
    <address domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
    </source>
    <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
  </hostdev>
  ```

- Profit

### Fix Memory Modules Misconfigured warning (suppress it) (only required when using Ventura and Sonoma with MacPro7,1)

- download RestrictEvents kext from https://github.com/acidanthera/RestrictEvents/releases
- Open OpenCore Configurator and go to Kernel
- Go to Download/Update kexts and drag the RestrictEvents.kext file into that window.
- Not sure of proper position in the list but I moved it to bottom
- Go to NVRAM then UUID (at bottom of list) then boot-args
- Add “revpatch=pci revblock=pci” to the list of boot-args
- Go to PlatformInfo → DataHub - Generic and check the “AdviseFeatures” block
- Save and reboot.

### Move VM disks

- This one is easy and just puts my mind at ease to create future Mac VMs without overwriting data AND to make everything look the same in my vms folder.
- copy the vm folder that contains your macos_disk.img file to it's new name.
  ```bash
  cp -r Macinabox\ Monterey/ monterey/
  ```

- make sure you edit the xml file to point to the new location.
- you can also change permissions to match other folders and edit groups and users.

### Move NVRAM to normal location

- open up XML to get the UUID of the vm.
- copy the custom NVRAM currently used to the default NVRAM folder for unraid and use that UUID while following the naming convention.
  ```bash
  cp Macinabox_VARS-pure-efi.fd /etc/libvirt/qemu/nvram/the-vms-UUID-number-here_VARS-pure-efi.fd
  ```

- make sure you edit the xml file to point to the new location.

### Point OVMF to normal location

- For this one, you can copy your custom one into the default folder.
  ```bash
  cp Macinabox_CODE-pure-efi-fd /usr/share/qemu/ovmf-x64/
  ```

- OR you can do what I did and use the one that is already in there (unraid's default one) and just follow the next step (make sure you get the one that is the same size).
- make sure you edit the xml file to point to the new location.

### csr-active-config

- This was set to 260F and I changed it to 00000000 to get rid of the error in the OpenCore Validator. I only did this in Sonoma and I haven't yet seen any issues it caused.
- Will report back!
