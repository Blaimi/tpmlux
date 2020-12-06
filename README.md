# TL;DR;
* unlock LUKS-encrypted Partitions with TPM2.0
* if TPM-configuration has changed, ask for the password and reenable tpm-unlock
* unlock partitions via ssh
* stay secure
* Tested for fedora 33 but others may also work. Dual-/Multiboot-environments are not, and will not be, tested (by me).

# Motivation
## remote and WOL
1. I have a performant Tower-PC at home and a not so performant notebook when I'm not at home.
1. From time to time when I'm not at home, I want to use the performance of my PC to develop or test stuff on VMs.
1. When I'm not at home for weeks, I don't want to have the system running while it's not used.
1. I want to keep my principle to encrypt everything except my raspberry-pi which is used to watch TV and as jump-host.

## because I can
* I'm too lazy to type my password every time I boot my system, this should be done with the TPM
* I's a shame, that Bitlocker can do this but not GNU/Linux/LUKS

# target group
People who want to encrypt their data by principle and want to prevent any foreigners to have a look in the last
holiday-pictures when the device gets lost at the train station.

## not a target group
Paranoids. If you are afraid that some intelligence services are interested in your data,
legitimately or illegitimately, you should not use a TPM, you should probably not use an electronic device whenever
possible.

# Current state
## grub
Most of the information you can find about the current state of grub is outdated.
Grub2 comes with a tpm-module to enable measured boot.
At least on fedora 33 this module ist automatically inserted during boot if a TPM ist detected.
The grub2 tpm-Module measures the loaded binaries and commands (including kernel, initrd and cmdline)
and uses PCR#8 and PCR#9 for the results.
If you don't have a problem with manually inserting the key again on any change, this is the option to go.

