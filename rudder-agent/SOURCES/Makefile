#####################################################################################
# Copyright 2015 Normation SAS
#####################################################################################
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, Version 3.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
#
#####################################################################################

.DEFAULT_GOAL := build

# Versions to use
RUDDER_VERSION_TO_PACKAGE = <put Rudder version or version-snapshot here>
RUDDER_MAJOR_VERSION := $(shell echo ${RUDDER_VERSION_TO_PACKAGE} | cut -d'.' -f 1-2)

CFENGINE_RELEASE = 3.10.2
CFENGINE_SHA1 = c40adf04c7ef9a6d115957484a901a48f1dcf6a1
FUSION_RELEASE = 2.3.19
FUSION_SHA1 = a01af094613a69feac778c2402cb639cb227eb1f
LMDB_RELEASE = 0.9.21
LMDB_SHA1 = 54d10ee9afaba8db75c8c73832da10b0a47e5807
OPENSSL_RELEASE = 1.0.2n
OPENSSL_SHA1 = 0ca2957869206de193603eca6d89f532f61680b1
PERL_VERSION = 5.22.0
PERL_SHA1 = e4c9e40d18efa7368e77343e0fd3339ca87e34f8
PCRE_RELEASE = 8.38
PCRE_SHA1 = 3ab418d0026c2a4e693ec783bd60660debc32b8f
CPANMINUS_RELEASE = 1.7036
CPANMINUS_SHA1 = a12043498ab54d9abdc58d1ebb4988da3bd515db
#CPAN_URL = https://www.cpan.org
CPAN_URL = $(HTTPS)://www.normation.com/tarball/cpan/$(RUDDER_MAJOR_VERSION)

# Defauth PATHs
RUDDER_DIR = /opt/rudder
DESTDIR = $(CURDIR)/target
INSTALL_DIR = $(DESTDIR)$(RUDDER_DIR)

TRY_LDFLAGS=-pie -Wl,-z,relro -Wl,-z,now
TRY_CFLAGS=-fPIE -fstack-protector

# GENERIC_GET: do not remove this line
# This is a generic get code that is common to many of our Makefiles.
# The tag is here to find them.
# If you change this code, change all the places where you can find this tag too.
# It is duplicated because it is the code that is used to manage dependency
# to other repositories, hence allowing deduplication of other code.
# Autodetect wget, curl or fetch usage and proxy configuration
# Usage: $(GET) <destination_file> <url>
PROXY_ENV = $(if $(PROXY), http_proxy=$(PROXY) ftp_proxy=$(PROXY))
# No accept encoding to prevent the webserver form altering the distributed file
WGET = wget -q --header="accept-encoding:" -O
CURL = curl -s -L -o
FETCH = fetch -q -o
ifneq (,$(wildcard /usr/bin/curl))
_GET = $(PROXY_ENV) $(CURL)
GET_OPT = --curl
CPAN_OPT = --curl --no-wget
else
ifneq (,$(wildcard /usr/bin/fetch))
_GET = $(PROXY_ENV) $(FETCH)
else
_GET = $(PROXY_ENV) $(WGET)
GET_OPT = --wget
CPAN_OPT = --wget
endif
endif
# We are using sha1 as sha256 is not supported on RHEL3
# Not using the sha1sum command as it is not available on AIX
# Pass an empty string to skip hash check
GET=get() { $(_GET) "$$1.part" "$$2" && { openssl dgst -sha1 "$$1.part" | grep -q "$$3" || { echo "Wrong checksum, aborting"; exit 1; }; } && mv "$$1.part" "$$1"; }; get

# Autodetect presence of gnu tools
TAR := $(shell type gtar >/dev/null 2>&1 && echo gtar || echo tar)
PATCH := $(shell type gpatch >/dev/null 2>&1 && echo gpatch || echo patch)
SED := $(shell type gsed >/dev/null 2>&1 && echo gsed || echo sed)
INSTALL := $(shell type ginstall >/dev/null 2>&1 && echo ginstall || echo install)
CP_A := cp -rp
FIND := find

