# TR3000 一键：更新代码 → 构建前端 → download → 编译固件
# 位置: lede/my_config/Makefile
#
# 布局（与 LEDE 依赖/产物对齐，均在 etmu 挂载下）:
#   etmu/lede/              固件源码
#   etmu/lede/dl/           软件依赖包（make download）
#   etmu/lede/dl/etmu-web/  前端源码（git）
#   etmu/lede/bin/etmu-web/ 前端构建产物（与 bin/targets 同级）
#   etmu/lede/bin/targets/...  固件产物
#
#   cd .../lede/my_config && make
#   make JOBS=4 BRANCH=tr3000

MK_DIR    := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))
LEDE_DIR  := $(abspath $(MK_DIR)/..)
ETMU_DIR  := $(abspath $(MK_DIR)/../..)

# 前端源码：放在 dl/ 下，与其它下载依赖同级
WEB_DIR   := $(LEDE_DIR)/dl/etmu-web
# 前端产物：与固件一样进 bin/
DIST_DIR  := $(LEDE_DIR)/bin/etmu-web

CONTAINER  ?= ubuntu_tr3000
BRANCH     ?= tr3000
WEB_BRANCH ?= web_tr3000
WEB_REPO   ?= https://etmu:EnchantedTMU123@gitee.com/etmu/erouter
JOBS       ?= $(shell nproc 2>/dev/null || echo 4)
MIRROR     ?= https://mirrors.tuna.tsinghua.edu.cn/openwrt/sources

DOCKER_ETMU = docker exec -u etmu \
	-e ALL_PROXY= -e all_proxy= -e http_proxy= -e https_proxy= \
	-e HTTP_PROXY= -e HTTPS_PROXY= \
	-e DOWNLOAD_MIRROR=$(MIRROR) \
	$(CONTAINER)

.PHONY: all help update update-web update-lede config build download firmware make ensure-container

all: update build config download firmware

help:
	@echo "目录: $(MK_DIR)"
	@echo "目标: all | update | update-web | update-lede | config | build | download | firmware"
	@echo "变量: CONTAINER=$(CONTAINER) BRANCH=$(BRANCH) WEB_BRANCH=$(WEB_BRANCH) JOBS=$(JOBS)"
	@echo "      WEB_REPO=$(WEB_REPO)"
	@echo "      MIRROR=$(MIRROR)"
	@echo "路径: WEB源码=$(WEB_DIR)"
	@echo "      WEB产物=$(DIST_DIR)"
	@echo "      LEDE=$(LEDE_DIR)"

ensure-container:
	@docker inspect $(CONTAINER) >/dev/null 2>&1 || { echo "错误: 无容器 $(CONTAINER)"; exit 1; }
	@if [ "$$(docker inspect -f '{{.State.Status}}' $(CONTAINER))" != "running" ]; then \
		echo "启动 $(CONTAINER)..."; docker start $(CONTAINER) >/dev/null; \
	fi

update: update-web update-lede

update-web:
	@echo "==> 更新前端源码 → $(WEB_DIR) (branch=$(WEB_BRANCH))"
	@mkdir -p $(LEDE_DIR)/dl
	@if [ ! -d $(WEB_DIR)/.git ]; then \
		rm -rf $(WEB_DIR); \
		git clone -b $(WEB_BRANCH) --single-branch --depth 1 $(WEB_REPO) $(WEB_DIR); \
	else \
		cd $(WEB_DIR) && \
		git remote set-url origin $(WEB_REPO) && \
		git fetch --depth 1 --prune origin $(WEB_BRANCH) && \
		if [ -n "$$(git status --porcelain)" ]; then \
			echo "错误: etmu-web 有未提交改动"; git status --short; exit 1; \
		fi && \
		git checkout -B $(WEB_BRANCH) origin/$(WEB_BRANCH) && \
		git reset --hard origin/$(WEB_BRANCH); \
	fi
	@echo "etmu-web: $$(git -C $(WEB_DIR) rev-parse --short HEAD)"

update-lede: ensure-container
	@echo "==> 更新 lede ($(BRANCH), depth=1)"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
		git fetch --depth 1 --prune origin $(BRANCH); \
		if [ -n "$$(git status --porcelain)" ]; then \
			echo "错误: lede 有未提交改动"; git status --short; exit 1; \
		fi; \
		git checkout -B $(BRANCH) origin/$(BRANCH); \
		git reset --hard origin/$(BRANCH); \
		echo "lede: $$(git rev-parse --short HEAD)"'

config:
	@echo "==> 应用 my_config/.config"
	@test -f $(MK_DIR)/.config || { echo "错误: 缺少 $(MK_DIR)/.config"; exit 1; }
	/bin/cp -f $(MK_DIR)/.config $(LEDE_DIR)/.config
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; make oldconfig </dev/null || yes "" | make oldconfig'

build: ensure-container
	@echo "==> 容器内构建前端 → $(DIST_DIR)"
	@test -f $(WEB_DIR)/package.json || { echo "错误: 缺少前端源码，先 make update-web"; exit 1; }
	mkdir -p $(DIST_DIR)
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; \
		cd /home/etmu/lede/dl/etmu-web; \
		npm config set allow-remote all >/dev/null 2>&1 || true; \
		npm install; \
		npm run build-only -- --outDir /home/etmu/lede/bin/etmu-web --emptyOutDir'
	@test -f $(DIST_DIR)/index.html || { echo "错误: 构建未生成 index.html"; exit 1; }

download: ensure-container
	@echo "==> make download -j$(JOBS) (无代理, $(MIRROR))"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; \
		printf "%s\n" "$(MIRROR)" > scripts/localmirrors; \
		if [ -f .config ]; then \
			if grep -q "^CONFIG_LOCALMIRROR=" .config; then \
				sed -i "s|^CONFIG_LOCALMIRROR=.*|CONFIG_LOCALMIRROR=\"$(MIRROR)\"|" .config; \
			else \
				echo "CONFIG_LOCALMIRROR=\"$(MIRROR)\"" >> .config; \
			fi; \
		fi; \
		make download -j$(JOBS)'

firmware: ensure-container build
	@echo "==> make V=s -j$(JOBS)（含 erouter-web 前端）"
	$(DOCKER_ETMU) bash -lc 'set -euo pipefail; cd /home/etmu/lede; make V=s -j$(JOBS)'

make: firmware