### pros and cons
You can use this e.g. in combination with clevis (`clevis luks bind -d /dev/vda3 tpm2 '{"pcr_ids":"0,2,4,8,9,10"}'`).
This works well (`dnf remove clevis-pin-tpm2` https://github.com/latchset/clevis/issues/260).
The contents of PCR#8 and PCR#9 should be predictable but may be very hard to implement,
since all commmand are collected _they_ are not easy predictable
because they depend on the very specific system-configuration
(different harddrive-configurations, different uuids, different modules for different graphic cards, etc.).
Maybe it's possible to collect all commands from the last boot and replace the kernel-version.
Another problem is to differentiate between a temporary and a permanent change of the configuration
to decide if a new secret should be configured or not.
(Save the PCR-sums of the last time, compare PCR#{0-7,10-}
and configure a new secret if they changed but not PCR#{8,9}?)

Anyway, I gave up on this for the moment.
Not only because it's hard to predict the grub-commands
but also it's not possible to use the predicted PCR with the clevis-commands.

Without these two problems this would be the best solution,
not only because is has a infrastructure for secure-boot signing with shim.

## clevis
provides a kind of standardized way to unlock luks-drives via tpm.

### pros and cons
* easy to install and use
* can't use predicted PCR-records to configure keys

## systemd-boot
systemd-boot (formerly gummiboot) can be used as an alternative to grub.

### pros and cons
systemd-boot implements “measured-boot” and is enabled on fedora since 232-11 (2017-01-29).
As [systemd/systemd/#12716](https://github.com/systemd/systemd/issues/12716) points out, it doesn't measure the initrd
which makes “measured-boot” completely useless.
(If one can edit the kernel-parameters or initrd to change the init-process e.g. to `/bin/bash` without changing the
tpm-parameters, he or she can access the unlocked filesystem without any password.)

## grub and systemd-boot “dual-boot”
Besides the scripts in `/usr/lib/kernel/install.d/*.install` are not prepared for such a dual-boot,
it's very easy to get this.

```bash
bootctl install
cp /boot/{initramfs,vmlinuz}-* /boot/efi/
cp /boot/loader/entries/$(cat /etc/machine-id)-* /boot/efi/loader/entries/
# ${EDITOR-nano} /boot/efi/loader/loader.conf # to uncomment the timeout line to show the menu
```

It's not very useful except for testing purposes,
but it would not not be a big deal to write a script in `install.d` to automate this.

## efi-stub
You can build your own `.efi`-bootloader from necessary stuff with a stub-bootloader and `objcopy`.
The script `92-efistub.install` uses this principle to create or delete a appropriate `.efi`-file with a corresponding
efi-boot-menu-entry so you have an entry for each currently installed kernel.

## pros and cons
Building a single `.efi`-file which contains everything, from boot-loader, kernel and initrd to the cmdline,
everything which is necessary to ensure your own configuration (which requires passwords for any action after boot)
by the firmware which makes it easy to predict any future PCR-Contents.


# the update-key.service
TODO;
I want to write a service-file which checks if the boot was done via efi-stub or not.
And if it was an efi-stub-boot, it means that if you are aware of the unlock-password, any future unlock should be
automatically.

## how?
### detect if booting from efi-stub
I think that's not a very hard problem, I just don't yet know how. `bootctl` can help here probably.

### is any password-update even necessary
How can be detected if the device was unlocked by the tpm or by the password?
And if this is not possible, is it a problem to reconfigure the unlocking-mechanism on every boot?
This could be avoided by comparing some checksums or keys.

### updateing the key
removing and adding the key requires to give one of the remaining keys of the drive.
To get this running noninteractive, the plaintext password has to be saved or—much better—a second key
which is saved on a/the encrypted drive itself has to be used for updating the unlock-keys.

# unlock via ssh
TODO;
try https://github.com/gsauthof/dracut-sshd

# secure-boot
Using the TPM without secure-boot potentially insecure – but it would require a very good knowledge in the process
and your system.
If this is a problem for you,
use secure boot with your own keys or read chapter [not a target group](#not-a-target-group).

## TPM value-prediction
TODO; predict the PCR#?-contents and configure the unlock-keys on `kernel-install add`
and remove them on `kernel-install remove`

# Installation
TODO; So long: read the script(s) and pick what you need.

# Links
* [Full disc encryption on Arch Linux backed by TPM 2.0 (Pawit Pornkitprasan, 2019-06-09)](https://medium.com/@pawitp/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704)
* [The Correct Way to use Secure Boot with Linux (Pawit Pornkitprasan, 2019-07-27)](https://medium.com/@pawitp/the-correct-way-to-use-secure-boot-with-linux-a0421796eade)
* [Managing EFI Boot Loaders for Linux: Controlling Secure Boot (Rod Smith, 2015-02-22–2018-07-07)](http://www.rodsbooks.com/efi-bootloaders/controlling-sb.html)
* [tpm_futurepcr (Mantas Mikulėnas)](https://github.com/grawity/tpm_futurepcr)
* [Automatic LUKS 2 disk decryption with TPM 2 and Clevis on Fedora 31 (Kowalski7cc)](https://kowalski7cc.xyz/blog/luks2-tpm2-clevis-fedora31)
* [Systemd-boot install on Fedora 32 (Kowalski7cc)](https://kowalski7cc.xyz/blog/systemd-boot-fedora-32)
* [Configuring Secure Boot + TPM 2 (Karl O, 2018-06-21)](https://threat.tevora.com/secure-boot-tpm-2/)

# Testing
KVM can emulate EFI as well as a TPM.
To do this, you only have to install `swtmp-tools`, `edk2-ovmf` and `virt-manager` on fedora.
Don't forget to edit the machine before first boot and change the firmware to EFI and add the emulated TPM.
The current version seems to have a problem with plymouth (at least under wayland),
press ESC or send some keys like ctrl+alt+del when you can only see a black screen when booting.
I didn't have this problem with the server-edition.

# License
[![WTFPL 2.0](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
