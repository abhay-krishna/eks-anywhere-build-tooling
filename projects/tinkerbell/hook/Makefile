BASE_DIRECTORY:=$(abspath ../../../)
GIT_TAG=$(shell cat GIT_TAG)
KERNEL_VERSION:=$(shell cat LINUX_KERNEL_VERSION)
GOLANG_VERSION=$(shell cat GOLANG_VERSION)
REPO=hook
REPO_OWNER=tinkerbell

BASE_IMAGE_NAME?=eks-distro-minimal-base
DOCKERFILE_FOLDER=./docker/linux/$(IMAGE_NAME)

HOOK_BOOTKIT_IMAGE_COMPONENT=tinkerbell/hook-bootkit
HOOK_DOCKER_IMAGE_COMPONENT=tinkerbell/hook-docker
HOOK_DIND_IMAGE_COMPONENT=tinkerbell/hook-dind
KERNEL_IMAGE_COMPONENT=tinkerbell/hook-kernel
HOOK_RUNC_IMAGE_COMPONENT=tinkerbell/hook-runc
HOOK_CONTAINERD_IMAGE_COMPONENT=tinkerbell/hook-containerd
HOOK_EMBEDDED_IMAGE_COMPONENT=tinkerbell/hook-embedded
HOOK_IP_IMAGE_COMPONENT=tinkerbell/hook-ip
HOOK_MDEV_IMAGE_COMPONENT=tinkerbell/hook-mdev

IMAGE_NAMES=hook-bootkit hook-docker hook-runc hook-containerd kernel hook-ip hook-mdev hook-dind hook-embedded
# historically we have included these two images in our bundle even tho we really do not
# need them at runtime since they are built into the final vmlinuz/initramfs files
# when in codebuild we run the combine-images build, only bother merging these two
# merging the others is actually a bit more complicated since we override the dockerfile location
# the combine-images target does not easily support this case since we have not needed it before
ifneq ($(findstring combine-images,$(RELEASE_TARGETS)),)
IMAGE_NAMES=hook-bootkit hook-docker
endif

BINARY_TARGET_FILES=hook-bootkit hook-docker
SOURCE_PATTERNS=./ ./
GO_MOD_PATHS=images/hook-bootkit images/hook-docker
# by default the first target is assumed to be the primary and will produce a attribution with no prefix
# for this build we want to prefix all the files
HOOK_BOOTKIT_ATTRIBUTION_OVERRIDE=hook-bootkit

CREATE_HOOK_FILES_PATTERN=$(REPO)/out/hook/vmlinuz
HOOK_PLATFORMS=$(call IF_OVERRIDE_VARIABLE,IMAGE_PLATFORMS,linux/$(BUILDER_PLATFORM_ARCH))
CREATE_HOOK_FILES=$(foreach platform,$(HOOK_PLATFORMS),$(CREATE_HOOK_FILES_PATTERN)-$(if $(findstring amd64,$(platform)),x86_64,aarch64))
HOOK_IMAGE_FILES=$(OUTPUT_DIR)/kernel/config-patches $(OUTPUT_DIR)/kernel/keys.asc $(OUTPUT_DIR)/kernel/configs/generic-$(KERNEL_MAJOR_MINOR).y-aarch64 $(OUTPUT_DIR)/kernel/configs/generic-$(KERNEL_MAJOR_MINOR).y-x86_64 

HAS_S3_ARTIFACTS=true
SIMPLE_CREATE_TARBALLS=false

FIX_LICENSES_HOOK_BOOTKIT_TARGET=$(REPO)/images/hook-bootkit/LICENSE
FIX_LICENSES_HOOK_DOCKER_TARGET=$(REPO)/images/hook-docker/LICENSE

KERNEL_MAJOR_V=v$(firstword $(subst ., ,$(KERNEL_VERSION))).x
KERNEL_MAJOR_MINOR=$(subst $(SPACE),.,$(wordlist 1,2,$(subst ., ,$(KERNEL_VERSION))))
DEFCONFIG=generic-$(basename $(KERNEL_VERSION)).y

