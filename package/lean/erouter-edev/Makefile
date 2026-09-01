#
# eRouter 设备代理：编译时从 erouter 仓库 edev 分支拉取 files/ 并安装
#

include $(TOPDIR)/rules.mk

PKG_NAME:=erouter-edev
PKG_VERSION:=0.0.20
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

EDEV_SRC_DIR:=$(TOPDIR)/dl/etmu-edev
EDEV_REPO?=https://etmu:EnchantedTMU123@gitee.com/etmu/erouter
EDEV_BRANCH?=edev

define Package/erouter-edev
  SECTION:=net
  CATEGORY:=Network
  TITLE:=eRouter device agent (HTTP OTA)
  PKGARCH:=all
  DEPENDS:=+jshn +uhttpd +rpcd +uhttpd-mod-ubus +openssl-util
endef

define Package/erouter-edev/description
  Pulls agent scripts from erouter.git ($(EDEV_BRANCH)) at build time.
  HTTP OTA agent + /cgi-bin/erouter-status + erouter-upgrade-action for local Vue UI.
endef

define Package/erouter-edev/conffiles
/etc/config/erouter
endef

define Build/Prepare
	mkdir -p $(TOPDIR)/dl $(PKG_BUILD_DIR)
	if [ ! -f "$(EDEV_SRC_DIR)/files/usr/sbin/erouterd" ]; then \
		echo "Cloning etmu-edev ($(EDEV_BRANCH))..."; \
		rm -rf "$(EDEV_SRC_DIR)"; \
		git clone -b $(EDEV_BRANCH) --single-branch --depth 1 "$(EDEV_REPO)" "$(EDEV_SRC_DIR)"; \
	fi
	@test -f "$(EDEV_SRC_DIR)/files/usr/sbin/erouterd" || { \
		echo "ERROR: missing erouterd in $(EDEV_SRC_DIR)"; exit 1; \
	}
endef

define Build/Compile
endef

define Package/erouter-edev/install
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_DIR) $(1)/etc/erouter
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_DIR) $(1)/usr/sbin
	$(INSTALL_DIR) $(1)/usr/lib/erouter
	$(INSTALL_DIR) $(1)/www/cgi-bin
	$(INSTALL_CONF) $(EDEV_SRC_DIR)/files/etc/config/erouter $(1)/etc/config/erouter
	$(INSTALL_BIN) $(EDEV_SRC_DIR)/files/etc/init.d/erouter $(1)/etc/init.d/erouter
	$(INSTALL_BIN) $(EDEV_SRC_DIR)/files/usr/sbin/erouterd $(1)/usr/sbin/erouterd
	$(INSTALL_DATA) $(EDEV_SRC_DIR)/files/usr/lib/erouter/common.sh $(1)/usr/lib/erouter/common.sh
	$(INSTALL_BIN) $(EDEV_SRC_DIR)/files/www/cgi-bin/erouter-status $(1)/www/cgi-bin/erouter-status
	$(INSTALL_BIN) $(EDEV_SRC_DIR)/files/www/cgi-bin/erouter-upgrade-action $(1)/www/cgi-bin/erouter-upgrade-action
	# 固件组成见 LEDE files/etc/swver（编译时由 my_config inject-swver 生成）
endef

$(eval $(call BuildPackage,erouter-edev))