# Use parameters to guess what should be packaged
BUILD_DEPS := ./rudder.8.gz
INSTALL_DEPS :=
# default: use https for urls
ifeq (false,$(USE_HTTPS))
HTTPS:=http
else
HTTPS:=https
endif
# default: do not build openssl
ifeq (false,$(USE_SYSTEM_OPENSSL))
BUILD_DEPS := $(BUILD_DEPS) build-ssl
INSTALL_DEPS := $(INSTALL_DEPS) install-ssl
OPENSSL_ARG := --with-openssl=$(CURDIR)/build-ssl/opt/rudder
endif
# default: build lmdb
ifneq (true,$(USE_SYSTEM_LMDB))
BUILD_DEPS := $(BUILD_DEPS) build-lmdb
INSTALL_DEPS := $(INSTALL_DEPS) install-lmdb
LMDB_ARG := --with-lmdb=$(CURDIR)/build-lmdb/opt/rudder
endif
# default: do not build pcre
ifeq (false,$(USE_SYSTEM_PCRE))
BUILD_DEPS := $(BUILD_DEPS) build-pcre
INSTALL_DEPS := $(INSTALL_DEPS) install-pcre
PCRE_ARG := --with-pcre=$(CURDIR)/build-pcre/opt/rudder
endif
# default: build perl
ifneq (true,$(USE_SYSTEM_PERL))
BUILD_DEPS := $(BUILD_DEPS) build-perl
FUSION_DEPS := install-perl
PERL:=$(INSTALL_DIR)/bin/perl
else
FUSION_DEPS := cpan-source
PERL := perl
endif
# default: embed fusion
ifneq (true,$(USE_SYSTEM_FUSION))
INSTALL_DEPS := $(INSTALL_DEPS) install-fusion
endif

# Hack needed because openssl is not guessing correctly the architecture on solaris 32bits
ifeq (i386-pc-solaris2.10,$(shell ./config.guess))
SSL_CONFIGURE:=./Configure solaris-x86-gcc
# Hack needed because openssl is not guessing correctly the architecture on debian 32 bits chroot
else
ifeq (i386,$(shell [ -x /usr/bin/dpkg ] && dpkg --print-architecture))
SSL_CONFIGURE:=./Configure linux-generic32
else
SSL_CONFIGURE:=./config
endif
endif

# download dependencies
depend: localdepends
localdepends: initial-promises initial-ncf rudder-agent-utilities perl-$(PERL_VERSION) fusioninventory-agent cfengine-source openssl-source lmdb-source pcre-source rudder-agent.cron ./rudder.8.gz

# build binaries
build: build-cfengine

# install to DESTDIR
install: build $(INSTALL_DEPS) install-cfengine initial-promises initial-ncf rudder-agent-utilities rudder-agent.cron
	# Directories
	mkdir -p $(INSTALL_DIR)/etc
	mkdir -p $(INSTALL_DIR)/bin
	mkdir -p $(INSTALL_DIR)/share/commands
	mkdir -p $(INSTALL_DIR)/share/lib
	mkdir -p $(DESTDIR)/var/rudder/cfengine-community/bin
	mkdir -p $(DESTDIR)/var/rudder/cfengine-community/inputs
	mkdir -p $(DESTDIR)/var/rudder/cfengine-community/ppkeys
	chmod 700 $(DESTDIR)/var/rudder/cfengine-community/ppkeys
	mkdir -p $(DESTDIR)/var/rudder/tmp
	mkdir -p $(DESTDIR)/var/rudder/tools
	mkdir -p $(DESTDIR)/var/rudder/ncf/local
	mkdir -p $(DESTDIR)/var/rudder/ncf/common
	mkdir -p $(DESTDIR)/var/log/rudder/install
	mkdir -p $(DESTDIR)/var/log/rudder/agent-check
	mkdir -p $(DESTDIR)/usr/bin
	mkdir -p $(DESTDIR)/etc/cron.d

	# Systemd files
ifeq (true,$(USE_SYSTEMD))
	mkdir -p $(DESTDIR)/lib/systemd/system
	$(INSTALL) -m 644 systemd/rudder-agent.service $(DESTDIR)/lib/systemd/system/rudder-agent.service
	$(INSTALL) -m 644 systemd/rudder-cf-execd.service $(DESTDIR)/lib/systemd/system/rudder-cf-execd.service
	$(INSTALL) -m 644 systemd/rudder-cf-serverd.service $(DESTDIR)/lib/systemd/system/rudder-cf-serverd.service
endif

	# Init files
