# Build system related variables
MAKE = make
GOCMD = go
GOOS = $(shell go env GOOS)
GOARCH = $(shell go env GOARCH)
GOHOSTOS = $(shell go env GOHOSTOS)
GOHOSTARCH = $(shell go env GOHOSTARCH)
FIPS_CRYPTO_BACKEND_DEFAULT = opensslcrypto
FIPS_CRYPTO_BACKEND =
HASH = sha
HASH_ALGORITHM = 256
CLI_V1_VERSION_TAG = 0.0.0
CLI_V1_LOCATION = 
USE_LEGACY_EXECUTABLE_NAME =
LDFLAGS = -s -w
SHASUM_CMD = shasum

# Make directories per convention
prefix = /usr/local
exec_prefix = $(prefix)
bindir = $(exec_prefix)/bin

# Project related variables
WORKING_DIR = $(CURDIR)
BUILD_DIR = $(WORKING_DIR)/_bin
CACHE_DIR = $(WORKING_DIR)/_cache
SRCS = $(shell find $(WORKING_DIR) -type f -name '*.go')

# load cached variables if available
-include $(CACHE_DIR)/variables.mk

# the platform string used by the deployed binaries is not excatly OS-ARCH so we need to translate a bit
_GO_OS = $(GOOS)
_V1_OS = $(GOOS)
_V1_ARCH = $(GOARCH)
_EMPTY =
_EXE_POSTFIX =
_FIPS_EXTENSION = fips
_SEPARATOR = _

ifeq ($(_V1_OS), darwin)
	_V1_OS = macos
#temporarily ignoring any architecture for macos v1 binaries, this will enable to natively compile v2 as darwin arm64 and bundle v1 as amd64
	_V1_ARCH = 
else ifeq ($(_V1_OS), windows)
	_V1_OS = win
	_EXE_POSTFIX = .exe
endif

ifeq ($(_V1_ARCH), amd64)
	_V1_ARCH = 
endif

ifeq ($(_V1_ARCH), $(_EMPTY))
	V1_PLATFORM_STING = $(_V1_OS)
else 
	V1_PLATFORM_STING = $(_V1_OS)-$(_V1_ARCH)
endif

ifeq ($(_GO_OS), alpine)
	_GO_OS = linux
endif

# some globally assembled variables
APPLICATION_NAME = snyk
TEST_NAME = $(APPLICATION_NAME)$(_SEPARATOR)tests
V2_PLATFORM_STRING = $(GOOS)$(_SEPARATOR)$(GOARCH)
V2_EXECUTABLE_NAME = $(APPLICATION_NAME)$(_SEPARATOR)$(V2_PLATFORM_STRING)$(_EXE_POSTFIX)

ifneq ($(USE_LEGACY_EXECUTABLE_NAME), $(_EMPTY))
	V2_EXECUTABLE_NAME = $(V1_EXECUTABLE_NAME)
	_SEPARATOR = -
endif 

V1_EXECUTABLE_NAME = $(APPLICATION_NAME)-$(V1_PLATFORM_STING)$(_EXE_POSTFIX)
V2_DIRECTORY = $(WORKING_DIR)/internal/cliv2
V1_DIRECTORY = $(WORKING_DIR)/internal/embedded/cliv1
V1_EMBEDDED_FILE_TEMPLATE = $(V1_DIRECTORY)/embedded_binary_template.txt
V1_EMBEDDED_FILE_OUTPUT = embedded$(_SEPARATOR)$(V2_PLATFORM_STRING).go
V1_WORKING_DIR = $(WORKING_DIR)/..
V1_BUILD_TYPE = build:prod
V1_BINARY_FOLDER = ts-cli-binaries
HASH_STRING = $(HASH)$(HASH_ALGORITHM)
SIGN_SCRIPT = $(WORKING_DIR)/scripts/sign_$(_GO_OS).sh
ISSIGNED_SCRIPT = $(WORKING_DIR)/scripts/issigned_$(_GO_OS).sh
EMBEDDED_DATA_DIR = $(WORKING_DIR)/internal/embedded/_data

ifeq ($(GOHOSTOS), windows)
	SPECIAL_SHELL = powershell
	SHASUM_CMD = $(SPECIAL_SHELL) $(WORKING_DIR)/scripts/shasum.ps1
	SIGN_SCRIPT = $(SPECIAL_SHELL) $(WORKING_DIR)/scripts/sign_$(GOHOSTOS).ps1
	ISSIGNED_SCRIPT = $(SPECIAL_SHELL) $(WORKING_DIR)/scripts/issigned_$(GOHOSTOS).ps1
endif

ifeq ($(_GO_OS), windows)
	FIPS_CRYPTO_BACKEND_DEFAULT = cngcrypto
endif

# some make file variables
LOG_PREFIX = --

$(BUILD_DIR):
	@mkdir $@

$(CACHE_DIR):
	@mkdir $@

$(CACHE_DIR)/variables.mk: $(CACHE_DIR)
	@printf "GOOS=$(GOOS)\nGOARCH=$(GOARCH)\nUSE_LEGACY_EXECUTABLE_NAME=$(USE_LEGACY_EXECUTABLE_NAME)\n" > $(CACHE_DIR)/variables.mk

