# Maintainer: Frederic Bezies <fredbezies@gmail.com>
# Contributor: ajs124 < aur at ajs124 dot de>
# Contributor: Devin Cofer <ranguvar{AT]archlinux[DOT}us>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Sébastien "Seblu" Luttringer <seblu@seblu.net>

pkgbase=qemu-git
_gitname=qemu
pkgname=(qemu-headless-git qemu-block-iscsi-git)
pkgdesc="A generic and open source machine emulator and virtualizer. Git version."
pkgver=v2.11.0.rc1.r0.g1fa0f627d0
pkgrel=1
epoch=2
arch=(i686 x86_64)
license=(GPL2 LGPL2.1)
url="http://wiki.qemu.org/"
_headlessdeps=(gnutls libpng libaio numactl jemalloc xfsprogs libnfs
               lzo snappy curl vde2 libcap-ng spice libcacard usbredir)
depends=(dtc "${_headlessdeps[@]}")
makedepends=(spice-protocol python2 libiscsi git)
source=(git://git.qemu.org/qemu.git
        qemu.sysusers
        qemu-ga.service
        65-kvm.rules
        # https://github.com/spheenik/qemu
        0001-Remove-several-layers-of-latency-producing-shells.patch
        0002-auto-adjust-buffer-sizes.patch
        0003-expose-settings-set-reasonable-defaults.patch
        0004-reduce-logging.patch
        0005-improve-audio-timer.patch
        0006-query-HDA-with-a-separate-timer.patch
        0007-adjust-defaults.patch
        0008-Revert-improve-audio-timer.patch
        0009-fail-safety-in-qpa_run_out.patch
        0010-add-input-functionality-back-to-paaudio.c.patch
        0011-fix-wrong-calculation-of-input-buffer-transfer-size.patch
        0012-try-better-correction.patch
        0013-PA_STREAM_ADJUST_LATENCY-supported-since-PA-0.9.11-J.patch
        0014-leave-code-in-public-domain.patch
        0015-cleanup-HDA-timer-code.patch
        0016-cleanup-PA-driver-code.patch
        0017-expose-maxlength-parameter-for-recording-device.patch
        0018-do-not-change-unrelated-code.patch
        0019-ignore-old-fields-in-live-migration.patch
        0020-move-HDA_TIMER_TICKS-to-codec.patch
        0021-do-not-up-version.patch
        0022-debug-output-only-when-it-s-enabled.patch
        0023-checkpatch-cleanup-1.patch
        0024-checkpatch-cleanup-2.patch
        0025-checkpatch-cleanup-3.patch
        0026-checkpatch-cleanup-4.patch
        0027-whitespace.patch
        0028-enable-debug-output.patch
        0029-whitespace.patch
        0030-bigger-default-fraglength-to-reduce-PA-CPU-usage.patch
        clover.patch
        vcpu-pinning.patch)

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
    --disable-virglrenderer \
    --disable-brlapi
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
  install -Dm644 qemu.sysusers "$pkgdir/usr/lib/sysusers.d/qemu.conf"

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
            'dd43e2ef062b071a0b9d0d5ea54737f41600ca8a84a8aefbebb1ff09f978acfb'
            '0b4f3283973bb3bc876735f051d8eaab68f0065502a3a5012141fad193538ea1'
            '60dcde5002c7c0b983952746e6fb2cf06d6c5b425d64f340f819356e561e7fc7'
            '85360f7a233ebcfdf517032a24fd0b52a107cf83b25820675563d8e40e9726c2'
            '9f8cf72658eabd4ea76e9a24c2b02094384d85175bf266ac42684e59a1f4b04c'
            'ddeb184d60593fb28d1e25b61d374439020de7b28b696aa62af12873bbb58cfc'
            'cf9e32eea0d5f2a26fcfe3aa18b1700e0e50ba3c355bbd1a226d5663ffe59ab1'
            '61021e0457386c5982bfbcc585036c8708ffbe478b8a852e830360e8847e7298'
            'eb444a64a1c2337268c5390e02ff8775fd8897f0d8b519569da3adeb11f28ad4'
            '804991c9aed46e1f9548703119810fb360d1a4392fc64ee9112dd66557cbd148'
            '70b448986967216b0435b1a0e1940b9fd16e3678d3a605f5b7a7ee8c72704b84'
            '95fc1f452c3b61b6d3b2fcaf87301c9198b9719e5a37124051efb26afd983caa'
            'f9f711282faba63372fa7af0a69f01cc83cdece09e236a8bdca4446e7feb3ee7'
            '41ed0d91d21b7977d0c01c15cd82935217876e1fc97ce9e45b8d0207180a401b'
            '4a4cac8d927f7f73ecea8ddce0c9639fa10f9c8e808a977d786aba18b2b1df83'
            '9deb72cd0b8c1070fa1273e0d6f9ff31aee03df374926ecb9927889194d493f7'
            '9323d5b61001159f3685f5d089197124e278732ab9505c376cb8c8f58f513e03'
            'a2882c2ae158be8d25c03f3e181d5d8d5673da4db93dbdccb88cf0ea891eb44b'
            '7e1e2be79eb524117670e642d5f7dc92175d5f45cdb748dbc6d254d6c734eefd'
            '2b1c86d09af15df0f251e59890dfaa1c380fa341e6dd60256fefcb31f764c49a'
            '0208a9b96e19a39dede30800cecea6c79215519d02341e41fb17c02650b82b68'
            '35e50f6372430ebe9022276d7d0daee65e6ec57183a4bf7fec341686cbeea77a'
            '43d300455009ab930dab915faf29425768e3b31dc8eeb73bd9510a1397d5b005'
            '4763ab8550e8ab59a78ee583ead0765e6221a0f995901eeb3ea2c71aac12d6e0'
            '69e65d3865f4e2b3d22d42764e8e9e17411659128a05861f8dcdb609d26104a6'
            'c147c53f7a7086cd0779d13e9df08f31849bd1b5fe0b000c47ab4ba6b0815ad0'
            '77e3ee42020750cba9da2ca9d5ddb760876a56f2769a0bcf774ec2fe6e86df7a'
            '2179c73056a8631588a2f1fdce9e63c532d6fdab2a00d598bca1238266244f29'
            'ca087060e273622160435363b3f596c5fe938c787a800a6abb20753ee0c68f60'
            'd8f63c12d038b9070eecdae8a30c39037f982d75f2852757d98c1c8e08d30f2f'
            'c8400d2b82a64a20705d01b868fe7155ae6abccf35d354d259857baf2201612c'
            'd7fd8b99edc9886df535748e74bd4bb431cbb1f1dd532b5309f3ba626ea01ff2'
            '3e996108ffdaac51bd21fb49c13b811b48adc53f1413c77d0fd2d94f7887f5f0'
            'a718d43c1f34c5b0eaa99f28d525e64edfca287ffce4f65c43a9ff220fc7805b'
            '852ff04c6ae976a78ca1823fbb8eb61fc47bb281672851c0146bf9815d0b177f')
