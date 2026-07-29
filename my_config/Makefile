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
#   make upload                 # 上传最新 sysupgrade 到 mw（VERSION 读 my_config/VERSION）
#   make upload CHANNEL=stable UPLOAD_TOKEN=...

MK_DIR    := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))
LEDE_DIR  := $(abspath $(MK_DIR)/..)
ETMU_DIR  := $(abspath $(MK_DIR)/../..)

WEB_DIR   := $(LEDE_DIR)/dl/etmu-web
EDEV_DIR  := $(LEDE_DIR)/dl/etmu-edev
DIST_DIR  := $(LEDE_DIR)/bin/etmu-web

CONTAINER  ?= ubuntu_tr3000
BRANCH     ?= tr3000
WEB_BRANCH ?= web_tr3000
EDEV_BRANCH?= edev
WEB_REPO   ?= https://etmu:EnchantedTMU123@gitee.com/etmu/erouter
EDEV_REPO  ?= $(WEB_REPO)
JOBS       ?= $(shell nproc 2>/dev/null || echo 4)

# Release 版本（大.小.调试，调试段百进制）；前端/后端/插件各自另有版本，互不自动同步
VERSION_FILE ?= $(MK_DIR)/VERSION
VERSION      ?= $(shell tr -d ' \t\r\n' < "$(VERSION_FILE)" 2>/dev/null)
# 调试段上限（百进制：0～DBG_MAX，再 +1 则小版本进位）
DBG_MAX      ?= 99
# NOTES 默认在 upload 配方内由 lede/web/edev 最新 commit 生成；可 NOTES=... 覆盖
NOTES        ?=
# make firmware 前是否自动调试段 +1 再注入；BUMP=0 关闭（同版本重编）
BUMP         ?= 1
# 与 mw erouter.upload_token 开发约定一致；.upload_token 或命令行可覆盖
UPLOAD_TOKEN ?= erouter
UPLOAD_URL   ?= http://8.138.182.21:8081/erouter/firmware/
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
	1 2 3 tuna ustc nju socks proxy firmware firmware-bump upload inject-version bump-version make ensure-container

all: update build config download firmware

# 与 download 并列的选择器（无实际动作）
1 2 3 tuna ustc nju socks proxy:
	@:

help:
	@echo "目录: $(MK_DIR)"
	@echo "目标: all | update | update-web | update-edev | update-lede | config | build | rebuild-erouter | download | firmware | upload"
	@echo "说明: make firmware = build + 注入 Release + 强制重编 erouter-web/edev 包 + 出固件"
	@echo "Release: VERSION=$(VERSION)（$(VERSION_FILE)；大.小.调试，百进制调试≤$(DBG_MAX)）"
	@echo "固件: make firmware 默认先 +1 再注入 Release（BUMP=0 跳过递增）"
	@echo "上传: make upload  →  POST $(UPLOAD_URL)；用当前 VERSION（与已编固件一致）"
	@echo "      令牌: 默认 UPLOAD_TOKEN=$(UPLOAD_TOKEN)；可 .upload_token 或命令行覆盖"
	@echo "      变量: BOARD=$(BOARD) CHANNEL=$(CHANNEL) VERSION=$(VERSION) FW=路径 NOTES=覆盖"
	@echo "版本: make bump-version  →  调试段 +1（>$(DBG_MAX) → 小版本+1、调试归零）"
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
	@true
else
ensure-container:
	@docker inspect $(CONTAINER) >/dev/null 2>&1 || { echo "错误: 无容器 $(CONTAINER)"; exit 1; }
	@if [ "$$(docker inspect -f '{{.State.Status}}' $(CONTAINER))" != "running" ]; then \
		echo "启动 $(CONTAINER)..."; docker start $(CONTAINER) >/dev/null; \
	fi
endif

update: update-web update-edev update-lede

