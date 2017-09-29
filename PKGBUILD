# Maintainer:  Marcin Wieczorek <marcin@marcin.co>
# Contributor: Thomas Kuther <archlinux@kuther.net>
# Contributor: Gianni Vialetto <gianni at rootcube dot net>
# Contributor: Paul N. Maxwell <msg dot maxwel at gmail dot com>
# Contributor: Thomas Mudrunka <harvie@@email..cz>
# Contributor: Max Fierke <max@maxfierke.com>

pkgbase=apparmor
pkgname=($pkgbase apparmor-parser apparmor-libapparmor apparmor-utils apparmor-profiles apparmor-pam apparmor-vim)
pkgver=2.11.0
#_majorver=${pkgver%.*}  # bleh, AUR...
_majorver=2.11
pkgrel=1
pkgdesc='Linux application security framework - mandatory access control for programs'
arch=('i686' 'x86_64')
license=('GPL')
url='http://wiki.apparmor.net/index.php/Main_Page'
makedepends=('flex' 'swig' 'perl' 'python' 'perl-locale-gettext' 'perl-rpc-xml' 'audit')

source=(https://launchpad.net/$pkgname/${_majorver}/${_majorver}/+download/${pkgname}-${pkgver}.tar.gz{,.asc}
        "apparmor_load.sh"
        "apparmor_unload.sh"
        "apparmor.service")

sha256sums=('b1c489ea11e7771b8e6b181532cafbf9ebe6603e3cb00e2558f21b7a5bdd739a'
            'SKIP'
            '124300162dab2a923c024b91c5a977dbee901376a22eefc64cad2f91319876d5'
            '9704478ae13fe1c3fb2747afac86c31b1b4593493f0e1425ae2b77d47878e32e'
            'eea47ec2a3fb0c1104193bed91586cfccda745f2e0a473f6d1d2a0d2fe42c413')

# 3D3664BB: AppArmor Development Team (AppArmor signing key) <apparmor@lists.ubuntu.com>
validpgpkeys=('3ECDCBA5FB34D254961CC53F6689E64E3D3664BB')

#Configuration
core_perl_dir='/usr/bin/core_perl'
export MAKEFLAGS+=" POD2MAN=${core_perl_dir}/pod2man"
export MAKEFLAGS+=" POD2HTML=${core_perl_dir}/pod2html"
export MAKEFLAGS+=" PODCHECKER=${core_perl_dir}/podchecker"
export MAKEFLAGS+=" PROVE=${core_perl_dir}/prove"
export MAKEFLAGS+=" PYTHON=python3"


prepare() {
  cd "${srcdir}/${pkgbase}-${pkgver}/parser"
  # avoid depend on texlive-latex
  sed -i -e 's/pdflatex/true/g' Makefile

  cd "${srcdir}/${pkgbase}-${pkgver}/utils"
  # Set Arch paths
  sed -e '/logfiles/ s/syslog /syslog.log /g' \
      -e '/logfiles/ s/messages/messages.log/g' \
      -e '/parser/ s# /sbin/# /usr/bin/#g' \
      -i logprof.conf
  # do not build/install vim file with utils package (causes ref to $srcdir and wrong location)
  sed -i '/vim/d' Makefile

  cd "${srcdir}/${pkgbase}-${pkgver}/profiles/apparmor.d"
  # /usr merge vs. profiles
  for i in `find . -name "*sbin*"`; do sed -i -e 's@sbin@bin@g' ${i} && mv ${i} ${i/sbin/bin}; done
  for i in klogd ping syslog-ng syslogd; do
    sed -e "s@/bin/${i}@/usr/bin/${i}@g" \
        -e "s@bin\.${i}@usr\.bin\.${i}@g" \
        -i bin.${i} && \
    mv bin.${i} usr.bin.${i}
  done
}