ifneq (true,$(USE_SYSTEMD))
ifneq (true,$(NO_INIT))
	mkdir -p $(DESTDIR)/etc/init.d
	mkdir -p $(DESTDIR)/etc/default
	$(INSTALL) -m 755 rudder-agent.init $(DESTDIR)/etc/init.d/rudder-agent
	$(INSTALL) -m 644 rudder-agent.default $(DESTDIR)/etc/default/rudder-agent
endif
endif

	# cron files
ifneq (true,$(NO_CRON))
	$(INSTALL) -m 644 rudder-agent.cron $(DESTDIR)/etc/cron.d/rudder-agent
endif

	# Wrapper script
	$(INSTALL) -m 755 run-inventory $(INSTALL_DIR)/bin/

	# Signature script
	$(INSTALL) -m 755 rudder-sign $(INSTALL_DIR)/bin/
	$(INSTALL) -m 755 rudder-signature-check $(INSTALL_DIR)/bin/
	$(INSTALL) -m 755 signature.sh $(INSTALL_DIR)/bin/ # for backwards compatibility, remove in the next major version after the next LTS after 3.1 is announced

	# Install an empty uuid.hive file before generating an uuid
	$(INSTALL) -m 644 uuid.hive $(INSTALL_DIR)/etc/

	# Install /etc/ld.so.conf.d/rudder.conf in order to use libraries
	# contained in /opt/rudder/lib like LMDB or OpenSSL
	# (ld.so.conf.d is not supported on RHEL/CentOS 3 and aix)
ifneq (true,$(NO_LDSO))
	mkdir -p $(DESTDIR)/etc/ld.so.conf.d
	$(INSTALL) -m 644 rudder.conf $(DESTDIR)/etc/ld.so.conf.d/
