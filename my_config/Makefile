# TR3000 一键：更新代码 → 构建前端 → download → 编译固件
# 位置: lede/my_config/Makefile
#
# 布局（与 LEDE 依赖/产物对齐，均在 etmu 挂载下）:
#   etmu/lede/              固件源码
#   etmu/lede/dl/           软件依赖包（make download）
#   etmu/lede/dl/etmu-web/  前端源码（git · WEB_BRANCH）
#   etmu/lede/dl/etmu-edev/ 设备代理源码（git · EDEV_BRANCH）
#   etmu/lede/bin/etmu-web/ 前端构建产物（与 bin/targets 同级）
#   etmu/lede/bin/targets/...  固件产物
#
#   cd .../lede/my_config && make
#   make download              # 默认镜像 1=tuna，无 SOCKS
#   make download 1|2|3        # 预置源：1 tuna / 2 ustc / 3 nju
#   make download tuna|ustc|nju
#   make download socks        # 额外走 SOCKS（官方源兜底）
#   make download MIRROR=URL
#   make JOBS=4 BRANCH=tr3000 WEB_BRANCH=web_tr3000 EDEV_BRANCH=edev
#   make upload CHANNEL=debug   # 必须指定通道：stable | release | debug

MK_DIR    := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))
LEDE_DIR  := $(abspath $(MK_DIR)/..)
ETMU_DIR  := $(abspath $(MK_DIR)/../..)

WEB_DIR   := $(LEDE_DIR)/dl/etmu-web
EDEV_DIR  := $(LEDE_DIR)/dl/etmu-edev
DIST_DIR  := $(LEDE_DIR)/bin/etmu-web
SWVER_OUT ?= $(LEDE_DIR)/files/etc/swver

CONTAINER  ?= ubuntu_tr3000
BRANCH     ?= tr3000
WEB_BRANCH ?= web_tr3000
EDEV_BRANCH?= edev
WEB_REPO   ?= https://etmu:EnchantedTMU123@gitee.com/etmu/erouter
EDEV_REPO  ?= $(WEB_REPO)
JOBS       ?= $(shell nproc 2>/dev/null || echo 4)

