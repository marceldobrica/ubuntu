# ubuntu

## Install Ubuntu Server LTS
### 1. create startup-disk 
Download last LTS server version www.ubuntu.com/download/server
Download balena/etcher or sudo apt install usb-creator-gtk

### If installation stops at the begining
You may have problems with old UEFI system - upgrade or press e to edit grub. Some EFI are so old so can't be upgraded. For me I need to add nolapic. Asked chatGPT for that.