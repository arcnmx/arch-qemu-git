# Maintainer: Frederic Bezies <fredbezies@gmail.com>
# Contributor: ajs124 < aur at ajs124 dot de>
# Contributor: Devin Cofer <ranguvar{AT]archlinux[DOT}us>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Sébastien "Seblu" Luttringer <seblu@seblu.net>

pkgbase=qemu-git
_gitname=qemu
pkgname=(qemu-headless-git qemu-block-iscsi-git)
pkgdesc="A generic and open source machine emulator and virtualizer. Git version."
pkgver=v2.11.0.r53.g0ef0583d5a
pkgrel=1
epoch=3
arch=(i686 x86_64)
license=(GPL2 LGPL2.1)
url="http://wiki.qemu.org/"
_headlessdeps=(gnutls libpng libaio numactl jemalloc xfsprogs libnfs
               lzo snappy curl vde2 libcap-ng spice libcacard usbredir pulseaudio)
depends=(dtc "${_headlessdeps[@]}")
makedepends=(spice-protocol python2 libiscsi git)
source=(git://git.qemu.org/qemu.git
        qemu-ga.service
        65-kvm.rules
        # http://lists.nongnu.org/archive/html/qemu-devel/2017-12/msg01651.html
        ivshmem-0001.patch
        ivshmem-0002.patch
        ivshmem-0003.patch
        ivshmem-0004.patch
        # https://www.reddit.com/r/VFIO/comments/7dgf9d/ryzen_with_npt_patch_experiencing_fps_drops/dpxynub/
        k10smt.patch
        # https://github.com/spheenik/qemu
        pa-timer.patch::https://github.com/qemu/qemu/compare/master...spheenik:master.patch
        clover.patch
        vcpu-pinning.patch
        )

case $CARCH in
  i?86) _corearch=i386 ;;
  x86_64) _corearch=x86_64 ;;
esac

pkgver() {
  cd "${srcdir}/${_gitname}"
  git describe --long | sed 's/\([^-]*-g\)/r\1/;s/-/./g'
}

prepare() {
  cd "${srcdir}/${_gitname}"
  mkdir -p build-{full,headless}
  mkdir -p extra-arch-{full,headless}/usr/{bin,share/qemu}

  #cd "${srcdir}/${_gitname}"
  sed -i 's/vte-2\.90/vte-2.91/g' configure

  for p in "${srcdir}/"*.patch; do
    patch -p1 < "$p"
  done
}

build() {
  _build headless \
    --audio-drv-list="pa alsa" \
    --disable-bluez \
    --disable-sdl \
    --disable-gtk \
    --disable-vte \
    --disable-opengl \
    --disable-virglrenderer
}

_build() (
  cd ${srcdir}/${_gitname}/build-$1

  # qemu vs. make 4 == bad
  export ARFLAGS=rv

  # http://permalink.gmane.org/gmane.comp.emulators.qemu/238740
  export CFLAGS+=" -fPIC"

  ../configure \
    --prefix=/usr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --libexecdir=/usr/lib/qemu \
    --python=/usr/bin/python2 \
    --smbd=/usr/bin/smbd \
    --with-gtkabi=3.0 \
    --with-sdlabi=2.0 \
    --enable-modules \
    --enable-jemalloc \
    --disable-werror \
    "${@:2}"

  make
)

package_qemu-git() {
  optdepends=('qemu-arch-extra-git: extra architectures support')
  conflicts=('qemu-headless' 'qemu' 'kvm' 'kvm-git' 'qemu-spice')
  provides=('qemu-headless' 'qemu' 'qemu-kvm' 'qemu-spice')
  replaces=(qemu-kvm)

  _package full
}

package_qemu-headless-git() {
  pkgdesc="QEMU without GUI. Git version."
  depends=("${_headlessdeps[@]}")
  optdepends=('qemu-headless-arch-extra-git: extra architectures support')
  conflicts=('qemu-headless')

  _package headless
}