# NOTES 默认由 upload 读 swver / 各仓最新 commit；可 NOTES=... 覆盖
NOTES        ?=
# FORCE=1 强制 clean 重编 erouter 包（与源码指纹无关）
FORCE        ?= 0
# 与 mw erouter.upload_token 开发约定一致；.upload_token 或命令行可覆盖
UPLOAD_TOKEN ?= erouter
UPLOAD_URL   ?= https://8.138.182.21/api/erouter/firmware/
BOARD        ?= cudy-tr3000
CHANNEL      ?= stable
FW_PATTERN   ?= $(LEDE_DIR)/bin/targets/mediatek/filogic/*tr3000*sysupgrade.bin
FW           ?=

# npm 镜像（make build 拉前端依赖）
NPM_REGISTRY ?= https://mirrors.cloud.tencent.com/npm/

# 预置 openwrt/sources 镜像（make download 1|2|3 或 tuna|ustc|nju）
MIRROR_1   ?= https://mirrors.tuna.tsinghua.edu.cn/openwrt/sources
MIRROR_2   ?= https://mirrors.ustc.edu.cn/openwrt/sources
MIRROR_3   ?= https://mirror.nju.edu.cn/openwrt/sources
MIRROR     ?= $(MIRROR_1)
ifneq ($(filter 1 tuna,$(MAKECMDGOALS)),)
  MIRROR := $(MIRROR_1)
endif
ifneq ($(filter 2 ustc,$(MAKECMDGOALS)),)
  MIRROR := $(MIRROR_2)
endif
ifneq ($(filter 3 nju,$(MAKECMDGOALS)),)
  MIRROR := $(MIRROR_3)
endif

# SOCKS：make download socks  /  SOCKS=1（与预置序号 1 无关）
SOCKS_URL  ?= socks5://host.docker.internal:1080
USE_SOCKS  := $(if $(filter socks proxy,$(MAKECMDGOALS)),1,$(if $(filter 1 yes y,$(SOCKS)),1,))

# 已在容器内（/.dockerenv）则本地执行；宿主机则 docker exec
IN_DOCKER := $(shell test -f /.dockerenv && echo 1)
ifeq ($(IN_DOCKER),1)
DOCKER_ETMU = env ALL_PROXY= all_proxy= http_proxy= https_proxy= \
	HTTP_PROXY= HTTPS_PROXY= DOWNLOAD_MIRROR=$(MIRROR)
ifeq ($(USE_SOCKS),1)
DOWNLOAD_ETMU = env ALL_PROXY=$(SOCKS_URL) all_proxy=$(SOCKS_URL) \
	http_proxy=$(SOCKS_URL) https_proxy=$(SOCKS_URL) \
	HTTP_PROXY=$(SOCKS_URL) HTTPS_PROXY=$(SOCKS_URL) \
	DOWNLOAD_MIRROR=$(MIRROR)
else
DOWNLOAD_ETMU = $(DOCKER_ETMU)
endif
else
DOCKER_ETMU = docker exec -u etmu \
	-e ALL_PROXY= -e all_proxy= -e http_proxy= -e https_proxy= \
	-e HTTP_PROXY= -e HTTPS_PROXY= \
	-e DOWNLOAD_MIRROR=$(MIRROR) \
	$(CONTAINER)
ifeq ($(USE_SOCKS),1)
DOWNLOAD_ETMU = docker exec -u etmu \
	-e ALL_PROXY=$(SOCKS_URL) -e all_proxy=$(SOCKS_URL) \
	-e http_proxy=$(SOCKS_URL) -e https_proxy=$(SOCKS_URL) \
	-e HTTP_PROXY=$(SOCKS_URL) -e HTTPS_PROXY=$(SOCKS_URL) \
	-e DOWNLOAD_MIRROR=$(MIRROR) \
	$(CONTAINER)
else
DOWNLOAD_ETMU = $(DOCKER_ETMU)
endif
endif

GIT_SAFE = git -c safe.directory='*'

.PHONY: all help update update-web update-edev update-lede config build rebuild-erouter download \
	1 2 3 tuna ustc nju socks proxy firmware erouter-prepare upload inject-swver make ensure-container ensure-etmu-writable

all: update build config download firmware

# 与 download 并列的选择器（无实际动作）
1 2 3 tuna ustc nju socks proxy:
	@:

help:
	@echo "目录: $(MK_DIR)"
	@echo "目标: all | update | update-web | update-edev | update-lede | config | build | rebuild-erouter | download | firmware | upload"
	@echo "说明: make firmware = build + 生成 /etc/swver + 按需重编 erouter + 出固件"
	@echo "版本: 编译时写入 files/etc/swver（lede/web/edev 提交信息）；无 my_config/VERSION"
	@echo "固件: 对比 web/edev 源码指纹 + erouter 包 Makefile；有变更或 FORCE=1 则 clean 重编"
	@echo "上传: make upload CHANNEL=stable|release|debug"
	@echo "      必须命令行指定 CHANNEL（不写则失败，避免默默传到 stable）"
	@echo "      令牌: 默认 UPLOAD_TOKEN=$(UPLOAD_TOKEN)；可 .upload_token 或命令行覆盖"
	@echo "      变量: BOARD=$(BOARD) CHANNEL=$(CHANNEL) FW=路径 NOTES=覆盖"
	@echo "权限: 写 lede 树一律以容器用户 etmu；make ensure-etmu-writable 自检"
	@echo "下载镜像预置:"
	@echo "      make download / make download 1 / tuna   # $(MIRROR_1)"
	@echo "      make download 2 / ustc                   # $(MIRROR_2)"
	@echo "      make download 3 / nju                    # $(MIRROR_3)"
	@echo "      make download MIRROR=URL                 # 自定义"
	@echo "SOCKS: make download socks   或  SOCKS=1  SOCKS_URL=$(SOCKS_URL)"
	@echo "变量: CONTAINER=$(CONTAINER) BRANCH=$(BRANCH) JOBS=$(JOBS) MIRROR=$(MIRROR)"
	@echo "路径: WEB=$(WEB_DIR) EDEV=$(EDEV_DIR) DIST=$(DIST_DIR) LEDE=$(LEDE_DIR)"
	@echo "执行: $(if $(IN_DOCKER),容器内本地,宿主机→docker $(CONTAINER))"
ifeq ($(IN_DOCKER),1)
ensure-container:
	@$(MAKE) --no-print-directory ensure-etmu-writable
else
ensure-container:
	@docker inspect $(CONTAINER) >/dev/null 2>&1 || { echo "错误: 无容器 $(CONTAINER)"; exit 1; }
	@if [ "$$(docker inspect -f '{{.State.Status}}' $(CONTAINER))" != "running" ]; then \
		echo "启动 $(CONTAINER)..."; docker start $(CONTAINER) >/dev/null; \
	fi
	@$(MAKE) --no-print-directory ensure-etmu-writable
endif

# 关键路径须对 etmu 可写（禁止宿主机 root 占有后静默失败）
ensure-etmu-writable:
ifeq ($(IN_DOCKER),1)
	@set -e; \
	for d in /home/etmu/lede/my_config /home/etmu/lede/files /home/etmu/lede/files/etc \
		/home/etmu/lede/files/etc/erouter \
		/home/etmu/lede/dl /home/etmu/lede/bin /home/etmu/lede/tmp \
		/home/etmu/lede/package/lean/erouter-edev /home/etmu/lede/package/lean/erouter-web; do \
		mkdir -p "$$d"; \
		tf="$$d/.etmu_write_test"; \
		if ! ( : > "$$tf" ) 2>/dev/null; then \
			echo "错误: etmu 不可写 $$d（属主/权限被宿主机 root 弄乱，请 chown -R etmu:etmu 对应挂载）"; \
			ls -ld "$$d" "$(dirname "$$d")" 2>/dev/null || true; \
			exit 1; \
		fi; \
		rm -f "$$tf"; \
	done
else
	@$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		for d in /home/etmu/lede/my_config /home/etmu/lede/files /home/etmu/lede/files/etc \
			/home/etmu/lede/files/etc/erouter \
			/home/etmu/lede/dl /home/etmu/lede/bin /home/etmu/lede/tmp \
			/home/etmu/lede/package/lean/erouter-edev /home/etmu/lede/package/lean/erouter-web; do \
			mkdir -p "$$d"; \
			tf="$$d/.etmu_write_test"; \
			if ! ( : > "$$tf" ) 2>/dev/null; then \
				echo "错误: etmu 不可写 $$d（属主/权限被宿主机 root 弄乱，请在宿主机: chown -R 1001:1001 …/ubuntu_tr3000/etmu）"; \
				ls -ld "$$d" 2>/dev/null || true; \
				exit 1; \
			fi; \
			rm -f "$$tf"; \
		done'
endif

update: update-web update-edev update-lede

update-web: ensure-container
	@echo "==> 更新前端源码 → $(WEB_DIR) (branch=$(WEB_BRANCH))"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		WEB_DIR=/home/etmu/lede/dl/etmu-web; \
		mkdir -p /home/etmu/lede/dl; \
		if [ -d "$$WEB_DIR/.git" ]; then \
			git -c safe.directory="*" -C "$$WEB_DIR" remote set-url origin "$(WEB_REPO)"; \
			git -c safe.directory="*" -C "$$WEB_DIR" fetch --depth 1 --prune origin $(WEB_BRANCH); \
			git -c safe.directory="*" -C "$$WEB_DIR" checkout -f -B $(WEB_BRANCH) FETCH_HEAD; \
			git -c safe.directory="*" -C "$$WEB_DIR" reset --hard FETCH_HEAD; \
			git -c safe.directory="*" -C "$$WEB_DIR" clean -fd -e node_modules -e node_modules/**; \
		else \
			git -c safe.directory="*" clone -b $(WEB_BRANCH) --single-branch --depth 1 "$(WEB_REPO)" "$$WEB_DIR"; \
		fi; \
		echo "etmu-web: $$(git -c safe.directory="*" -C "$$WEB_DIR" rev-parse --short HEAD)"'

update-edev: ensure-container
	@echo "==> 更新设备代理源码 → $(EDEV_DIR) (branch=$(EDEV_BRANCH))"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		EDEV_DIR=/home/etmu/lede/dl/etmu-edev; \
		DST_MK=/home/etmu/lede/package/lean/erouter-edev/Makefile; \
		mkdir -p /home/etmu/lede/dl; \
		if [ -d "$$EDEV_DIR/.git" ]; then \
			git -c safe.directory="*" -C "$$EDEV_DIR" remote set-url origin "$(EDEV_REPO)"; \
			git -c safe.directory="*" -C "$$EDEV_DIR" fetch --depth 1 --prune origin $(EDEV_BRANCH); \
			git -c safe.directory="*" -C "$$EDEV_DIR" checkout -f -B $(EDEV_BRANCH) FETCH_HEAD; \
			git -c safe.directory="*" -C "$$EDEV_DIR" reset --hard FETCH_HEAD; \
			git -c safe.directory="*" -C "$$EDEV_DIR" clean -fd; \
		else \
			git -c safe.directory="*" clone -b $(EDEV_BRANCH) --single-branch --depth 1 "$(EDEV_REPO)" "$$EDEV_DIR"; \
		fi; \
		test -f "$$EDEV_DIR/files/usr/sbin/erouterd" || { echo "错误: 缺少 erouterd"; exit 1; }; \
		src_mk="$$EDEV_DIR/Makefile"; \
		if [ -f "$$src_mk" ] && [ -f "$$DST_MK" ]; then \
			ver=$$(sed -n "s/^PKG_VERSION:=//p" "$$src_mk" | head -1); \
			rel=$$(sed -n "s/^PKG_RELEASE:=//p" "$$src_mk" | head -1); \
			[ -n "$$ver" ] && sed -i "s/^PKG_VERSION:=.*/PKG_VERSION:=$$ver/" "$$DST_MK"; \
			[ -n "$$rel" ] && sed -i "s/^PKG_RELEASE:=.*/PKG_RELEASE:=$$rel/" "$$DST_MK"; \
			echo "==> 已同步 package/lean/erouter-edev PKG_VERSION=$${ver:-?} PKG_RELEASE=$${rel:-?}"; \
		fi; \
		echo "etmu-edev: $$(git -c safe.directory="*" -C "$$EDEV_DIR" rev-parse --short HEAD)"'