endif

	mkdir -p $(INSTALL_DIR)/share/package-scripts
	$(INSTALL) -m 755 rudder-agent-postinst $(INSTALL_DIR)/share/package-scripts/
	$(INSTALL) -m 755 check-rudder-agent $(INSTALL_DIR)/bin/
	$(INSTALL) -m 755 vzps.py $(INSTALL_DIR)/bin/
	$(INSTALL) -m 755 rudder-perl $(INSTALL_DIR)/bin/
	$(INSTALL) -m 755 rudder-uuidgen $(INSTALL_DIR)/bin/

	# Rudder agent utilities
	$(INSTALL) -m 755 rudder-agent-utilities/bin/rudder $(INSTALL_DIR)/bin/
	$(INSTALL) -m 755 rudder-agent-utilities/share/commands/* $(INSTALL_DIR)/share/commands/
	$(INSTALL) -m 755 rudder-agent-utilities/share/lib/* $(INSTALL_DIR)/share/lib/
	$(INSTALL) -m 755 rudder-agent-utilities/agent-capabilities $(INSTALL_DIR)/etc/

	# Install /etc/bash_completion.d/rudder.sh in order to autocomplete rudder commands
	mkdir -p $(DESTDIR)/etc/bash_completion.d
	$(INSTALL) -m 644 rudder-agent-utilities/etc/bash_completion.d/rudder.sh $(DESTDIR)/etc/bash_completion.d/

	# Install rudder man page
	mkdir -p $(INSTALL_DIR)/share/man/man8/
	$(INSTALL) -m 644 rudder.8.gz $(INSTALL_DIR)/share/man/man8/

	# Create a symlink to make "rudder" available as part of the default PATH
	ln -sf $(RUDDER_DIR)/bin/rudder $(DESTDIR)/usr/bin/rudder

	# Install a profile script to make cf-* part of the PATH
ifneq (true,$(NO_PROFILE))
	mkdir -p $(DESTDIR)/etc/profile.d
	$(INSTALL) -m 644 rudder-agent.sh $(DESTDIR)/etc/profile.d/
endif

	cp -r initial-promises/ $(INSTALL_DIR)/share/
	cp -r initial-ncf/ $(INSTALL_DIR)/share/

# clean 
clean: localclean
localclean:
	rm -rf $(DESTDIR)
	rm -rf build-ssl build-lmdb build-cfengine build-perl fusion-modules build-cfengine-stamp
	rm -rf rudder-sources
	rm -f ./rudder.8.gz
	[ -d cfengine-source ] && cd cfengine-source && $(MAKE) clean 2>/dev/null || true
	[ -d lmdb-source ] && cd lmdb-source && $(MAKE) clean 2>/dev/null || true
	[ -d openssl-source ] && cd openssl-source && $(MAKE) clean 2>/dev/null || true
	[ -d perl-$(PERL_VERSION) ] && cd perl-$(PERL_VERSION) && $(MAKE) distclean 2>/dev/null || true

# standard name is mrproper
mrproper: veryclean
veryclean:
	rm -f rudder-agent.cron
	rm -f ./rudder-sources.tar.bz2
	rm -rf initial-promises initial-ncf rudder-agent-utilities fusioninventory-agent
	rm -rf cfengine-source lmdb-source openssl-source perl-$(PERL_VERSION)

##########################################
# Dependencies separated into components #
##########################################

cfengine-source:
ifeq ($(shell ../../build-caching get ./cfengine-source/ --force-config name=cfengine-src version=$(CFENGINE_RELEASE) >/dev/null 2>&1 || echo KO), KO)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	# Original URL: https://cfengine-package-repos.s3.amazonaws.com/tarballs/cfengine-$(CFENGINE_RELEASE).tar.gz
	$(GET) $(TMP_DIR)/cfengine.tgz $(HTTPS)://www.normation.com/tarball/cfengine/cfengine-$(CFENGINE_RELEASE).tar.gz $(CFENGINE_SHA1)
	gunzip < $(TMP_DIR)/cfengine.tgz | $(TAR) xf -
	mv ./cfengine-$(CFENGINE_RELEASE) ./cfengine-source
	../../build-caching put ./cfengine-source/ --force-config name=cfengine-src version=$(CFENGINE_RELEASE)
	rm -rf $(TMP_DIR)
endif

	# Apply patches
	for PATCHNAME in patches/cfengine/*.patch; do echo "Applying $$PATCHNAME..."; $(PATCH) -d ./cfengine-source -p1 < $$PATCHNAME; echo ""; done
	# Make sure there were no rejects
	test `$(FIND) ./cfengine-source -name \*.rej | wc -l` = 0

openssl-source:
ifeq ($(shell ../../build-caching get ./openssl-source/ --force-config name=openssl-src version=$(OPENSSL_RELEASE) >/dev/null 2>&1 || echo KO), KO)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	# Original URL: https://www.openssl.org/source/openssl-$(OPENSSL_RELEASE).tar.gz
	$(GET) $(TMP_DIR)/openssl.tgz $(HTTPS)://www.normation.com/tarball/openssl/openssl-$(OPENSSL_RELEASE).tar.gz $(OPENSSL_SHA1)
	gunzip < $(TMP_DIR)/openssl.tgz | $(TAR) xf -
	mv ./openssl-$(OPENSSL_RELEASE) ./openssl-source
	../../build-caching put ./openssl-source/ --force-config name=openssl-src version=$(OPENSSL_RELEASE)
	rm -rf $(TMP_DIR)
endif

lmdb-source:
ifeq ($(shell ../../build-caching get ./lmdb-source/ --force-config name=lmdb-src version=$(LMDB_RELEASE) >/dev/null 2>&1 || echo KO), KO)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	# Original URL: http://ftp.fr.debian.org/debian/pool/main/l/lmdb/lmdb_$(LMDB_RELEASE).orig.tar.xz
	$(GET) $(TMP_DIR)/lmdb.tgz $(HTTPS)://www.normation.com/tarball/lmdb/lmdb-$(LMDB_RELEASE).tar.gz $(LMDB_SHA1)
	gunzip < $(TMP_DIR)/lmdb.tgz | $(TAR) xf -
	mv ./lmdb-LMDB_$(LMDB_RELEASE) ./lmdb-source
	../../build-caching put ./lmdb-source/ --force-config name=lmdb-src version=$(LMDB_RELEASE)
	rm -rf $(TMP_DIR)
endif

pcre-source:
ifeq ($(shell ../../build-caching get ./pcre-source/ --force-config name=pcre-src version=$(PCRE_RELEASE) >/dev/null 2>&1 || echo KO), KO)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	# Original URL: http://vorboss.dl.sourceforge.net/project/pcre/pcre/8.38/pcre-8.38.tar.gz
	$(GET) $(TMP_DIR)/pcre.tgz $(HTTPS)://www.normation.com/tarball/pcre/pcre-$(PCRE_RELEASE).tar.gz $(PCRE_SHA1)
	gunzip < $(TMP_DIR)/pcre.tgz | $(TAR) xf -
	mv ./pcre-$(PCRE_RELEASE) ./pcre-source
	../../build-caching put ./pcre-source/ --force-config name=pcre-src version=$(PCRE_RELEASE)
	rm -rf $(TMP_DIR)
endif

rudder-sources.tar.bz2:
	$(GET) rudder-sources.tar.bz2 $(HTTPS)://www.rudder-project.org/archives/rudder-sources-$(RUDDER_VERSION_TO_PACKAGE).tar.bz2 ""

rudder-sources: rudder-sources.tar.bz2
	bunzip2 < rudder-sources.tar.bz2 | $(TAR) xf -
	mv rudder-sources-$(RUDDER_VERSION_TO_PACKAGE)*/ rudder-sources/ # the '*' is needed for nightly release
	touch rudder-sources # to make it younger than the bzip source file