KERNEL_ARCH=$(if $(findstring amd64,$(BUILDER_PLATFORM_ARCH)),x86_64,aarch64)
EKSA_DEFCONFIG=$(DEFCONFIG)-$(KERNEL_ARCH)-eksa
USER_DEFCONFIG=$(DEFCONFIG)-$(KERNEL_ARCH)
KERNEL_CONFIG_HOST_PATH=$(MAKE_ROOT)/$(OUTPUT_DIR)/kernel-config

HOOK_EMBEDDED_FOLDER=$(REPO)/images/hook-embedded
EMBEDDED_IMAGES=$(foreach platform,$(HOOK_PLATFORMS),$(OUTPUT_DIR)/hook-embedded/$(subst /,-,$(platform))/images)

# we need to set IMAGE_BUILD_ARGS here even though its the same as the default. 
# it is set in Common.mk on the images target (https://github.com/aws/eks-anywhere-build-tooling/blob/8b6d6d66974e881b22e3c9c8ea29adc26f7df5fd/Common.mk#L799)
# and the combine-images target (https://github.com/aws/eks-anywhere-build-tooling/blob/8b6d6d66974e881b22e3c9c8ea29adc26f7df5fd/Common.mk#L846)
# since combine-images has images as prereq target, the ?= does not really behavior as one might expect.
# the images target being the actual action, its version of the set takes prioirty and resets to empty
# setting it explicitly to empty here takes allows the combine-images override to take proirty
IMAGE_BUILD_ARGS=

BUILDSPECS=buildspec.yml buildspecs/combine-images.yml
BUILDSPEC_1_COMPUTE_TYPE=BUILD_GENERAL1_LARGE
BUILDSPEC_1_VARS_KEYS=IMAGE_PLATFORMS
BUILDSPEC_1_VARS_VALUES=IMAGE_PLATFORMS
BUILDSPEC_1_ARCH_TYPES=LINUX_CONTAINER ARM_CONTAINER
BUILDSPEC_1_DEPENDS_ON_OVERRIDE=containerd_containerd_linux_amd64 containerd_containerd_linux_arm64 tinkerbell_tink_linux_amd64 tinkerbell_tink_linux_arm64 tinkerbell_hub
BUILDSPEC_2_DEPENDS_ON_OVERRIDE=tinkerbell_hook_linux_amd64 tinkerbell_hook_linux_arm64

PROJECT_DEPENDENCIES=eksa/containerd/containerd eksa/tinkerbell/hub eksa/tinkerbell/tink

# Since we build the arm and amd binaries on difference instances in codebuild
# we do not want to delete missing files when s3 sync-ing from local to the bucket
# since this would clobber the other build's artifacts
UPLOAD_DO_NOT_DELETE=true

include $(BASE_DIRECTORY)/Common.mk


$(GATHER_LICENSES_TARGETS): | $(FIX_LICENSES_HOOK_BOOTKIT_TARGET) $(FIX_LICENSES_HOOK_DOCKER_TARGET)

# For local image builds we override to push to local registry.
%/images/amd64 %/images/arm64: IMAGE_OUTPUT=push=true
%/images/amd64 %/images/arm64: IMAGE_OUTPUT_TYPE=image

# hook-docker image required docker runtime.
# We are using eks-distro-minimal-base-glibc as the base and builder to install docker.
hook-docker/images/% hook-dind/images/%: BASE_IMAGE_NAME=eks-distro-minimal-base-iptables

hook-dind/images/%: DOCKERFILE_FOLDER=./docker/linux/hook-docker
hook-dind/images/%: IMAGE_TARGET=hook-dind

kernel-config/images/% kernel/images/%: IMAGE_BUILD_ARGS=KERNEL_MAJOR_V KERNEL_VERSION DEFCONFIG
kernel-config/images/% kernel/images/%: IMAGE_CONTEXT_DIR=$(OUTPUT_DIR)/kernel
kernel-config/images/% kernel/images/%: BASE_IMAGE=public.ecr.aws/amazonlinux/amazonlinux:2023
kernel-config/images/% kernel/images/%: DOCKERFILE_FOLDER=$(REPO)/kernel