update-lede: ensure-container
	@echo "==> 更新 lede ($(BRANCH), depth=1)"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
		git fetch --depth 1 --prune origin refs/heads/$(BRANCH):refs/remotes/origin/$(BRANCH); \
		if [ -n "$$(git status --porcelain)" ]; then \
			echo "错误: lede 有未提交改动"; git status --short; exit 1; \
		fi; \
		git checkout -B $(BRANCH) origin/$(BRANCH); \
		git reset --hard origin/$(BRANCH); \
		echo "lede: $$(git rev-parse --short HEAD)"'

config: ensure-container
	@echo "==> 应用 my_config/.config"
	@test -f $(MK_DIR)/.config || { echo "错误: 缺少 $(MK_DIR)/.config"; exit 1; }
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		cp -f /home/etmu/lede/my_config/.config /home/etmu/lede/.config; \
		rm -f /home/etmu/lede/package/feeds/luci/luci-mod-system; \
		cd /home/etmu/lede; make oldconfig </dev/null || yes "" | make oldconfig'

build: ensure-container
	@echo "==> 容器内构建前端 → $(DIST_DIR)"
	@test -f $(WEB_DIR)/package.json || { echo "错误: 缺少前端源码，先 make update-web"; exit 1; }
	mkdir -p $(DIST_DIR)
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		cd /home/etmu/lede/dl/etmu-web; \
		npm config set allow-remote all >/dev/null 2>&1 || true; \
		pkg_hash=$$(sha256sum package.json | awk "{print \$$1}"); \
		if [ ! -d node_modules/vite ] || [ ! -f node_modules/.erouter-pkg-hash ] \
			|| [ "$$(cat node_modules/.erouter-pkg-hash)" != "$$pkg_hash" ]; then \
			echo "==> npm install（package.json 变更或缺依赖）"; \
			npm install --registry=$(NPM_REGISTRY) --no-audit --no-fund --prefer-offline; \
			printf "%s\n" "$$pkg_hash" > node_modules/.erouter-pkg-hash; \
		else \
			echo "==> 跳过 npm install（node_modules 与 package.json 一致）"; \
		fi; \
		npm run build-only -- --outDir /home/etmu/lede/bin/etmu-web --emptyOutDir'
	@test -f $(DIST_DIR)/index.html || { echo "错误: 构建未生成 index.html"; exit 1; }
	@! grep -q 'You did it' $(DIST_DIR)/assets/*.js 2>/dev/null || { \
		echo "错误: 产物仍是 Vite 欢迎页，检查 $(WEB_DIR) 是否已 update-web"; exit 1; }

# OpenWrt 不感知 dl/etmu-* 的 git 变更；内容变更时 clean 重编 erouter 包
# 指纹：web/edev 的 HEAD（工作区脏则附加 diff 哈希）+ erouter 包 Makefile；不含 dist
STAMP_FILE   ?= $(MK_DIR)/.erouter-stamp
EROUTER_STATE?= $(MK_DIR)/.erouter-state

# 编译时生成 files/etc/swver（设备路径 /etc/swver）
inject-swver: ensure-container
	@$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		LEDE_DIR=/home/etmu/lede; \
		WEB_DIR=/home/etmu/lede/dl/etmu-web; \
		EDEV_DIR=/home/etmu/lede/dl/etmu-edev; \
		OUT=/home/etmu/lede/files/etc/swver; \
		BOARD="$(BOARD)"; CHANNEL="$(CHANNEL)"; BUILDER="$(CONTAINER)"; \
		GIT="git -c safe.directory=*"; \
		test -f "$$WEB_DIR/package.json" || { echo "错误: 缺少 etmu-web，先 make update-web"; exit 1; }; \
		test -f "$$EDEV_DIR/files/usr/sbin/erouterd" || { echo "错误: 缺少 etmu-edev，先 make update-edev"; exit 1; }; \
		lede_branch=$$($$GIT -C "$$LEDE_DIR" rev-parse --abbrev-ref HEAD 2>/dev/null || echo unknown); \
		lede_commit=$$($$GIT -C "$$LEDE_DIR" rev-parse --short HEAD); \
		lede_subject=$$($$GIT -C "$$LEDE_DIR" log -1 --format=%s | tr -d "\r\n" | sed "s/[=#]//g"); \
		lede_committed_at=$$($$GIT -C "$$LEDE_DIR" log -1 --format=%ci); \
		lede_dirty=0; \
		[ -n "$$($$GIT -C "$$LEDE_DIR" status --porcelain 2>/dev/null | grep -v node_modules || true)" ] && lede_dirty=1; \
		web_branch=$$($$GIT -C "$$WEB_DIR" rev-parse --abbrev-ref HEAD 2>/dev/null || echo unknown); \
		web_commit=$$($$GIT -C "$$WEB_DIR" rev-parse --short HEAD); \
		web_subject=$$($$GIT -C "$$WEB_DIR" log -1 --format=%s | tr -d "\r\n" | sed "s/[=#]//g"); \
		web_committed_at=$$($$GIT -C "$$WEB_DIR" log -1 --format=%ci); \
		web_dirty=0; \
		[ -n "$$($$GIT -C "$$WEB_DIR" status --porcelain 2>/dev/null | grep -v node_modules || true)" ] && web_dirty=1; \
		edev_branch=$$($$GIT -C "$$EDEV_DIR" rev-parse --abbrev-ref HEAD 2>/dev/null || echo unknown); \
		edev_commit=$$($$GIT -C "$$EDEV_DIR" rev-parse --short HEAD); \
		edev_subject=$$($$GIT -C "$$EDEV_DIR" log -1 --format=%s | tr -d "\r\n" | sed "s/[=#]//g"); \
		edev_committed_at=$$($$GIT -C "$$EDEV_DIR" log -1 --format=%ci); \
		edev_dirty=0; \
		[ -n "$$($$GIT -C "$$EDEV_DIR" status --porcelain 2>/dev/null | grep -v node_modules || true)" ] && edev_dirty=1; \
		edev_pkg=$$(sed -n "s/^PKG_VERSION:=//p" "$$LEDE_DIR/package/lean/erouter-edev/Makefile" 2>/dev/null | head -1); \
		edev_rel=$$(sed -n "s/^PKG_RELEASE:=//p" "$$LEDE_DIR/package/lean/erouter-edev/Makefile" 2>/dev/null | head -1); \
		web_pkg=$$(sed -n "s/^PKG_VERSION:=//p" "$$LEDE_DIR/package/lean/erouter-web/Makefile" 2>/dev/null | head -1); \
		web_rel=$$(sed -n "s/^PKG_RELEASE:=//p" "$$LEDE_DIR/package/lean/erouter-web/Makefile" 2>/dev/null | head -1); \
		rtty_pkg=$$(sed -n "s/^PKG_VERSION:=//p" "$$LEDE_DIR/feeds/packages/utils/rtty/Makefile" 2>/dev/null | head -1); \
		rtty_rel=$$(sed -n "s/^PKG_RELEASE:=//p" "$$LEDE_DIR/feeds/packages/utils/rtty/Makefile" 2>/dev/null | head -1); \
		xfrpc_pkg=$$(sed -n "s/^PKG_VERSION:=//p" "$$LEDE_DIR/feeds/packages/net/xfrpc/Makefile" 2>/dev/null | head -1); \
		xfrpc_rel=$$(sed -n "s/^PKG_RELEASE:=//p" "$$LEDE_DIR/feeds/packages/net/xfrpc/Makefile" 2>/dev/null | head -1); \
		id="$$lede_commit-$$web_commit-$$edev_commit"; \
		built_at=$$(TZ=Asia/Shanghai date "+%Y-%m-%d %H:%M:%S %z"); \
		mkdir -p "$$(dirname "$$OUT")"; \
		{ \
			echo "# /etc/swver — 固件组成快照（编译时生成）"; \
			echo "# 时间：Asia/Shanghai"; \
			echo; \
			echo "[meta]"; \
			echo "board=$$BOARD"; \
			echo "channel=$$CHANNEL"; \
			echo "built_at=$$built_at"; \
			echo "builder=$$BUILDER"; \
			echo "id=$$id"; \
			echo; \
			echo "[lede]"; \
			echo "branch=$$lede_branch"; \
			echo "commit=$$lede_commit"; \
			echo "subject=$$lede_subject"; \
			echo "committed_at=$$lede_committed_at"; \
			echo "dirty=$$lede_dirty"; \
			echo; \
			echo "[web]"; \
			echo "branch=$$web_branch"; \
			echo "commit=$$web_commit"; \
			echo "subject=$$web_subject"; \
			echo "committed_at=$$web_committed_at"; \
			echo "dirty=$$web_dirty"; \
			echo; \
			echo "[edev]"; \
			echo "branch=$$edev_branch"; \
			echo "commit=$$edev_commit"; \
			echo "subject=$$edev_subject"; \
			echo "committed_at=$$edev_committed_at"; \
			[ -n "$$edev_pkg" ] && echo "pkg_version=$$edev_pkg"; \
			echo "dirty=$$edev_dirty"; \
			echo; \
			echo "[packages]"; \
			[ -n "$$rtty_pkg" ] && echo "rtty-openssl=$$rtty_pkg-$${rtty_rel:-1}"; \
			[ -n "$$xfrpc_pkg" ] && echo "xfrpc=$$xfrpc_pkg-$${xfrpc_rel:-1}"; \
			[ -n "$$edev_pkg" ] && echo "erouter-edev=$$edev_pkg-$${edev_rel:-1}"; \
			[ -n "$$web_pkg" ] && echo "erouter-web=$$web_pkg-$${web_rel:-1}"; \
		} > "$$OUT"; \
		rm -f "$$LEDE_DIR/files/etc/erouter/VERSION"; \
		echo "==> swver id=$$id → $$OUT"'

# 对比源码指纹，决定是否强制重编 erouter 包（不依赖 dist；不再递增 Release）
erouter-prepare: ensure-container
	@$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		WEB_DIR=/home/etmu/lede/dl/etmu-web; \
		EDEV_DIR=/home/etmu/lede/dl/etmu-edev; \
		LEDE_DIR=/home/etmu/lede; \
		STAMP_FILE=/home/etmu/lede/my_config/.erouter-stamp; \
		EROUTER_STATE=/home/etmu/lede/my_config/.erouter-state; \
		FORCE="$(FORCE)"; \
		test -f "$$WEB_DIR/package.json" || { echo "错误: 缺少 etmu-web，先 make update-web"; exit 1; }; \
		test -f "$$EDEV_DIR/files/usr/sbin/erouterd" || { echo "错误: 缺少 etmu-edev，先 make update-edev"; exit 1; }; \
		web_h=$$(git -c safe.directory="*" -C "$$WEB_DIR" rev-parse HEAD 2>/dev/null || echo none); \
		web_d=$$(git -c safe.directory="*" -C "$$WEB_DIR" status --porcelain 2>/dev/null | grep -v node_modules || true); \
		if [ -n "$$web_d" ]; then \
			web_dh=$$( (git -c safe.directory="*" -C "$$WEB_DIR" diff HEAD 2>/dev/null; printf "%s\n" "$$web_d") | sha256sum | awk "{print \$$1}"); \
			web="$$web_h:dirty:$$web_dh"; \
		else web="$$web_h"; fi; \
		edev_h=$$(git -c safe.directory="*" -C "$$EDEV_DIR" rev-parse HEAD 2>/dev/null || echo none); \
		edev_d=$$(git -c safe.directory="*" -C "$$EDEV_DIR" status --porcelain 2>/dev/null | grep -v node_modules || true); \
		if [ -n "$$edev_d" ]; then \
			edev_dh=$$( (git -c safe.directory="*" -C "$$EDEV_DIR" diff HEAD 2>/dev/null; printf "%s\n" "$$edev_d") | sha256sum | awk "{print \$$1}"); \
			edev="$$edev_h:dirty:$$edev_dh"; \
		else edev="$$edev_h"; fi; \
		pkg=$$(cat "$$LEDE_DIR/package/lean/erouter-web/Makefile" \
			"$$LEDE_DIR/package/lean/erouter-edev/Makefile" 2>/dev/null | sha256sum | awk "{print \$$1}"); \
		[ -n "$$pkg" ] || pkg=none; \
		fp="web=$$web|edev=$$edev|pkg=$$pkg"; \
		old=$$(cat "$$STAMP_FILE" 2>/dev/null || true); \
		need_rebuild=0; \
		if [ "$$FORCE" = "1" ]; then \
			need_rebuild=1; \
			echo "==> FORCE=1：强制重编 erouter"; \
		elif [ "$$fp" != "$$old" ]; then \
			need_rebuild=1; \
			echo "==> 源码指纹有变更：将重编 erouter 包"; \
			echo "    旧: $${old:-<none>}"; \
			echo "    新: $$fp"; \
		else \
			echo "==> 源码指纹未变：跳过强制重编"; \
		fi; \
		printf "FP=%s\nNEED_REBUILD=%s\n" "$$fp" "$$need_rebuild" > "$$EROUTER_STATE"'

rebuild-erouter: ensure-container inject-swver
	@need=$$(sed -n 's/^NEED_REBUILD=//p' "$(EROUTER_STATE)" 2>/dev/null || echo 1); \
	fp=$$(sed -n 's/^FP=//p' "$(EROUTER_STATE)" 2>/dev/null || true); \
	if [ "$$need" = "1" ]; then \
		echo "==> 强制重编 erouter-web / erouter-edev"; \
		$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
			make package/erouter-web/clean package/erouter-edev/clean; \
			make package/erouter-web/compile package/erouter-edev/compile -j$(JOBS)'; \
	else \
		echo "==> 跳过 erouter 包 clean（输入未变）"; \
		$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
			make package/erouter-web/compile package/erouter-edev/compile -j$(JOBS)'; \
	fi; \
	if [ -n "$$fp" ]; then \
		$(DOCKER_ETMU) bash -lc 'printf "%s\n" "'"$$fp"'" > /home/etmu/lede/my_config/.erouter-stamp'; \
	fi

firmware: ensure-container build erouter-prepare rebuild-erouter
	@echo "==> make V=s -j$(JOBS)"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; make V=s -j$(JOBS)'

download: ensure-container
	@echo "==> make download -j$(JOBS) ($(if $(USE_SOCKS),SOCKS $(SOCKS_URL),无SOCKS), 镜像 $(MIRROR))"
	$(DOWNLOAD_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
		rm -f scripts/localmirrors; \
		printf "%s\n" "$(MIRROR)" > scripts/localmirrors; \
		if [ -f .config ]; then \
			if grep -q "^CONFIG_LOCALMIRROR=" .config; then \
				sed -i "s|^CONFIG_LOCALMIRROR=.*|CONFIG_LOCALMIRROR=\"$(MIRROR)\"|" .config; \
			else \
				echo "CONFIG_LOCALMIRROR=\"$(MIRROR)\"" >> .config; \
			fi; \
		fi; \
		make download -j$(JOBS)'

# 不依赖 docker：直接 curl 上传本地 sysupgrade（产物在挂载上）
# CHANNEL 必须在命令行给出，避免默认 stable 误传到正式通道
upload:
	@if [ "$(origin CHANNEL)" != "command line" ]; then \
		echo "错误: 须指定 CHANNEL=stable|release|debug（例如 make upload CHANNEL=debug）"; exit 1; \
	fi; \
	case "$(CHANNEL)" in \
		stable|release|debug) ;; \
		*) echo "错误: CHANNEL 须为 stable、release 或 debug，当前=$(CHANNEL)"; exit 1;; \
	esac; \
	token="$(UPLOAD_TOKEN)"; \
	if [ "$(origin UPLOAD_TOKEN)" != "command line" ] && [ -f "$(MK_DIR)/.upload_token" ]; then \
		token=$$(tr -d '\r\n' < "$(MK_DIR)/.upload_token"); \
	fi; \
	if [ -z "$$token" ]; then \
		echo "错误: 上传令牌为空"; exit 1; \
	fi; \
	swver="$(SWVER_OUT)"; \
	if [ ! -f "$$swver" ]; then \
		echo "错误: 缺少 $$swver，先 make inject-swver 或 make firmware"; exit 1; \
	fi; \
	ver=$$(awk -F= '/^\[meta\]/{m=1;next} /^\[/{m=0} m&&$$1=="id"{print $$2;exit}' "$$swver" | tr -d ' \t\r\n'); \
	if [ -z "$$ver" ]; then \
		echo "错误: $$swver 无 meta.id"; exit 1; \
	fi; \
	if [ -n "$(FW)" ]; then \
		bin="$(FW)"; \
	else \
		bin=$$(ls -1t $(FW_PATTERN) 2>/dev/null | head -1 || true); \
	fi; \
	if [ -z "$$bin" ] || [ ! -f "$$bin" ]; then \
		echo "错误: 无固件，先 make firmware（或 FW=/path/to.bin）"; exit 1; \
	fi; \
	notes="$(NOTES)"; \
	if [ -z "$$notes" ]; then \
		notes=$$(awk -F= '/^\[lede\]/{s="lede"} /^\[web\]/{s="web"} /^\[edev\]/{s="edev"} s&&$$1=="commit"{c[s]=$$2} s&&$$1=="subject"{sub[s]=$$2} END{ \
			o=""; for (n in c){ if(o!="") o=o" | "; o=o n": "c[n]" "sub[n]; } print o; \
		}' "$$swver"); \
	fi; \
	sha=$$(sha256sum "$$bin" | awk '{print $$1}'); \
	echo "==> 上传 $$bin"; \
	echo "    → $(UPLOAD_URL)"; \
	echo "    board=$(BOARD) channel=$(CHANNEL) version=$$ver"; \
	echo "    sha256=$$sha"; \
	echo "    notes=$$notes"; \
	bodyf=$$(mktemp); \
	code=$$(curl -S --progress-bar -o "$$bodyf" -w '%{http_code}' -X POST "$(UPLOAD_URL)" \
		-H "Authorization: Bearer $$token" \
		-F "file=@$$bin" \
		-F "board=$(BOARD)" \
		-F "channel=$(CHANNEL)" \
		-F "version=$$ver" \
		-F "sha256=$$sha" \
		-F "notes=$$notes"); \
	echo; \
	cat "$$bodyf"; echo; \
	rm -f "$$bodyf"; \
	case "$$code" in \
		201) echo "成功 HTTP $$code";; \
		*) echo "失败 HTTP $$code"; exit 1;; \
	esac

make: firmware
