# Makefile
# Usage:
#  This makefile has two targets: build-base and build-community
#  Use environment VERSION and REV to specify TEX version and OCI Image tag revision number, e.g.:
#    VERSION=2022 REV=3 make
#  VERSION defaults to latest and REV defaults to 1
#  All variables below assigned with ?= can be safely overwritten by env

MONOREPO_ROOT := ../
HERE=$(shell pwd)

REV ?= 1
VERSION ?= latest

ifeq ($(VERSION),latest)
MIRROR ?= https://mirror.ox.ac.uk/sites/ctan.org/systems/texlive/tlnet
else
MIRROR ?= https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/$(VERSION)/tlnet-final
endif

export MONOREPO_REVISION := $(shell git rev-parse HEAD)
export BRANCH_NAME ?= $(shell git rev-parse --abbrev-ref HEAD)
OVERLEAF_BASE ?= ghcr.io/lcpu-club/sharelatex-base
OVERLEAF ?= ghcr.io/lcpu-club/sharelatex
export OVERLEAF_BASE_BRANCH ?= $(OVERLEAF_BASE):$(BRANCH_NAME)
export OVERLEAF_BASE_LATEST ?= $(OVERLEAF_BASE)
export OVERLEAF_BASE_TAG ?= $(OVERLEAF_BASE):$(BRANCH_NAME)-$(MONOREPO_REVISION)
export OVERLEAF_BRANCH ?= $(OVERLEAF):$(BRANCH_NAME)
export OVERLEAF_LATEST ?= $(OVERLEAF)
export OVERLEAF_TAG ?= $(OVERLEAF):$(BRANCH_NAME)-$(MONOREPO_REVISION)

ifeq ($(VERSION),latest)
YEAR ?= $(shell date '+%Y')
IS_LATEST ?= docker tag $(OVERLEAF_BASE):$(YEAR).$(REV) $(OVERLEAF_BASE):latest # && docker push $(OVERLEAF_BASE):latest
else
YEAR?=$(VERSION)
IS_LATEST ?= @true
endif

all: build-base build-community

clean:
	-docker rmi --force $(OVERLEAF_BASE_TAG) $(OVERLEAF_TAG)

refresh-cache: refresh-cache-branch
refresh-cache-branch:
	docker inspect $(OVERLEAF_BASE_BRANCH) > /dev/null && docker pull $(OVERLEAF_BASE_BRANCH) || true
	docker inspect $(OVERLEAF_BRANCH) > /dev/null && docker pull $(OVERLEAF_BRANCH) || true
refresh-cache: refresh-cache-latest
refresh-cache-latest:
	docker inspect $(OVERLEAF_BASE_LATEST) > /dev/null && docker pull $(OVERLEAF_BASE_LATEST) || true
	docker inspect $(OVERLEAF_LATEST) > /dev/null && docker pull $(OVERLEAF_LATEST) || true

build-base:
	cp .dockerignore $(MONOREPO_ROOT)
	docker build \
	  --build-arg BUILDKIT_INLINE_CACHE=1 \
	  --progress=plain \
	  --file Dockerfile-base \
	  --pull \
	  --cache-from $(OVERLEAF_BASE_LATEST) \
	  --cache-from $(OVERLEAF_BASE_BRANCH) \
	  --tag $(OVERLEAF_BASE_TAG) \
	  --tag $(OVERLEAF_BASE_BRANCH) \
	  --build-arg TEXLIVE_MIRROR=$(MIRROR) \
	  --build-arg TEXLIVE_VERSION=$(YEAR) \
	  $(MONOREPO_ROOT)
	$(IS_LATEST)

build-community:
	cp .dockerignore $(MONOREPO_ROOT)
	docker build \
	  --build-arg BUILDKIT_INLINE_CACHE=1 \
	  --progress=plain \
	  --build-arg OVERLEAF_BASE_TAG \
	  --label "com.overleaf.ce.revision=$(MONOREPO_REVISION)" \
	  --cache-from $(OVERLEAF_LATEST) \
	  --cache-from $(OVERLEAF_BRANCH) \
	  --file Dockerfile \
	  --tag $(OVERLEAF_TAG) \
	  --tag $(OVERLEAF_BRANCH) \
	  $(MONOREPO_ROOT)

push:
	docker push $(OVERLEAF_BASE_TAG)
	docker push $(OVERLEAF_BASE_BRANCH)
	docker push $(OVERLEAF_TAG)
	docker push $(OVERLEAF_BRANCH)

SHELLCHECK_OPTS = \
	--shell=bash \
	--external-sources \
	--exclude=SC1091
SHELLCHECK_COLOR := $(if $(CI),--color=never,--color)
SHELLCHECK_FILES := { git ls-files "*.sh" -z; git grep -Plz "\A\#\!.*bash"; } | sort -zu

shellcheck:
	@$(SHELLCHECK_FILES) | xargs -0 -r docker run --rm -v $(HERE):/mnt -w /mnt \
		koalaman/shellcheck:stable $(SHELLCHECK_OPTS) $(SHELLCHECK_COLOR)

shellcheck_fix:
	@$(SHELLCHECK_FILES) | while IFS= read -r -d '' file; do \
		diff=$$(docker run --rm -v $(HERE):/mnt -w /mnt koalaman/shellcheck:stable $(SHELLCHECK_OPTS) --format=diff "$$file" 2>/dev/null); \
		if [ -n "$$diff" ] && ! echo "$$diff" | patch -p1 >/dev/null 2>&1; then echo "\033[31m$$file\033[0m"; \
		elif [ -n "$$diff" ]; then echo "$$file"; \
		else echo "\033[2m$$file\033[0m"; fi \
	done

.PHONY: all \
	build-base build-community \
	shellcheck shellcheck_fix