$(call IMAGE_TARGETS_FOR_NAME, kernel): $(HOOK_IMAGE_FILES)
$(call IMAGE_TARGETS_FOR_NAME, kernel-config): $(HOOK_IMAGE_FILES)

kernel-config/images/%: IMAGE_TARGET=kernel-configurator
kernel-config/images/%: IMAGE_OUTPUT_TYPE=docker
kernel-config/images/%: IMAGE_IMPORT_CACHE=
kernel-config/images/%: LATEST_IMAGE=hook-kernel-builder:$(KERNEL_VERSION)

$(call IMAGE_TARGETS_FOR_NAME,hook-embedded): $(EMBEDDED_IMAGES)

# upstream hook builds these instead of using the linuxkit provided images
# building this from their dockerfile
hook-ip/images/%: DOCKERFILE_FOLDER=$(REPO)/images/hook-ip
hook-ip/images/%: IMAGE_CONTEXT_DIR=$(REPO)/images/hook-ip

hook-mdev/images/%: DOCKERFILE_FOLDER=$(REPO)/images/hook-mdev
hook-mdev/images/%: IMAGE_CONTEXT_DIR=$(REPO)/images/hook-mdev

s3-artifacts: $(CREATE_HOOK_FILES)

$(OUTPUT_DIR)/kernel/config-patches:
	@mkdir -p $(OUTPUT_DIR)/kernel
	cp -rf config-patches $@

$(OUTPUT_DIR)/kernel/%:
	@mkdir -p $(@D)
	cp -f $(REPO)/kernel/$* $@

