.PHONY: all clean test coverage lint deps_gen deps depspurge proto mocks mocks_clean bin libs hsr

## Functions to simplify rules.
# Get parent directory of $(1), stripping trailing /
parent = $(patsubst %/,%,$(dir $(1)))
# Convert mock src input to mock src output.
mock_src_in_to_out = $(dir $(1))mock_$(notdir $(call parent,$(1)))/$(notdir $(1))
# Convert mock std input to mock std output
mock_std_in_to_out = lib/mocks/mock_$(dir $(1))$(shell echo $(notdir $(1)) | tr 'A-Z' 'a-z').go

SHELL=/bin/bash
LOCAL_DIRS = $(shell find * -maxdepth 0 -type d | grep -v '^vendor$$')
LOCAL_PKGS = $(patsubst %, ./%/..., $(LOCAL_DIRS))
LOCAL_GOBIN = $(shell realpath -s $$PWD/../bin)
BUILD_CHAIN = $(shell go version)
HEAD_TAG = $(shell git tag -l --points-at HEAD)
GIT_DESC = $(shell git describe --tags)
DIRTY = $(shell git diff HEAD --quiet || echo '-dirty')
LINK_FLAGS = -X \"github.com/scionproto/scion/go/lib/env.StartupBuildChain=$(BUILD_CHAIN)\"
ifeq ($(HEAD_TAG), $(GIT_DESC))
    BUILD_DATE = $(shell date -u -Is)
    LINK_FLAGS += -X \"github.com/scionproto/scion/go/lib/env.StartupBuildDate=$(BUILD_DATE)\"
    # add commit, because on a clean tag `git describe --tags` doesn't contain the commit.
    GIT_TAG =$(GIT_DESC)-$(shell git rev-parse --short HEAD)$(DIRTY)
else
    GIT_TAG = $(GIT_DESC)$(DIRTY)
endif
LINK_FLAGS += -X \"github.com/scionproto/scion/go/lib/env.StartupVersion=$(GIT_TAG)\"
GOTAGS = assert
MOCK_SRC_IN =\
    lib/infra/common.go \
    lib/pathdb/pathdb.go \
    lib/snet/interface.go \
    lib/snet/internal/pathsource/pathsource.go \
    lib/snet/internal/ctxmonitor/ctxmonitor.go \
    lib/pathmgr/pathmgr.go \
    lib/snet/snetproxy/reconnecter.go \
    lib/snet/snetproxy/io.go \
    lib/log/wrappers.go \
    lib/sciond/sciond.go
MOCK_SRC_OUT = $(foreach in,$(MOCK_SRC_IN),$(call mock_src_in_to_out,$(in)))
# This uses the format <stdlib package>/<interface>:
MOCK_STD_IN = net/Addr net/PacketConn
MOCK_STD_OUT = $(foreach in,$(MOCK_STD_IN),$(call mock_std_in_to_out,$(in)))

all: deps_gen bin libs

clean: mocks_clean
	rm -f gocover.html vendor/.deps.stamp
	GOBIN=${LOCAL_GOBIN} go clean ${LOCAL_PKGS}
	go clean -i ./vendor/...
	cd proto && $(MAKE) clean

test: deps_gen
	GOCONVEY_REPORTER=story govendor test +local

test_running: deps_gen
	GOCONVEY_REPORTER=story govendor test +local -tags infrarunning

coverage: deps_gen
	set -o pipefail; GOCONVEY_REPORTER=story gocov test ${LOCAL_PKGS} | gocov-html > gocover.html
	@echo
	@echo "Go coverage report here: file://$$PWD/gocover.html"