_package() {
  optdepends+=('ovmf: Tianocore UEFI firmware for qemu'
               'samba: SMB/CIFS server support'
               'qemu-block-iscsi-git: iSCSI block support'
               'qemu-block-rbd-git: RBD block support'
               'qemu-block-gluster-git: glusterfs block support')
  install=qemu.install
  options=(!strip)

  make -C ${srcdir}/${_gitname}/build-$1 DESTDIR="$pkgdir" install "${@:2}"

  # systemd stuff
  install -Dm644 65-kvm.rules "$pkgdir/usr/lib/udev/rules.d/65-kvm.rules"

  # remove conflicting /var/run directory
  cd "$pkgdir"
  rm -r var

  cd usr/lib
  tidy_strip

  # bridge_helper needs suid
  # https://bugs.archlinux.org/task/32565
  chmod u+s qemu/qemu-bridge-helper

  # remove split block modules
  rm qemu/block-{iscsi,rbd,gluster}.so

  cd ../bin
  tidy_strip

  # remove extra arch
  for _bin in qemu-*; do
    [[ -f $_bin ]] || continue

    case ${_bin#qemu-} in
      # guest agent
      ga) rm "$_bin"; continue ;;

      # tools
      img|io|nbd) continue ;;

      # core emu
      system-${_corearch}) continue ;;
    esac

    mv "$_bin" "$srcdir/$_gitname/extra-arch-$1/usr/bin"
  done

  cd ../share/qemu
  for _blob in *; do
    [[ -f $_blob ]] || continue

   case $_blob in
      # provided by seabios package
      bios.bin|acpi-dsdt.aml|bios-256k.bin|vgabios-cirrus.bin|vgabios-qxl.bin|\
      vgabios-stdvga.bin|vgabios-vmware.bin) rm "$_blob"; continue ;;


  # iPXE ROMs
      efi-*|pxe-*) continue ;;

      # core blobs
      kvmvapic.bin|linuxboot*|multiboot.bin|sgabios.bin|vgabios*) continue ;;

      # Trace events definitions
      trace-events*) continue ;;

      # Logos
      *.bmp|*.svg) continue ;;
    esac

    mv "$_blob" "$srcdir/$_gitname/extra-arch-$1/usr/share/qemu"
  done
}

package_qemu-arch-extra-git() {
  pkgdesc="QEMU for foreign architectures. Git version."
  depends=(qemu)
  provides=(qemu-arch-extra)
  conflicts=(qemu-arch-extra)
  options=(!strip)

  mv $srcdir/$_gitname/extra-arch-full/usr "$pkgdir"
}

package_qemu-headless-arch-extra-git() {
  pkgdesc="QEMU without GUI, for foreign architectures. Git version."
  depends=(qemu-headless)
  options=(!strip)
  conflicts=(qemu-headless-arch-extra)
  provides=(qemu-headless-arch-extra)

  mv $srcdir/$_gitname/extra-arch-headless/usr "$pkgdir"
}

package_qemu-block-iscsi-git() {
  pkgdesc="QEMU iSCSI block module. Git version."
  depends=(glib2 libiscsi jemalloc)
  conflicts=(qemu-block-iscsi)
  provides=(qemu-block-iscsi)

  install -D $srcdir/$_gitname/build-headless/block-iscsi.so "$pkgdir/usr/lib/qemu/block-iscsi.so"
}

package_qemu-block-rbd-git() {
  pkgdesc="QEMU RBD block module. Git version."
  depends=(glib2 ceph)
  conflicts=(qemu-block-rbd)
  provides=(qemu-block-rbd)

  install -D $srcdir/$_gitname/build-full/block-rbd.so "$pkgdir/usr/lib/qemu/block-rbd.so"
}

package_qemu-block-gluster-git() {
  pkgdesc="QEMU GlusterFS block module. Git version."
  depends=(glib2 glusterfs)
  conflicts=(qemu-block-gluster)
  provides=(qemu-block-gluster)

  install -D $srcdir/$_gitname/build-full/block-gluster.so "$pkgdir/usr/lib/qemu/block-gluster.so"
}

package_qemu-guest-agent-git() {
  pkgdesc="QEMU Guest Agent. Git version."
  depends=(gcc-libs glib2)
  conflicts=(qemu-guest-agent)
  provides=(qemu-guest-agent)

  install -D $srcdir/$_gitname/build-full/qemu-ga "$pkgdir/usr/bin/qemu-ga"
  install -Dm644 $srcdir/qemu-ga.service "$pkgdir/usr/lib/systemd/system/qemu-ga.service"
  install -Dm755 "$srcdir/$_gitname/scripts/qemu-guest-agent/fsfreeze-hook" "$pkgdir/etc/qemu/fsfreeze-hook"
}

# vim:set ts=2 sw=2 et:

# makepkg -g >> PKGBUILD
sha256sums=('SKIP'
            '0b4f3283973bb3bc876735f051d8eaab68f0065502a3a5012141fad193538ea1'
            '60dcde5002c7c0b983952746e6fb2cf06d6c5b425d64f340f819356e561e7fc7'
            '758ae180dc571a91e3d293be98e13fac3f9ecba2970a7a10027ffa26c681691a'
            '87dcd51a3500382c2b0e1462eb99c21ec355ab0d2b41705f7e0a356827accceb'
            'fd619e15797dd38bdfe822d36b3d41064b5585e0961a7cde0cf88c21c9dcd466'
            'e5f81c6df9f8344b78f70023bafa87fe2e950c0302ab234da08309e45c9f1be6'
            'd1af59a159f08340ed445d2b39a0d42154727662cf5c52a0e1c4f3dbdc1910cd'
            '132b10125c869d0f0b3a808257763b5c692fbef78eb9de702a5e568efb9b8b33'
            'a718d43c1f34c5b0eaa99f28d525e64edfca287ffce4f65c43a9ff220fc7805b'
            '852ff04c6ae976a78ca1823fbb8eb61fc47bb281672851c0146bf9815d0b177f')
