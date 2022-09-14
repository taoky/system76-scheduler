# PKGBUILD file from https://aur.archlinux.org/cgit/aur.git/tree/PKGBUILD?h=system76-scheduler
# Original Maintainer: Donald Carr <d _at_ chaos-reins.com>

_pkgname=system76-scheduler
pkgname=${_pkgname}-taoky
pkgver=r101.0a87b92
pkgrel=1
pkgdesc='system76 userspace scheduler, forked to workaround some bugs'
arch=(x86_64)
url='https://github.com/taoky/system76-scheduler'
license=('MPL-2')
depends=(bcc-tools python-bcc)
makedepends=(git rust just linux-headers)
provides=(${_pkgname})
conflicts=(${_pkgname})
# source=(".")
# sha256sums=('6d9adf1a9136956710843111a229df46b9f03f3db00017715e063711180fd93d')
install=install.sh
backup=(
  'etc/system76-scheduler/assignments/default.ron'
)

pkgver() {
#   cd "$pkgname"
  printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

build() {
#   cd ${_pkgname}-${pkgver}
  export EXECSNOOP_PATH=/usr/share/bcc/tools/execsnoop
  just all
}

package() {
#   cd ${_pkgname}-${pkgver}
  just rootdir=${pkgdir} install
}