rudder.8.gz:
	# Get man page from rudder-project.org
	$(GET) rudder.8 $(HTTPS)://www.rudder-project.org/rudder-doc-${RUDDER_MAJOR_VERSION}/rudder.8 ""
	gzip -f rudder.8

initial-promises: ./rudder-sources
	rm -rf ./initial-promises/
	cp -r ./rudder-sources/rudder-techniques/initial-promises/node-server/ ./initial-promises

initial-ncf: ./rudder-sources ./initial-promises
	rm -rf ./initial-ncf
	cp -r ./rudder-sources/ncf/tree ./initial-ncf
	# retreive the rudder specific ncf.conf
	cp ./initial-promises/distributePolicy/ncf/ncf.conf ./initial-ncf/

rudder-agent-utilities: rudder-sources
	rm -rf ./rudder-agent-utilities/
	cp -r ./rudder-sources/rudder-agent/ ./rudder-agent-utilities
	# Add capabilities
	echo cfengine >> ./rudder-agent-utilities/agent-capabilities
	echo dumpdatastate >> ./rudder-agent-utilities/agent-capabilities
	echo bodydefault >> ./rudder-agent-utilities/agent-capabilities

fusioninventory-agent:
ifeq ($(shell ../../build-caching get ./fusioninventory-agent/ --force-config name=fusioninventory-agent version=$(FUSION_RELEASE) >/dev/null 2>&1 || echo KO), KO)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	#Original URL: https://github.com/fusioninventory/fusioninventory-agent/releases/download/2.3.19/FusionInventory-Agent-$(FUSION_RELEASE).tar.gz
	$(GET) $(TMP_DIR)/fusion.tgz $(HTTPS)://www.normation.com/tarball/fusioninventory/FusionInventory-Agent-$(FUSION_RELEASE).tar.gz $(FUSION_SHA1)
	gunzip < $(TMP_DIR)/fusion.tgz | $(TAR) xf -
	mv ./FusionInventory-Agent-$(FUSION_RELEASE) ./fusioninventory-agent
	../../build-caching put ./fusioninventory-agent/ --force-config name=fusioninventory-agent version=$(FUSION_RELEASE)
	rm -rf $(TMP_DIR)