$(CREATE_HOOK_FILES_PATTERN)-%: BUILD_ARCH=$(if $(findstring x86_64,$*),amd64,arm64)
$(CREATE_HOOK_FILES_PATTERN)-%: | $$(ENABLE_DOCKER)
	@source $(BUILD_LIB)/common.sh && build::common::use_go_version $(GOLANG_VERSION); \
	cd $(REPO); \
	DEBUG=yes HOOK_KERNEL_OCI_BASE=$(IMAGE_REPO)/$(KERNEL_IMAGE_COMPONENT) \
		HOOK_LK_CONTAINERS_OCI_BASE=$(IMAGE_REPO)/tinkerbell/ \
		HOOK_KERNEL_POINT_RELEASE=$(lastword $(subst ., ,$(KERNEL_VERSION))) \
		HOOK_KERNEL_OCI_VERSION=$(LATEST_TAG)$(IMAGE_TAG_SUFFIX) \
		HOOK_LK_CONTAINERS_OCI_VERSION=$(LATEST_TAG)$(IMAGE_TAG_SUFFIX) \
		./build.sh build hook-default-$(BUILD_ARCH); \
	cd ..; \
	mkdir -p $(OUTPUT_DIR)/hook/$(GIT_TAG); \
	cp $(REPO)/out/hook/vmlinuz-$* $(OUTPUT_DIR)/hook/$(GIT_TAG)/; \
	cp $(REPO)/out/hook/initramfs-$* $(OUTPUT_DIR)/hook/$(GIT_TAG)/; \
	mkdir -p $(ARTIFACTS_PATH); \
	cp -rf $(OUTPUT_DIR)/hook/* $(ARTIFACTS_PATH)

$(REPO)/%/LICENSE: | $(GO_MOD_DOWNLOAD_TARGETS)
#go-licenses requires a LICENSE file in each folder with the go.mod
	cp $(REPO)/LICENSE $@

$(OUTPUT_DIR)/hook-embedded/linux-%/images: MAKEFLAGS=--no-print-directory
# in codebuild this is set to arm64/amd64, for the images pulled during this target
# we never want to use the -<arch>
$(OUTPUT_DIR)/hook-embedded/linux-%/images: IMAGE_TAG_SUFFIX=
$(OUTPUT_DIR)/hook-embedded/linux-%/images: DIND_IMAGE=$(shell $(MAKE) var-value-LATEST_IMAGE IMAGE_NAME=hook-dind)
$(OUTPUT_DIR)/hook-embedded/linux-%/images: $(GIT_PATCH_TARGET)
	mkdir -p $(@D)
	echo "$$($(MAKE) -C $(MAKE_ROOT)/../tink var-value-LATEST_IMAGE IMAGE_NAME=tink-worker) 127.0.0.1/embedded/tink-worker true" > $@.txt
	for action in cexec kexec image2disk oci2disk writefile reboot; do \
		echo "$$($(MAKE) -C $(MAKE_ROOT)/../hub var-value-LATEST_IMAGE IMAGE_NAME=$$action) 127.0.0.1/embedded/$$action true" >> $@.txt; \
	done
	cat $@.txt
	@if [[ "$(JOB_TYPE)" == "presubmit" ]] && [[ "$(DOCKER_AVAILABLE)" == "false" ]]; then \
		mkdir -p $@; \
	else \
		source $(BUILD_LIB)/common.sh; \
		function docker() { \
			local -r subcommand="$${1:-}"; \
			if [[ "$$subcommand" == "pull" ]]; then \
				retry command docker "$$@"; \
			else \
				command docker "$$@"; \
			fi; \
		}; \
		docker pull $(DIND_IMAGE); \
		cd $(@D) && source $(MAKE_ROOT)/$(HOOK_EMBEDDED_FOLDER)/pull-images.sh $* $(DIND_IMAGE); \
		sudo chown -R 1000:1000 ./images; \
	fi

.PHONY: run-kernel-in-qemu
run-kernel-in-qemu: $(CREATE_HOOK_FILES)
	mkdir -p $(REPO)/out/linuxkit-hook-default-$(BUILDER_PLATFORM_ARCH)
	cp $(REPO)/out/hook/vmlinuz-$(KERNEL_ARCH) $(REPO)/out/linuxkit-hook-default-$(BUILDER_PLATFORM_ARCH)/hook-kernel
	cp $(REPO)/out/hook/initramfs-$(KERNEL_ARCH) $(REPO)/out/linuxkit-hook-default-$(BUILDER_PLATFORM_ARCH)/hook-initrd.img
	cd $(REPO); \
	DEBUG=yes TINK_WORKER_IMAGE=127.0.0.1/embedded/tink-worker \
		sudo -E ./build.sh run hook-default-$(BUILDER_PLATFORM_ARCH)

kernel-config-%: | ensure-docker
	@$(MAKE) kernel-config/images/$(BUILDER_PLATFORM_ARCH) IMAGE_NAMES=kernel-config
	@mkdir -p $(KERNEL_CONFIG_HOST_PATH)
	docker run -it -v $(KERNEL_CONFIG_HOST_PATH):/host hook-kernel-builder:$(KERNEL_VERSION) bash -c "cp .config /host/$(EKSA_DEFCONFIG) && make $* && cp -v .config /host/$(USER_DEFCONFIG)"

menuconfig: kernel-config-menuconfig

create-new-config-patch: kernel-config-menuconfig
	@diff $(KERNEL_CONFIG_HOST_PATH)/$(EKSA_DEFCONFIG) $(KERNEL_CONFIG_HOST_PATH)/$(USER_DEFCONFIG) || true


clean-repo: clean-extra

# the extract container images in the _output directory
# will have root permissions and require sudo to delete
.PHONY: clean-extra
clean-extra:	
	sudo rm -rf $(EMBEDDED_IMAGES) $(REPO)/out

########### DO NOT EDIT #############################
# To update call: make add-generated-help-block
# This is added to help document dynamic targets and support shell autocompletion
# Run make help for a formatted help block with all targets
include Help.mk
########### END GENERATED ###########################