$(V1_DIRECTORY)/$(V1_EMBEDDED_FILE_OUTPUT):
	@echo "$(LOG_PREFIX) Generating ( $(V1_DIRECTORY)/$(V1_EMBEDDED_FILE_OUTPUT) )"
	@sed -e 's/FILENAME/$(V1_EXECUTABLE_NAME)/g' $(V1_EMBEDDED_FILE_TEMPLATE) > $(V1_DIRECTORY)/$(V1_EMBEDDED_FILE_OUTPUT)

$(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME):
	@echo "$(LOG_PREFIX) Copying cliv1 executable ( $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME) )"
	@cp $(CLI_V1_LOCATION)/$(V1_EXECUTABLE_NAME) $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME)

$(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING): $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME)
	@echo "$(LOG_PREFIX) Copying cliv1 checksum ( $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING) )"
	@cp $(CLI_V1_LOCATION)/$(V1_EXECUTABLE_NAME).$(HASH_STRING) $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING)

.PHONY: _validate_sha_v1
_validate_sha_v1: $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING)
	@echo "$(LOG_PREFIX) Validating checksum ( $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING) )"
	@cd $(V1_DIRECTORY) && $(SHASUM_CMD) -b -q -a $(HASH_ALGORITHM) -c $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING)

# separate dependency target
.PHONY: dependencies
dependencies: $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME) $(V1_DIRECTORY)/$(V1_EXECUTABLE_NAME).$(HASH_STRING) _validate_sha_v1

.PHONY: summary
summary:
	@echo "$(LOG_PREFIX) Build Summary"
	@echo "$(LOG_PREFIX)  Using Go installation:        $(shell which go)"
	@echo "$(LOG_PREFIX)  Non-Standard Crypto-Backend:  $(FIPS_CRYPTO_BACKEND)"
	@echo "$(LOG_PREFIX)  Building Binary:              $(BUILD_DIR)/$(V2_EXECUTABLE_NAME)"
	@echo "$(LOG_PREFIX)  Building version:             $(CLI_V1_VERSION_TAG)"


# prepare the workspace and cache global parameters
.PHONY: configure
configure: summary $(CACHE_DIR) $(CACHE_DIR)/variables.mk $(V1_DIRECTORY)/$(V1_EMBEDDED_FILE_OUTPUT) dependencies $(CACHE_DIR)/prepare-3rd-party-licenses

$(BUILD_DIR)/$(V2_EXECUTABLE_NAME): $(BUILD_DIR) $(SRCS)
	@echo "$(LOG_PREFIX) Building ( $(BUILD_DIR)/$(V2_EXECUTABLE_NAME) )"
	@GOEXPERIMENT=$(FIPS_CRYPTO_BACKEND) GOOS=$(_GO_OS) GOARCH=$(GOARCH) $(GOCMD) build -tags=application -ldflags="$(LDFLAGS) -X main.internalOS=$(GOOS) -X github.com/snyk/cli/cliv2/internal/embedded/cliv1.snykCLIVersion=$(CLI_V1_VERSION_TAG)" -o $(BUILD_DIR)/$(V2_EXECUTABLE_NAME) $(WORKING_DIR)/cmd/cliv2/main.go

.PHONY: fips
fips:
# adapt executable name if fips is used
ifneq (, $(_EXE_POSTFIX))
	$(eval V2_EXECUTABLE_NAME := $(subst $(_EXE_POSTFIX),$(_SEPARATOR)$(_FIPS_EXTENSION)$(_EXE_POSTFIX), $(V2_EXECUTABLE_NAME)))
else
	$(eval V2_EXECUTABLE_NAME := "$(V2_EXECUTABLE_NAME)$(_SEPARATOR)$(_FIPS_EXTENSION)")
endif

# assign name of crypto backend to use
	$(eval FIPS_CRYPTO_BACKEND := $(FIPS_CRYPTO_BACKEND_DEFAULT))
	@echo "$(LOG_PREFIX) FIPS"

.PHONY: build
build: configure $(BUILD_DIR)/$(V2_EXECUTABLE_NAME)

$(WORKING_DIR)/internal/httpauth/generated/httpauth_generated_mock.go:
	@$(GOCMD) generate ./internal/httpauth/

$(WORKING_DIR)/internal/httpauth/generated/spnego_generated_mock.go:
	@$(GOCMD) generate ./internal/httpauth/


$(CACHE_DIR)/prepare-3rd-party-licenses: 
	@echo "$(LOG_PREFIX) Preparing 3rd Party Licenses"
	@GOOS=$(GOHOSTOS) GOARCH=$(GOHOSTARCH) scripts/prepare_licenses.sh > $(CACHE_DIR)/prepare-3rd-party-licenses 2> /dev/null

.PHONY: generate
generate: $(WORKING_DIR)/internal/httpauth/generated/httpauth_generated_mock.go $(WORKING_DIR)/internal/httpauth/generated/spnego_generated_mock.go

.PHONY: whiteboxtest
whiteboxtest:
	@echo "$(LOG_PREFIX) Running $@"
	@$(GOCMD) test -cover ./...

