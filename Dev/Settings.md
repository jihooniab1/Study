# Settings
Various settings

## Index
- [1. Windows 11 emulation](#windows-11-emulation)

## Windows 11 emulation
Build ovmf and emulate it with qemu for Windows 11 emulation. <br>
First, install qemu package

```
sudo apt install qemu-system qemu-utils qemu-block-extra​
```
Next, build ovmf with edk2
```
sudo apt update && sudo apt install -y \
    automake autoconf bash coreutils expect libtool sed \
    libssl-dev libtpms-dev fuse libfuse2 libfuse-dev \
    libglib2.0-0 libglib2.0-dev libjson-glib-dev \
    net-tools python3 python3-twisted \
    selinux-policy-dev socat \
    gnutls-bin libgnutls28-dev \
    libtasn1-6 libtasn1-bin libtasn1-dev \
    rpm libseccomp-dev nasm acpica-tools

git clone https://github.com/tianocore/edk2.git
cd edk2

git submodule update --init --recursive

make -C BaseTools

source ./edksetup.sh

build -p OvmfPkg/OvmfPkgX64.dsc -a X64 -b DEBUG -t GCC5 \
  -D SECURE_BOOT_ENABLE=TRUE \
  -D TPM_ENABLE=TRUE \
  -D TPM2_ENABLE=TRUE \
  -D DEBUG_ON_SERIAL_PORT=TRUE
```
Can find OVMF_CODE and OVMF_VARS in edk2/Build/OvmfX64/DEBUG_GCC5/FV/OVMF_* <br>

Let's build swtpm(+ libtpms)

```
sudo ln -s /usr/bin/python3 /usr/bin/python

git clone https://github.com/stefanberger/libtpms
cd libtpms
git checkout v0.10.0
./autogen.sh --with-tpm2 --with-openssl --prefix=/usr
make
make check
sudo make install

git clone https://github.com/stefanberger/swtpm
cd swtpm
git checkout v0.10.0
./autogen.sh --with-openssl --prefix=/usr
sudo make -j4
sudo make -j4 check
sudo make install 
```

Now, final step. First prepare Windows iso file from official homepage(https://www.microsoft.com/ko-kr/software-download/windows11) <br>

Make disk
```
qemu-img create -f qcow2 Win11.qcow2 128g
```

I prepared three scripts(hda.sh, tpm.sh, win.sh)

1. hda.sh
Put something inside **fat** folder and execute hda.sh. You can use it inside VM. 
```
rm -rf fat
dd if=/dev/zero of=hda-contents.img bs=1M count=10
mkfs.fat hda-contents.img
mkdir fat
sudo mount -o loop hda-contents.img fat
sudo cp -r hda-contents/* fat
sudo umount fat
```

2. tpm.sh
This initiates swtpm (for Win11)
```
sudo mkdir /tmp/emulated_tpm

sudo swtpm socket --tpmstate dir=/tmp/emulated_tpm \
--ctrl type=unixio,path=/tmp/emulated_tpm/swtpm-sock \
--log level=20 --tpm2

```

3. win.sh
This starts Windows 11 emulation with ovmf firmware 
```
sudo chown user:user /tmp/emulated_tpm/swtpm-sock
sudo qemu-system-x86_64 \
    -cdrom /home/user/Work/ovmf/Win.iso \
    -cpu host \
    -enable-kvm \
    -M q35 \
    -m 8192 \
    --chardev socket,id=chrtpm,path=/tmp/emulated_tpm/swtpm-sock \
    -tpmdev emulator,id=tpm0,chardev=chrtpm \
    -device tpm-tis,tpmdev=tpm0 \
    -smp 4 \
    -device intel-hda \
    -device hda-duplex \
    -usb \
    -device usb-tablet \
    -vga qxl \
    -nic user,ipv6=off,model=rtl8139,mac=84:1b:77:c9:03:a6 \
    -drive if=pflash,format=raw,readonly=on,file=OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=OVMF_VARS.fd \
    -drive file=hda-contents.img,format=raw \
    -drive file=Win11.qcow2,format=qcow2
```
