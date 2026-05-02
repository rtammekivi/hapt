include $(TOPDIR)/rules.mk

PKG_NAME       := hapt
# x-release-please-start-version
PKG_VERSION    := 0.7.0
# x-release-please-end
PKG_LICENSE    := GPL-3.0-or-later
PKG_MAINTAINER := Oxan van Leeuwen <oxan@oxanvanleeuwen.nl>

include $(INCLUDE_DIR)/package.mk

define Package/hapt
	SECTION  := net
	CATEGORY := Network
	TITLE    := Wireless device presence tracker integrated with Home Assistant
	# Dependency is on micropython-lib-unix-src as it's a lot smaller than micropython-lib-unix, but both work.
	# micropython-mbedtls (vs -nossl) is required to reach Home Assistant over HTTPS.
	DEPENDS  := +micropython-mbedtls +micropython-lib +micropython-lib-unix-src
	PKGARCH  := all
endef

define Package/hapt/description
	Home Assistant Presence Tracker (hapt) tracks the presence of wireless devices on your network and synchronizes with Home Assistant.
endef

define Package/hapt/conffiles
/etc/config/hapt
endef

define Build/Prepare
endef

define Build/Configure
endef

define Build/Compile
endef

define Package/hapt/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) ./files/hapt.py $(1)/usr/bin/hapt

	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/hapt.init $(1)/etc/init.d/hapt

	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/hapt.config $(1)/etc/config/hapt
endef

define Package/hapt/postinst
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
/etc/init.d/hapt enable
[ -f /var/run/hapt.pid ] && /etc/init.d/hapt restart
exit 0
endef

define Package/hapt/prerm
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
/etc/init.d/hapt disable
/etc/init.d/hapt stop
exit 0
endef

$(eval $(call BuildPackage,hapt))