update-web:
	@echo "==> 更新前端源码 → $(WEB_DIR) (branch=$(WEB_BRANCH))"
	@mkdir -p $(LEDE_DIR)/dl
	@if [ -d "$(WEB_DIR)/.git" ]; then \
		$(GIT_SAFE) -C $(WEB_DIR) remote set-url origin $(WEB_REPO); \
		$(GIT_SAFE) -C $(WEB_DIR) fetch --depth 1 --prune origin $(WEB_BRANCH); \
		$(GIT_SAFE) -C $(WEB_DIR) checkout -B $(WEB_BRANCH) FETCH_HEAD; \
		$(GIT_SAFE) -C $(WEB_DIR) reset --hard FETCH_HEAD; \
		$(GIT_SAFE) -C $(WEB_DIR) clean -fd -e node_modules -e node_modules/**; \
	else \
		$(GIT_SAFE) clone -b $(WEB_BRANCH) --single-branch --depth 1 $(WEB_REPO) $(WEB_DIR); \
	fi
	@echo "etmu-web: $$($(GIT_SAFE) -C $(WEB_DIR) rev-parse --short HEAD)"

update-edev:
	@echo "==> 更新设备代理源码 → $(EDEV_DIR) (branch=$(EDEV_BRANCH))"
	@mkdir -p $(LEDE_DIR)/dl
	@if [ -d "$(EDEV_DIR)/.git" ]; then \
		$(GIT_SAFE) -C $(EDEV_DIR) remote set-url origin $(EDEV_REPO); \
		$(GIT_SAFE) -C $(EDEV_DIR) fetch --depth 1 --prune origin $(EDEV_BRANCH); \
		$(GIT_SAFE) -C $(EDEV_DIR) checkout -B $(EDEV_BRANCH) FETCH_HEAD; \
		$(GIT_SAFE) -C $(EDEV_DIR) reset --hard FETCH_HEAD; \
		$(GIT_SAFE) -C $(EDEV_DIR) clean -fd; \
	else \
		$(GIT_SAFE) clone -b $(EDEV_BRANCH) --single-branch --depth 1 $(EDEV_REPO) $(EDEV_DIR); \
	fi
	@test -f $(EDEV_DIR)/files/usr/sbin/erouterd || { echo "错误: 缺少 erouterd"; exit 1; }
	@echo "etmu-edev: $$($(GIT_SAFE) -C $(EDEV_DIR) rev-parse --short HEAD)"

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
	/bin/cp -f $(MK_DIR)/.config $(LEDE_DIR)/.config
	@rm -f $(LEDE_DIR)/package/feeds/luci/luci-mod-system
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; make oldconfig </dev/null || yes "" | make oldconfig'

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

# OpenWrt 不感知 dl/etmu-* 的 git 变更；必须 clean 后才会重跑 package Compile/install
# 注入 Release 到 rootfs overlay + 包安装路径共用 my_config/VERSION
inject-version:
	@test -f "$(VERSION_FILE)" || { echo "错误: 缺少 $(VERSION_FILE)"; exit 1; }
	@ver=$$(tr -d ' \t\r\n' < "$(VERSION_FILE)"); \
	echo "$$ver" | grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+$$' || { \
		echo "错误: Release 须为 大.小.调试，当前: $$ver"; exit 1; }; \
	dbg=$$(echo "$$ver" | cut -d. -f3); \
	[ "$$dbg" -le $(DBG_MAX) ] || { echo "错误: 调试段须≤$(DBG_MAX)，当前: $$ver"; exit 1; }; \
	mkdir -p "$(LEDE_DIR)/files/etc/erouter"; \
	printf '%s\n' "$$ver" > "$(LEDE_DIR)/files/etc/erouter/VERSION"; \
	echo "==> Release $$ver → files/etc/erouter/VERSION"

# 调试段 +1；超过 DBG_MAX 则小版本 +1、调试归零（百进制）
bump-version:
	@test -f "$(VERSION_FILE)" || { echo "错误: 缺少 $(VERSION_FILE)"; exit 1; }
	@ver=$$(tr -d ' \t\r\n' < "$(VERSION_FILE)"); \
	echo "$$ver" | grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+$$' || { \
		echo "错误: Release 须为 大.小.调试，当前: $$ver"; exit 1; }; \
	maj=$$(echo "$$ver" | cut -d. -f1); \
	min=$$(echo "$$ver" | cut -d. -f2); \
	dbg=$$(echo "$$ver" | cut -d. -f3); \
	dbg=$$((dbg + 1)); \
	if [ "$$dbg" -gt $(DBG_MAX) ]; then min=$$((min + 1)); dbg=0; fi; \
	printf '%s.%s.%s\n' "$$maj" "$$min" "$$dbg" > "$(VERSION_FILE)"; \
	echo "==> 版本 $$ver → $$(tr -d ' \t\r\n' < "$(VERSION_FILE)")"

rebuild-erouter: ensure-container inject-version
	@echo "==> 强制重编 erouter-web / erouter-edev（避免用缓存旧 ipk）"
	@test -f $(WEB_DIR)/package.json || { echo "错误: 缺少 etmu-web，先 make update-web"; exit 1; }
	@test -f $(EDEV_DIR)/files/usr/sbin/erouterd || { echo "错误: 缺少 etmu-edev，先 make update-edev"; exit 1; }
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
		make package/erouter-web/clean package/erouter-edev/clean; \
		make package/erouter-web/compile package/erouter-edev/compile -j$(JOBS)'

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

firmware-bump:
	@if [ "$(BUMP)" = "0" ]; then \
		echo "==> 跳过版本递增（BUMP=0），Release=$$(tr -d ' \t\r\n' < "$(VERSION_FILE)")"; \
	else \
		$(MAKE) bump-version; \
	fi

firmware: ensure-container firmware-bump build rebuild-erouter
	@echo "==> make V=s -j$(JOBS)（含已强制重编的 erouter-web / erouter-edev）"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; make V=s -j$(JOBS)'

# 不依赖 docker：直接 curl 上传本地 sysupgrade（产物在挂载上）
upload:
	@token="$(UPLOAD_TOKEN)"; \
	if [ "$(origin UPLOAD_TOKEN)" != "command line" ] && [ -f "$(MK_DIR)/.upload_token" ]; then \
		token=$$(tr -d '\r\n' < "$(MK_DIR)/.upload_token"); \
	fi; \
	if [ -z "$$token" ]; then \
		echo "错误: 上传令牌为空"; exit 1; \
	fi; \
	ver=$$(tr -d ' \t\r\n' < "$(VERSION_FILE)" 2>/dev/null || true); \
	if [ -z "$$ver" ]; then \
		echo "错误: 缺少 Release 版本（$(VERSION_FILE) 或 VERSION=）"; exit 1; \
	fi; \
	echo "$$ver" | grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+$$' || { \
		echo "错误: Release 须为 大.小.调试，当前: $$ver"; exit 1; }; \
	dbg=$$(echo "$$ver" | cut -d. -f3); \
	[ "$$dbg" -le $(DBG_MAX) ] || { echo "错误: 调试段须≤$(DBG_MAX)，当前: $$ver"; exit 1; }; \
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
		parts=""; \
		for pair in "lede:$(LEDE_DIR)" "web:$(WEB_DIR)" "edev:$(EDEV_DIR)"; do \
			name=$${pair%%:*}; dir=$${pair#*:}; \
			c=$$($(GIT_SAFE) -C "$$dir" log -1 --format='%h %s' 2>/dev/null || true); \
			[ -n "$$c" ] || continue; \
			if [ -n "$$parts" ]; then parts="$$parts | "; fi; \
			parts="$$parts$$name: $$c"; \
		done; \
		notes="$$parts"; \
	fi; \
	sha=$$(sha256sum "$$bin" | awk '{print $$1}'); \
	echo "==> 上传 $$bin"; \
	echo "    → $(UPLOAD_URL)"; \
	echo "    board=$(BOARD) channel=$(CHANNEL) version=$$ver"; \
	echo "    sha256=$$sha"; \
	echo "    notes=$$notes"; \
	resp=$$(curl -sS -w '\n%{http_code}' -X POST "$(UPLOAD_URL)" \
		-H "Authorization: Bearer $$token" \
		-F "file=@$$bin" \
		-F "board=$(BOARD)" \
		-F "channel=$(CHANNEL)" \
		-F "version=$$ver" \
		-F "sha256=$$sha" \
		-F "notes=$$notes"); \
	code=$$(printf '%s\n' "$$resp" | tail -n1); \
	body=$$(printf '%s\n' "$$resp" | sed '$$d'); \
	printf '%s\n' "$$body"; \
	case "$$code" in \
		201) echo "成功 HTTP $$code";; \
		*) echo "失败 HTTP $$code"; exit 1;; \
	esac

make: firmware