lint: vendor/.deps.stamp
	@echo "======> impi"
	@# Skip vendored, generated and CGO (https://github.com/pavius/impi/issues/5) files.
	impi --local github.com/scionproto/scion --scheme stdThirdPartyLocal --skip '^vendor/' --skip '\.capnp\.go$$' --skip '/mock_[^/]*/' --skip '/c\.go$$' ./...
	@echo "======> gofmt"
	out=$$(gofmt -d -s ${LOCAL_DIRS}); if [ -n "$$out" ]; then echo "$$out"; exit 1; fi
	@echo "======> linelen (lll)"
	out=$$(find . -type f -iname '*.go' -a '!' '(' -ipath './vendor/*' -o -ipath '*/*.capnp.go' -o -ipath '*/*.gen.go' -o -ipath '*/mock_*/*' ')' | lll -w 4 -l 100 --files -e '`comment:"|`ini:"|https?:'); if [ -n "$$out" ]; then echo "$$out"; exit 1; fi
	@echo "======> go vet"
	go vet ${LOCAL_PKGS}
	@echo "======> misspell"
	misspell ${LOCAL_DIRS}

deps_gen: deps proto

deps: vendor/.deps.stamp ../proto/go.capnp mocks
	@if [ -n "$$(govendor list -no-status +outside | grep -v '^context$$')" ]; then \
	    echo "ERROR: external/missing packages:"; \
	    govendor list +outside; \
	    exit 1; \
	fi;

vendor/.deps.stamp: vendor/vendor.json
	@[ -z "${QUIET}" ] && echo "$$(date -Iseconds) Remove unused deps"; \
	    govendor list -no-status +unused | while read pkg; do \
	    grep -q '"path": "'$$pkg'"' vendor/vendor.json && continue; \
	    echo "$$pkg"; \
	    govendor remove "$$pkg"; \
	done
	@[ -z "${QUIET}" ] && echo "$$(date -Iseconds) Syncing deps"; govendor sync -v
	@[ -z "${QUIET}" ] && echo "$$(date -Iseconds) Installing deps"; go install ./vendor/...
	touch $@

../proto/go.capnp:
	cp vendor/zombiezen.com/go/capnproto2/std/go.capnp ../proto/go.capnp

depspurge:
	rm -f vendor/.deps.stamp
	go clean -i ./vendor/...
	find vendor/* -maxdepth 0 -type d -exec rm -rf ./{} \;

proto: deps
	$(MAKE) -C proto

mocks: vendor/.deps.stamp $(MOCK_SRC_OUT) $(MOCK_STD_OUT)

mocks_clean:
	rm -f $(MOCK_SRC_OUT) $(MOCK_STD_OUT)
	for d in $(foreach f,$(MOCK_SRC_OUT) $(MOCK_STD_OUT),$(dir $(f))); do \
	    [ ! -e $$d ] || rmdir $$d; \
	done

define MOCK_SRC_template =
$(call mock_src_in_to_out,$(1)): $(1)
	@mkdir -p $$$$(dirname $$@)
	mockgen -destination=$$@ -source=$$<
endef
$(foreach in,$(MOCK_SRC_IN),$(eval $(call MOCK_SRC_template,$(in))))

define MOCK_STD_template =
$(call mock_std_in_to_out,$(1)): $$(shell which go)
	@mkdir -p $$$$(dirname $$@)
	mockgen -destination=$$@ $(call parent,$(1)) $(notdir $(1))
endef
$(foreach in,$(MOCK_STD_IN),$(eval $(call MOCK_STD_template,$(in))))

bin: deps_gen
	GOBIN=${LOCAL_GOBIN} govendor install --tags "$(GOTAGS)" -ldflags "$(LINK_FLAGS)" -v +local,program
	@sudo -p "go:braccept [sudo] password for %p: " true
	sudo setcap cap_net_admin,cap_net_raw+ep ../bin/braccept

libs: deps_gen
	govendor install --tags "$(GOTAGS)" -v +local,^program

hsr: libs
	GOBIN=${LOCAL_GOBIN} go install -v -tags "$(GOTAGS) hsr" ./border/...
	@sudo -p "go:hsr [sudo] password for %p: " true
	sudo setcap cap_dac_read_search,cap_dac_override,cap_sys_admin,cap_net_raw+ep ../bin/border
