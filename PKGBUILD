# Maintainer: Noa Himesaka
# Previous Maintainer: Redecorating
# Contributors: There are many, see `grep -h "From:" *.patch|sort|uniq -c`.
#               Additionally, MrARM and Ronald Tschalär wrote apple-bce and
#               apple-ibridge drivers, respectively.

pkgbase="linux-t2-rust"
_pkgver=6.11
pkgver=${_pkgver}.0
_srcname=linux-${_pkgver}
pkgrel=1
archrel=1
pkgdesc='Linux kernel for T2 Macs'
_srctag=v${_pkgver%.*}-${_pkgver##*.}
url="https://github.com/archlinux/linux/commits/$_srctag"
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc
  cpio
  gettext
  git
  jq
  llvm
  lld
  libelf
  pahole
  perl
  python
  rustup
  tar
  xz

  # htmldocs
  graphviz
  imagemagick
  python-sphinx
  texlive-latexextra
  xmlto
)
conflicts=('apple-gmux-t2-dkms-git')
replaces=('apple-gmux-t2-dkms-git')
options=('!strip')
_srcname="linux-${_pkgver}-arch${archrel}"
T2_PATCH_HASH=54b4f914930d92cf0b94601b402ec93f54a76390
source=(
  https://github.com/archlinux/linux/archive/refs/tags/v${_pkgver}-arch${archrel}.tar.gz
  config  # the main kernel config file
  rust-toolchain

  # t2linux Patches
  patches::git+https://github.com/t2linux/linux-t2-patches#commit=$T2_PATCH_HASH

  # Additional patches
  9001-AsahiLinux.patch
  9002-Rust-Improvements.patch
  9003-Rust-File-Abstraction.patch
  9004-Rust-Opaque-try_ffi_init.patch
)
validpgpkeys=(
  ABAF11C65A2970B130ABE3C479BE3E4300411886  # Linus Torvalds
  647F28654894E3BD457199BE38DBBDC86092693E  # Greg Kroah-Hartman
  A2FF3A36AAA56654109064AB19802F8B0D70FC30  # Jan Alexander Steffens (heftig)
  C7E7849466FE2358343588377258734B41C31549  # David Runge <dvzrv@archlinux.org>
)

export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

_make() {
  test -s version
  make LLVM=1 KERNELRELEASE="$(<version)" "$@"
}

prepare() {
  cd $_srcname

  echo "Setting version..."
  echo "-Rust-T2" > localversion.10-codename
  echo "-$pkgrel" > localversion.20-pkgrel
  echo "${pkgbase#linux}" > localversion.30-pkgname

  echo "Setting up Rust compiler version..."
  rustup install 1.81.0
  rustup override set 1.81.0

  echo "Installing rust-src..."
  rustup component add rust-src

  echo "Installing rust-bindgen..."
  cargo install --locked --version 0.65.1 bindgen-cli

  echo "Verifying that Rust support is available..."
  make LLVM=1 rustavailable

  make LLVM=1 defconfig
  make LLVM=1 -s kernelrelease > version
  make LLVM=1 mrproper

  t2linux_patches=$(ls $srcdir/patches | grep -e \.patch$)
  mv $srcdir/patches/*.patch $srcdir/
  local src
  for src in "${source[@]}" $t2linux_patches; do
    src="${src%%::*}"
    src="${src##*/}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  echo "Setting config..."
  cp ../config .config
  cat $srcdir/patches/extra_config >> .config
  _make olddefconfig
  diff -u ../config .config || :

  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  local installed_builddir="/usr/lib/modules/$(<version)/build"

  _make "KRUSTFLAGS=-Z unstable-options --remap-path-prefix $srcdir/$_srcname=$installed_builddir" \
    htmldocs all rust-analyzer
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules with support for Rust written modules"
  depends=(
    coreutils
    initramfs
    kmod
  )
  optdepends=(
    'wireless-regdb: to set the correct wireless channels of your country'
    'linux-firmware: firmware images needed for some devices'
  )
  provides=(
    KSMBD-MODULE
    VIRTUALBOX-GUEST-MODULES
    WIREGUARD-MODULE
  )
  replaces=(
    virtualbox-guest-modules-arch
    wireguard-arch
  )

  cd $_srcname
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(_make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  ZSTD_CLEVEL=19 _make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  # remove build link
  rm "$modulesdir"/build
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local installed_builddir="/usr/lib/modules/$(<version)/build"
  local builddir="$pkgdir$installed_builddir"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # required when STACK_VALIDATION is enabled
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # required when DEBUG_INFO_BTF_MODULES is enabled
  install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  # Rust support
  echo "Installing Rust files..."
  install -Dt "$builddir/rust" -m644 scripts/target.json
  install -Dt "$builddir/rust" -m644 rust/*.rmeta
  install -Dt "$builddir/rust" -m644 rust/*.so
  install -Dt "$builddir/rust" -m644 rust/Makefile
  install -Dt "$builddir" -m644 ../rust-toolchain

  install -Dt "$builddir" -m644 rust-project.json
  sed -i "s|$srcdir/$_srcname|$installed_builddir|g" "$builddir/rust-project.json"

  local file
  while read -rd '' file; do
    echo "Installing $file to $builddir/$file"
    install -D -m644 "$file" "$builddir/$file"
  done < <(find rust -type f -name '*.rs' -print0)

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

_package-docs() {
  pkgdesc="Documentation for the $pkgdesc kernel"
  provides=(linux-docs)

  cd $_srcname
  local installed_builddir="/usr/lib/modules/$(<version)/build"
  local builddir="$pkgdir$installed_builddir"

  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  # Replace $srcdir/$_srcname with $installed_builddir in all html files in $builddir/Documentation/rust/rustdoc
  local file
  while read -rd '' file; do
    sed -i "s|$srcdir/$_srcname|$installed_builddir|g" "$file"
  done < <(find "$builddir/Documentation/rust/rustdoc" -type f -name '*.html' -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
  "$pkgbase-docs"
)
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

sha256sums=('364bb32cadd41401f686dd58c58a08432f927aabdb35e2482ba62a4a559bd24d'
            '148c8cbe8e9ca3dbe1a19fc696882a3ec81a31fe599a203253e69cd9c24fbab7'
            '7ff8654efc550e7ed15bf7212a6d7358f98ad1455beafa46d26ecbb21fc062bf'
            'SKIP'
            'f09efb2507226c6a4d2a6e65a667460027b492469828f3a0e954b6702fa8d44f'
            '05384345f829abd75dffd9a0888167606f09d18500e8663ef6e3e542beb2948a'
            'f5800e38366fd6e8975311c6ac6a4fdc2b01e133098c50c690f010bac697b245'
            'd5fce1f295717acf313f50968b0e7697eb549fd06becefd60357902a1616b255')
# vim:set ts=8 sts=2 sw=2 et:
