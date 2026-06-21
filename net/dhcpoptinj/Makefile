#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=dhcpoptinj
PKG_RELEASE:=3

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/nanosonde/dhcpoptinj.git
PKG_SOURCE_DATE:=2025-04-04
PKG_SOURCE_VERSION:=dev
PKG_MIRROR_HASH:=skip

# PKG_MAINTAINER:=
PKG_LICENSE:=GPL-2.0-or-later

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

define Package/dhcpoptinj
  SECTION:=net
  CATEGORY:=Network
  DEPENDS:=+libnetfilter-queue +nftables +kmod-nft-queue
  TITLE:=Inject DHCP options into DHCP requests
  URL:=https://github.com/misje/dhcpoptinj
endef

define Package/dhcpoptinj/description
 Inject DHCP options into DHCP requests
endef

define Package/dhcpoptinj/conffiles
/etc/config/dhcpoptinj
endef

define Package/dhcpoptinj/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/dhcpoptinj $(1)/usr/bin/
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) files/$(PKG_NAME).config $(1)/etc/config/$(PKG_NAME)
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) files/$(PKG_NAME).init $(1)/etc/init.d/$(PKG_NAME)
endef

$(eval $(call BuildPackage,dhcpoptinj))