build() {
  msg2 "Building: apparmor-libapparmor"
  cd "${srcdir}/${pkgbase}-${pkgver}/libraries/libapparmor"
  unset PERL_MM_OPT
  NOCONFIGURE=1 ./autogen.sh
  ./configure --prefix=/usr --sbindir=/usr/bin --with-perl --with-python
  make

  cd "${srcdir}/${pkgbase}-${pkgver}"
  msg2 "Building: apparmor-parser"
  make -C parser

  msg2 "Building: apparmor-utils"
  make -C utils

  msg2 "Building: apparmor-profiles"
  make -C profiles

  msg2 "Building: apparmor-pam"
  make -C changehat/pam_apparmor

  msg2 "Building: apparmor-vim"
  make -C utils/vim -j1
}

package_apparmor() {
  pkgdesc='Linux application security framework - mandatory access control for programs (metapackage)'
  depends=(apparmor-parser apparmor-libapparmor apparmor-utils apparmor-profiles apparmor-pam apparmor-vim)
  optdepends=('linux-apparmor: an arch kernel with AppArmor patches')
  install='apparmor.install'
}

package_apparmor-parser() {
  pkgdesc='AppArmor parser - loads AA profiles to kernel module'
  depends=('apparmor-libapparmor')

  cd "${srcdir}/${pkgbase}-${pkgver}"
  make -C parser install DESTDIR=${pkgdir}
  mv "${pkgdir}/lib" "${pkgdir}/usr/lib"
  mv "${pkgdir}/sbin" "${pkgdir}/usr/bin"
}

package_apparmor-libapparmor() {
  pkgdesc='AppArmor library'
  makedepends=('swig' 'perl' 'python')
  depends=('python')

  cd "${srcdir}/${pkgbase}-${pkgver}"
  make -C libraries/libapparmor install DESTDIR="${pkgdir}"
  install -D -m644 "libraries/libapparmor/swig/perl/LibAppArmor.pm" "${pkgdir}/usr/lib/perl5/vendor_perl/"
}

package_apparmor-utils() {
  pkgdesc='AppArmor userspace utilities'
  depends=('perl' 'perl-locale-gettext' 'perl-term-readkey'
    'perl-file-tail' 'perl-rpc-xml' 'python')
  install='apparmor-utils.install'

  cd "${srcdir}/${pkgbase}-${pkgver}"
  make -C utils install DESTDIR="${pkgdir}" BINDIR="${pkgdir}/usr/bin"
  install -D -m755 "${srcdir}/apparmor_load.sh" "${pkgdir}/usr/bin/apparmor_load.sh"
  install -D -m755 "${srcdir}/apparmor_unload.sh" "${pkgdir}/usr/bin/apparmor_unload.sh"
  install -D -m644 "${srcdir}/apparmor.service" "${pkgdir}/usr/lib/systemd/system/apparmor.service"
}

package_apparmor-profiles() {
  pkgdesc='AppArmor sample pre-made profiles'
  depends=(apparmor-parser)

  # backup /etc/apparmor.d/* so using logprof is safe
  cd "${srcdir}/${pkgbase}-${pkgver}/profiles/apparmor.d"
  declare -a _profiles=(`find -type f|sed 's@./@etc/apparmor.d/@'`)
  backup=(`echo ${_profiles[@]}`)

  cd "${srcdir}/${pkgbase}-${pkgver}"
  make -C profiles install DESTDIR="${pkgdir}"
}

package_apparmor-pam() {
  pkgdesc='AppArmor PAM library'
  depends=('apparmor-libapparmor' 'pam')

  cd "${srcdir}/${pkgbase}-${pkgver}"
  make -C changehat/pam_apparmor install DESTDIR="${pkgdir}/usr"
  install -D -m644 changehat/pam_apparmor/README "${pkgdir}/usr/share/doc/apparmor/README.pam_apparmor"
}
package_apparmor-vim() {
  pkgdesc='AppArmor VIM support'
  depends=('vim')

  cd "${srcdir}/${pkgbase}-${pkgver}/utils/vim"
  install -D -m644 apparmor.vim \
    "${pkgdir}/usr/share/vim/vimfiles/syntax/apparmor.vim"
}

# vim:set ts=2 sw=2 et:
