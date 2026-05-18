#
# 覆盖 feeds 中同名包，附加 patches/ 下补丁（刷机重连地址等）
#
include $(TOPDIR)/rules.mk

LUCI_TITLE:=LuCI Administration - Global System Settings
LUCI_DEPENDS:=+luci-base

PKG_LICENSE:=Apache-2.0

include $(TOPDIR)/feeds/luci/luci.mk

# call BuildPackage - OpenWrt buildroot signature