.PHONY: test
test: whiteboxtest

.PHONY: lint
lint:
	./scripts/lint.sh

.PHONY: format
format:
	gofmt -w -l -e .

.PHONY: SIGN_SCRIPT
SIGN_SCRIPT:
	@echo "$(LOG_PREFIX) Running $(SIGN_SCRIPT) ( $(BUILD_DIR) )"
	$(SIGN_SCRIPT) $(BUILD_DIR) $(V2_EXECUTABLE_NAME)

.PHONY: ISSIGNED_SCRIPT
ISSIGNED_SCRIPT:
	$(ISSIGNED_SCRIPT) $(BUILD_DIR)/$(V2_EXECUTABLE_NAME)


.PHONY: sign
sign: SIGN_SCRIPT ISSIGNED_SCRIPT

.PHONY: test-signature
test-signature: ISSIGNED_SCRIPT

# Typescript CLI targets
$(V1_WORKING_DIR)/$(V1_BINARY_FOLDER)/$(V1_EXECUTABLE_NAME):
	@echo "$(LOG_PREFIX) Building legacy CLI"
	@cd $(V1_WORKING_DIR) && npm i && npm run $(V1_BUILD_TYPE)
	@$(MAKE) -C $(V1_WORKING_DIR) $(V1_BINARY_FOLDER)/$(V1_EXECUTABLE_NAME) BINARY_RELEASES_FOLDER_TS_CLI=$(V1_BINARY_FOLDER)
	
.PHONY: build-ts-cli
build-ts-cli: $(V1_WORKING_DIR)/$(V1_BINARY_FOLDER)/$(V1_EXECUTABLE_NAME)
	$(eval CLI_V1_VERSION_TAG := $(shell cat $(V1_WORKING_DIR)/$(V1_BINARY_FOLDER)/version))
	$(eval CLI_V1_LOCATION := $(V1_WORKING_DIR)/$(V1_BINARY_FOLDER)/)
	
.PHONY: clean-ts-cli
clean-ts-cli:
	@echo "$(LOG_PREFIX) Cleaning legacy CLI"
	@$(MAKE) -C $(V1_WORKING_DIR) clean-ts BINARY_RELEASES_FOLDER_TS_CLI=$(V1_BINARY_FOLDER)

# build the full CLI (Typescript+Golang)
.PHONY: build-full
build-full: build-ts-cli build

# clean the full CLI (Typescript+Golang)
.PHONY: clean-full
clean-full: | clean-ts-cli clean

.PHONY: all
all: build-full

.PHONY: clean
clean:
	@echo "$(LOG_PREFIX) Cleaning ( $(DESTDIR)$(bindir)/$(V2_EXECUTABLE_NAME) )"
	@$(GOCMD) clean
	@rm -f -r $(BUILD_DIR)
	@rm -f -r $(CACHE_DIR)
	@rm -f $(V1_DIRECTORY)/$(APPLICATION_NAME)-*
	@rm -f $(V1_DIRECTORY)/$(V1_EMBEDDED_FILE_OUTPUT)
	@rm -f -r $(EMBEDDED_DATA_DIR)/licenses

.PHONY: install
install:
	@echo "$(LOG_PREFIX) Installing ( $(DESTDIR)$(bindir)/$(V2_EXECUTABLE_NAME) )"
	@mkdir -p $(DESTDIR)$(bindir)
	@rm -f $(DESTDIR)$(bindir)/$(V2_EXECUTABLE_NAME)
	@cp $(BUILD_DIR)/$(V2_EXECUTABLE_NAME) $(DESTDIR)$(bindir)

.PHONY: help
help:
	@echo "\n Main targets (Golang only):"
	@echo "$(LOG_PREFIX) lint"
	@echo "$(LOG_PREFIX) format"
	@echo "$(LOG_PREFIX) fips"
	@echo "$(LOG_PREFIX) build"
	@echo "$(LOG_PREFIX) sign"
	@echo "$(LOG_PREFIX) test"
	@echo "$(LOG_PREFIX) test-signature"
	@echo "$(LOG_PREFIX) install"
	@echo "$(LOG_PREFIX) clean"
	@echo "\nFull local targets (Typescript + Golang):"
	@echo "$(LOG_PREFIX) build-full"               
	@echo "$(LOG_PREFIX) clean-full"
	@echo "\nAvailable parameter:"
	@echo "$(LOG_PREFIX) GOOS                       Specify Operating System to compile for (see golang GOOS, default=$(GOOS))"
	@echo "$(LOG_PREFIX) GOARCH                     Specify Architecture to compile for (see golang GOARCH, default=$(GOARCH))"
	@echo "$(LOG_PREFIX) CLI_V1_VERSION_TAG         Version of the CLIv1 to bundle"
	@echo "$(LOG_PREFIX) CLI_V1_LOCATION            Filesystem location of CLIv1 binaries to bundle, if specified, CLI_V1_VERSION_TAG is also required"
	@echo "$(LOG_PREFIX) prefix                     Installation prefix (default=$(prefix))"
	@echo "$(LOG_PREFIX) DESTDIR                    For staged installations"
