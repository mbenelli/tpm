# Investigate TPM measurement under Qemu with OVMF+swtpm

According to the documentation at https://en.opensuse.org/Software_TPM_Emulator_For_QEMU,
the following steps have been executed:

  1. install `swtpm`
  2. create a QEMU virtual machine that use OVMF firmware
  3. TPM measurement in OVMF [failed]

## Install swtpm

Cloned and installed `swtpm` from the repo: https://github.com/stefanberger/swtpm

Before starting the virtual machine, `swtpm` is started in this way:

    swtpm socket \
          --tpmstate dir=./mytpm0 \
          --ctrl type=unixio,path=./mytpm0/swtpm-sockw
          --log level=20


## Create a QEMU virtual machine that use OVMF firmware

Create a virtual disk

    truncate -s 10G ovmf.disk2

Install a distro:

    sudo qemu-system-x86_64 \
         -cpu host \
         -enable-kvm \
         -cdrom ubuntu-20.04.2-live-server-amd64.iso \
         -drive format=raw,file=ovmf.disk2 \
         -m 1024

Copy the OVMF bios in the directory where vm will be started:

    cp /usr/share/OVMF/OVMF_VARS.fd ./OVMF_VARS.fd

Start the vm:

    sudo qemu-system-x86_64 \
         -cpu host \
         -enable-kvm \
         -drive if=pflash,format=raw,readonly,file=/usr/share/OVMF/OVMF_CODE.fd \
         -drive if=pflash,format=raw,file=./OVMF_VARS.fd \

         -drive format=raw,file=ovmf.disk2 \
         -m 1024 \
         -chardev socket,id=chrtpm,path=./mytpm0/swtpm-sock \
         -tpmdev emulator,id=tpm0,chardev=chrtpm \
         -device tpm-tis,tpmdev=tpm0

### Configuring the guest

Following https://wiki.ubuntu.com/UEFI/SecureBoot/Testing :

    sudo apt-get install sbsigntool openssl grub-efi-amd64-signed fwts
    sudo grub-install --uefi-secure-boot

After a reboot, download `sb-setup` and `keys`
from https://git.launchpad.net/qa-regression-testing/plain/notes_testing/secure-boot/sb-setup
copy them to `/tmp` and then:

   /tmp/sbsetup enroll

The output seems correct, and, after reboot SecureBoot is enabled all the
parameters in `Secure Boot Configuration` are as expected.

However the system does not boot anymore because of invalid signature.
I noticed, that differently to the document I used as a reference,
the signature was added to a pre-existing one, that was not overwritten.
Then, I repeted the steps above using `/tpm/sb-setup enroll canonical`
but without success.

The next steps would have been using simpler images, probably just a kernel
and a ramdisk, with known signatures, but I run out of time.