endif

	# Apply patches
	for PATCHNAME in patches/fusioninventory/*.patch; do echo "Applying $$PATCHNAME... on FusionInventory"; $(PATCH) -d ./fusioninventory-agent -p1 < $$PATCHNAME; done
	# Make sure there were no rejects
	test `$(FIND) ./fusioninventory-agent -name \*.rej | wc -l` = 0

	# Fix a lsusb invocation that crashes some SLES machines
	$(FIND) ./fusioninventory-agent -name "USB.pm" -exec rm "{}" \;

# WARNING: Increment perl_source when changing one of the URLs (perl or cpanminus)
perl_source=002
perl-source: perl-$(PERL_VERSION)
perl-$(PERL_VERSION):
ifeq ($(shell ../../build-caching get ./perl-$(PERL_VERSION)/ --force-config name=perl-source versions=$(perl_source) >/dev/null 2>&1 || echo KO), KO)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	# Original URL: http://www.cpan.org/src/5.0/perl-5.22.0.tar.gz
	$(GET) $(TMP_DIR)/perl.tar.gz $(HTTPS)://www.normation.com/tarball/perl/perl-$(PERL_VERSION).tar.gz $(PERL_SHA1)
	gunzip < $(TMP_DIR)/perl.tar.gz | $(TAR) xf -
	# Original URL: http://www.cpan.org/modules/by-module/App/App-cpanminus-1.7036.tar.gz
	$(GET) ./perl-$(PERL_VERSION)/App-cpanminus.tar.gz $(HTTPS)://www.normation.com/tarball/cpan/App-cpanminus-$(CPANMINUS_RELEASE).tar.gz $(CPANMINUS_SHA1)
	cd perl-$(PERL_VERSION) && gunzip < App-cpanminus.tar.gz | $(TAR) xf -
	mv ./perl-$(PERL_VERSION)/App-cpanminus-$(CPANMINUS_RELEASE) ./perl-$(PERL_VERSION)/App-cpanminus
	# cpanminus+curl have a bug on rhel3+64bits, this is the workaround, see http://www.rudder-project.org/redmine/issues/8533
	sed -i -e "s/'-#',//" ./perl-$(PERL_VERSION)/App-cpanminus/bin/cpanm
	../../build-caching put ./perl-$(PERL_VERSION)/ --force-config name=perl-source versions=$(perl_source)
	rm -rf $(TMP_DIR)
endif

cpan-source: perl-$(PERL_VERSION)/App-cpanminus
perl-$(PERL_VERSION)/App-cpanminus:
	mkdir -p ./perl-$(PERL_VERSION)
	$(eval TMP_DIR := $(shell mktemp -dq /tmp/rudder.XXXXXX))
	$(GET) $(TMP_DIR)/App-cpanminus.tar.gz $(HTTPS)://www.normation.com/tarball/cpan/App-cpanminus-$(CPANMINUS_RELEASE).tar.gz $(CPANMINUS_SHA1)
	cd perl-$(PERL_VERSION) && gunzip < $(TMP_DIR)/App-cpanminus.tar.gz | $(TAR) xf -
	mv ./perl-$(PERL_VERSION)/App-cpanminus-$(CPANMINUS_RELEASE) ./perl-$(PERL_VERSION)/App-cpanminus
	rm -rf $(TMP_DIR)

rudder-agent.cron: rudder-sources
	cp ./rudder-sources/rudder-techniques/techniques/system/common/1.0/rudder-agent-community-cron ./rudder-agent.cron
	# Set unexpanded variables of the cron file
	$(SED) 's@\$${sys.workdir}@/var/rudder/cfengine-community@g' rudder-agent.cron | $(SED) 's@\$${g.rudder_base}@/opt/rudder@g' | $(SED) 's@\\&\\&@\&\&@g' | $(SED) 's@\\&1@\&1@g' > rudder-agent.cron.new
	mv rudder-agent.cron.new rudder-agent.cron

####################################
# Builds separated into components #
####################################

# TODO generalize install in build-* to allow caching
build-ssl: openssl-source
	cd openssl-source && $(SSL_CONFIGURE) -fPIC --prefix=$(RUDDER_DIR) --openssldir=$(RUDDER_DIR)/openssl shared
	cd openssl-source && $(MAKE)
	slibclean >/dev/null 2>&1 || true
	# install to a temporary location is needed to build cfengine
	cd openssl-source && $(MAKE) install INSTALL_PREFIX=$(CURDIR)/build-ssl

build-lmdb: lmdb-source
	cd lmdb-source/libraries/liblmdb && $(MAKE)
	# need a temporary install for cfengine
	mkdir -p build-lmdb/opt/rudder
	# LMDB's Makefile does not know how to create destination files, do it ourselves
	cd build-lmdb/opt/rudder && mkdir -p bin lib include man/man1
	# install to a temporary location is needed to build cfengine
	cd lmdb-source/libraries/liblmdb && $(MAKE) install prefix=$(RUDDER_DIR) DESTDIR=$(CURDIR)/build-lmdb

build-pcre: pcre-source
	cd pcre-source && ./configure --disable-cpp --enable-utf8 --enable-unicode-properties --prefix=$(RUDDER_DIR)
	cd pcre-source && $(MAKE)
	# install to a temporary location is needed to build cfengine
	cd pcre-source && $(MAKE) install DESTDIR=$(CURDIR)/build-pcre

build-cfengine: get-flags $(BUILD_DEPS) cfengine-source build-cfengine-stamp
build-cfengine-stamp:
	# If there is no configure, bootstrap with autogen.sh first
	cd cfengine-source && [ -x ./configure ] || NO_CONFIGURE=1 ./autogen.sh
	# remove temporary rpath added by configure to build options
	cd cfengine-source && $(SED) 's/[A-Z]*_LDFLAGS="$[A-Z]*_LDFLAGS -R$with_[a-z]*\/lib"/true/' configure > configure.new && mv configure.new configure && chmod +x configure
	cd cfengine-source && ./configure --prefix=$(RUDDER_DIR) --with-workdir=/var/rudder/cfengine-community --enable-static=yes --enable-shared=no --without-postgresql $(LMDB_ARG) $(OPENSSL_ARG) $(PCRE_ARG) CFLAGS="$(BUILD_CFLAGS) $(SECURE_CFLAGS)" LDFLAGS="-Wl,-R/opt/rudder/lib $(BUILD_LDFLAGS) $(SECURE_LDFLAGS)"
	cd cfengine-source && $(MAKE)
	touch $@

# Test if compiler supports hardening flags
get-flags:
	$(eval BUILD_CFLAGS:=$(shell [ -x /usr/bin/dpkg-buildflags ] && /usr/bin/dpkg-buildflags --get CFLAGS ))
	$(eval BUILD_LDFLAGS:=$(shell [ -x /usr/bin/dpkg-buildflags ] && /usr/bin/dpkg-buildflags --get LDFLAGS ))
	$(eval HAVE_PROTECTION:=$(shell \
	FLAG_TEST_FILE=`mktemp /tmp/hardening.XXXXXX` ;\
	echo "void main() {}" > "$${FLAG_TEST_FILE}.c" ;\
	gcc $(TRY_CFLAGS) $(TRY_LDFLAGS) -o "$${FLAG_TEST_FILE}" "$${FLAG_TEST_FILE}.c" 2>/dev/null && echo "OK";\
	rm -f "$${FLAG_TEST_FILE}" "$${FLAG_TEST_FILE}".c ))
	
	$(eval SECURE_CFLAGS:=$(if $(HAVE_PROTECTION),$(TRY_CFLAGS)))
	$(eval SECURE_LDFLAGS:=$(if $(HAVE_PROTECTION),$(TRY_LDFLAGS)))

build-perl: perl-$(PERL_VERSION)
	cd perl-$(PERL_VERSION) && $(MAKE) distclean 2>/dev/null || true # Or this can fail on an already used source
	cd perl-$(PERL_VERSION) && ./Configure -Dnoextensions=ODBM_File -Duserelocatableinc -Dusethreads -des -Dcc="gcc" -Dinstallprefix=$(RUDDER_DIR) -Dsiteprefix=$(RUDDER_DIR) -Dprefix=$(RUDDER_DIR)
	cd perl-$(PERL_VERSION) && $(MAKE)
	touch $@

#####################################
# Install separated into components #
#####################################

install-ssl: build-ssl
	mkdir -p $(DESTDIR)
	# TODO cleanup a bit, we don't need everything here
	$(CP_A) $(CURDIR)/build-ssl/* $(DESTDIR)

install-lmdb: build-lmdb
	mkdir -p $(DESTDIR)
	$(CP_A) $(CURDIR)/build-lmdb/* $(DESTDIR)

install-pcre: build-pcre
	mkdir -p $(DESTDIR)
	$(CP_A) $(CURDIR)/build-pcre/* $(DESTDIR)

install-cfengine: build-cfengine $(INSTALL_DEPS)
	cd cfengine-source && $(MAKE) install DESTDIR=$(DESTDIR) STRIP=""
	mkdir -p $(INSTALL_DIR)/share/man/man8
	cd cfengine-source && for binary in cf-agent cf-promises cf-key cf-execd cf-serverd cf-monitord cf-runagent; do \
		LD_LIBRARY_PATH="$(INSTALL_DIR)/lib" $${binary}/$${binary} -M | gzip > $(INSTALL_DIR)/share/man/man8/$${binary}.8.gz; \
	done

install-perl: build-perl
	cd perl-$(PERL_VERSION) && $(MAKE) install DESTDIR=$(DESTDIR)
	touch build-perl # because perl-$(PERL_VERSION) is touched by make install

CPANM:=$(CURDIR)/perl-$(PERL_VERSION)/App-cpanminus/bin/cpanm
# --notest because tests add new Test:: dependencies that cannot be installed properly
MODULE_INSTALL:=MAKEFLAGS= $(PERL) $(CPANM) $(CPAN_OPT) -L $(INSTALL_DIR) --from $(CPAN_URL) --notest
# List from README / Makefile.PL in fusion-inventory-agent
# -> requires + inventory and network recommends
FUSION_MODULES = File::Which LWP::UserAgent Net::IP Text::Template UNIVERSAL::require XML::TreePP Parse::EDID DateTime DateTime::TimeZone DateTime::TimeZone::Local::Unix Net::SNMP Net::NBName Thread::Queue
# update sources at normation.com with the command: ./mirror $VAR_CPAN $FUSION_MODULES 
# update core dependencies (for old perl versions) at normation.com with the command: ./mirror $VAR_CPAN Local::Win32 ExtUtils::MakeMaker parent IO::Compress::Bzip2 HTTP::Lite version CPAN::Meta::YAML CPAN::Meta::Requirements File::Temp Perl::OSType ExtUtils::Install threads::shared Module::Install URI::Escape ExtUtils::CBuilder Pod::Man ExtUtils::ParseXS TAP::Harness threads ExtUtils::Manifest Pod::Simple Module::Build Module::Implementation Test::Harness Module::CoreList File::Spec autodie Devel::PPPort Encode Pod::Escapes Class::C3 IPC::Cmd Digest::MD5 Locale::Maketext::Simple Module::Load Digest::base Params::Check Locale::Maketext::Simple

fusion-modules: $(FUSION_DEPS)
	mkdir -p $(INSTALL_DIR)/lib/perl5
ifeq (true,$(USE_SYSTEM_PERL))
	# install installer dependencies on the system
	MAKEFLAGS= $(PERL) $(CPANM) $(CPAN_OPT) --from $(CPAN_URL) --notest Module::CoreList
endif
	$(MODULE_INSTALL) $(FUSION_MODULES)
	# Replace destdir by prefix in files generated by cpanm
	find $(INSTALL_DIR)/lib/perl5 -type f -name "*.packlist" -o -name "*.pod" -exec sed -i "s,$(DESTDIR),,"  {} \;
	find $(INSTALL_DIR)/bin -type f -exec sed -i "s,$(DESTDIR),,"  {} \;
	touch $@

install-fusion: fusioninventory-agent fusion-modules
# Fusion installation
	mkdir -p $(INSTALL_DIR)/share/fusion-utils
	cp -r fusioninventory-agent/share/* $(INSTALL_DIR)/share/fusion-utils/
ifeq ($(INSTALL),ginstall)
	# fusion doesn't know how to make install without gnu tools in the path
	$(SED) 's/\([^g]\)install -m/\1ginstall -m/' fusioninventory-agent/Makefile.PL > fusioninventory-agent/Makefile.PL.new
	mv fusioninventory-agent/Makefile.PL.new fusioninventory-agent/Makefile.PL
endif
ifeq (true,$(USE_SYSTEM_PERL))
	# install installer dependencies on the system
	MAKEFLAGS= $(PERL) $(CPANM) $(CPAN_OPT) --from $(CPAN_URL) --notest Module::Install
endif
	cd fusioninventory-agent && PERL_MM_USE_DEFAULT=1 $(PERL) -I. Makefile.PL --default PREFIX=$(RUDDER_DIR)
	cd fusioninventory-agent && $(MAKE) install DESTDIR=$(DESTDIR)
	# bug in fusion installer that doesn't set the right owner to some files
	chown root $(DESTDIR)/opt/rudder/share/fusioninventory
	chown root $(DESTDIR)/opt/rudder/share/fusioninventory/html
	# fusion configuration file is never used, do not include it to avoid mistakes
	rm -rf $(DESTDIR)/opt/rudder/etc/fusioninventory
	cp -r fusioninventory-agent/lib/FusionInventory $(INSTALL_DIR)/lib/perl5/

.PHONY: localclean localdepends veryclean depend build install clean mrproper install-ssl install-lmdb install-cfengine install-perl install-fusion get-flags
