#
# eRouter custom web UI：编译时拉取/构建 etmu-web，产物进 bin/etmu-web
#

include $(TOPDIR)/rules.mk

PKG_NAME:=erouter-web
PKG_VERSION:=1.0.0
PKG_RELEASE:=3

include $(INCLUDE_DIR)/package.mk

WEB_SRC_DIR:=$(TOPDIR)/dl/etmu-web
WEB_DIST_DIR:=$(TOPDIR)/bin/etmu-web
WEB_REPO?=https://etmu:EnchantedTMU123@gitee.com/etmu/erouter
WEB_BRANCH?=web_mt1300

define Package/erouter-web
  SECTION:=net
  CATEGORY:=Network
  TITLE:=eRouter Web UI
  PKGARCH:=all
  DEPENDS:=+uhttpd
endef

define Package/erouter-web/description
	Build Vue web UI from dl/etmu-web into bin/etmu-web.
	Files live under /www/erouter + /www/assets; postinst overlays /www/index.html
	(so we do not clash with luci-base's owned index.html in the ipk file list).
	LuCI entry at /luci/ on the same HTTP port 80.
endef

define Build/Prepare
	mkdir -p $(TOPDIR)/dl $(PKG_BUILD_DIR) $(WEB_DIST_DIR)
	if [ ! -f "$(WEB_SRC_DIR)/package.json" ]; then \
		echo "Cloning etmu-web ($(WEB_BRANCH))..."; \
		rm -rf "$(WEB_SRC_DIR)"; \
		git clone -b $(WEB_BRANCH) --single-branch --depth 1 "$(WEB_REPO)" "$(WEB_SRC_DIR)"; \
	fi
endef

define Build/Compile
	@command -v npm >/dev/null || { echo "ERROR: npm not found in build environment"; exit 1; }
	cd "$(WEB_SRC_DIR)" && npm config set allow-remote all >/dev/null 2>&1 || true
	cd "$(WEB_SRC_DIR)" && npm install
	# run-p 的 build 会吞掉 --outDir；固件打包只用 vite 产物
	cd "$(WEB_SRC_DIR)" && npm run build-only -- --outDir "$(WEB_DIST_DIR)" --emptyOutDir
	@test -f "$(WEB_DIST_DIR)/index.html" || { echo "ERROR: web build missing index.html"; exit 1; }
	@! grep -q 'You did it' "$(WEB_DIST_DIR)/assets/"*.js 2>/dev/null || { \
		echo "ERROR: web dist still contains Vite welcome page; check $(WEB_SRC_DIR) branch/commit"; exit 1; }
endef

define Package/erouter-web/install
	$(INSTALL_DIR) $(1)/www/assets
	$(INSTALL_DIR) $(1)/www/erouter
	$(INSTALL_DIR) $(1)/etc/uci-defaults
	# 不把 index.html 写进 ipk 的 /www/（会与 luci-base 冲突）；由 postinst 覆盖
	$(CP) $(WEB_DIST_DIR)/index.html $(1)/www/erouter/index.html
	$(CP) $(WEB_DIST_DIR)/assets/* $(1)/www/assets/
	[ -f "$(WEB_DIST_DIR)/favicon.ico" ] && $(CP) $(WEB_DIST_DIR)/favicon.ico $(1)/www/erouter/favicon.ico || true
	$(INSTALL_BIN) ./files/99-erouter-web $(1)/etc/uci-defaults/99-erouter-web
endef

# 出镜像（IPKG_INSTROOT 有值）与真机升级都会执行，保证 /www/index.html 用最新前端
define Package/erouter-web/postinst
#!/bin/sh
ROOT="$${IPKG_INSTROOT}"
if [ -f "$${ROOT}/www/erouter/index.html" ]; then
	cp -f "$${ROOT}/www/erouter/index.html" "$${ROOT}/www/index.html"
fi
if [ -f "$${ROOT}/www/erouter/favicon.ico" ]; then
	cp -f "$${ROOT}/www/erouter/favicon.ico" "$${ROOT}/www/favicon.ico"
fi
exit 0
endef

$(eval $(call BuildPackage,erouter-web))
