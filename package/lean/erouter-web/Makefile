#
# eRouter custom web UI package
#

include $(TOPDIR)/rules.mk

PKG_NAME:=erouter-web
PKG_VERSION:=1.0.0
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

WEB_DIST_DIR:=$(TOPDIR)/../0_web/dist

define Package/erouter-web
  SECTION:=net
  CATEGORY:=Network
  TITLE:=eRouter Web UI
  PKGARCH:=all
  DEPENDS:=+uhttpd
endef

define Package/erouter-web/description
	Install Vue web UI from 0_web/dist.
	Vue home page replaces /www/index.html on first boot.
	LuCI entry at /luci/ (redirects to /cgi-bin/luci/) on the same HTTP port 80.
endef

define Build/Prepare
	@if [ ! -f "$(WEB_DIST_DIR)/index.html" ]; then \
		echo "ERROR: $(WEB_DIST_DIR)/index.html not found."; \
		echo "Run 'npm run build' in 0_web first."; \
		exit 1; \
	fi
endef

define Build/Compile
endef

define Package/erouter-web/install
	$(INSTALL_DIR) $(1)/www/erouter
	$(INSTALL_DIR) $(1)/www/assets
	$(INSTALL_DIR) $(1)/etc/uci-defaults
	$(CP) $(WEB_DIST_DIR)/index.html $(1)/www/erouter/
	$(CP) $(WEB_DIST_DIR)/assets/* $(1)/www/assets/
	[ -f "$(WEB_DIST_DIR)/favicon.ico" ] && $(CP) $(WEB_DIST_DIR)/favicon.ico $(1)/www/erouter/ || true
	$(INSTALL_BIN) ./files/99-erouter-web $(1)/etc/uci-defaults/99-erouter-web
endef

$(eval $(call BuildPackage,erouter-web))
