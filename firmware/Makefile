## -----------------------------------------------------------------------------
## Common variables
## -----------------------------------------------------------------------------
PROJ_DIR  := $(patsubst %/,%,$(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
PROJ_NAME := $(lastword $(subst /, ,$(PROJ_DIR)))
PROJ_YAML := $(wildcard $(PROJ_DIR)/$(PROJ_NAME).yaml)
BUILD_DIR := $(PROJ_DIR)/build
GEN_DIR   := $(PROJ_DIR)/generators
TEMP_DIR  := $(GEN_DIR)/templates
GUI_MODE  ?= batch

define IMPORT_MOD
include  $(1)/modules/$(2)/Make.cfg
endef

define ADD_MOD
$(1)_SRCS += $(patsubst %,modules/$(2)/%,$($(2)_SRCS))
endef

define ADD_MODULE
$(eval $(call IMPORT_MOD,$1,$2))
$(eval $(call ADD_MOD,$1,$2))
endef

include Make.cfg

## -----------------------------------------------------------------------------
## Common targets
## -----------------------------------------------------------------------------
$(BUILD_DIR):
	@mkdir -p $(BUILD_DIR)
$(BUILD_DIR)/%/:
	@mkdir -p $(BUILD_DIR)/$*

clean_all:
	rm -r $(BUILD_DIR)

## -----------------------------------------------------------------------------
## Software variables
## -----------------------------------------------------------------------------
SW_BUILD := $(BUILD_DIR)/$(SW_TARG)
SW_SRCS += $(patsubst %,$(SW_TARG)/%,$($(SW_TARG)_SRCS))
SW_INCS += $(patsubst %,$(SW_TARG)/%,$($(SW_TARG)_INCS))

## -----------------------------------------------------------------------------
## Software targets
## -----------------------------------------------------------------------------
.PHONY: sw clean_sw

# Build ELF file from target sources, no sub-build directory needed
$(SW_BUILD)/$(SW_TARG).elf: $(SW_SRCS) $(SW_INCS) |$(SW_BUILD)/
	@$(SW_TOOL) $(SW_SRCS) $(SW_INCS) -o $(SW_BUILD)/$(SW_TARG).elf

# Friendly target name
sw: $(SW_BUILD)/$(SW_TARG).elf

# Remove SW output files
clean_sw:
	rm -r $(BUILD_DIR)/*.elf

## -----------------------------------------------------------------------------
## Firmware variables
## -----------------------------------------------------------------------------
FW_BUILD := $(BUILD_DIR)/$(FW_TARG)
FW_SRCS  := $(patsubst %,$(FW_TARG)/%,$($(FW_TARG)_SRCS))
FW_IPS   := $(patsubst %,$(FW_TARG)/%,$($(FW_TARG)_IPS))
FW_BDS   := $(patsubst %,$(FW_TARG)/%,$($(FW_TARG)_BDS))
FW_SIMS  := $(patsubst %,$(FW_TARG)/%,$($(FW_TARG)_SIMS))

GENERICS ?=

ifeq ($(GUI_MODE),batch)
VIV_CMD := vivado -notrace -mode tcl $(FW_BUILD)/$(FW_TARG).xpr -source $(GEN_DIR)/scripts/vivado.tcl -tclargs $(PROJ_YAML)
else
VIV_CMD := vivado -notrace $(FW_BUILD)/$(FW_TARG).xpr -source $(GEN_DIR)/scripts/vivado.tcl -tclargs $(PROJ_YAML)
endif

## -----------------------------------------------------------------------------
## Firmware targets
## -----------------------------------------------------------------------------
.PHONY: vivado project bd ip lint synth impl program sim clean_fw

# Create project, create sub-build directory
$(FW_BUILD)/$(FW_TARG).xpr: |$(FW_BUILD)/
	@if [ ! -d $(FW_TARG) ]; then \
		echo "ERROR: FW target $(FW_TARG) does not exist"; \
		exit 1; \
	fi
	@echo "Creating Project"
	@cd $(FW_BUILD); \
	vivado -notrace -mode tcl -source $(GEN_DIR)/scripts/vivado.tcl \
		-tclargs $(PROJ_YAML) PROJ

# Build block designs if script is out of date
$(FW_BUILD)/__bd_done: $(FW_BDS) |$(FW_BUILD)/$(FW_TARG).xpr
	@echo "Building Block Designs $(FW_BDS)"
	@cd $(FW_BUILD); $(VIV_CMD) BD $(FW_BDS)
	@touch $(FW_BUILD)/__bd_done

# Build IP blocks if any are out of date, vivado will determine which need to be regenerated
$(FW_BUILD)/__ip_done: $(FW_IPS) $(FW_BUILD)/__bd_done |$(FW_BUILD)/$(FW_TARG).xpr
	@echo "Building IPS $(FW_IPS)"
	@cd $(FW_BUILD); $(VIV_CMD) IP $(FW_IPS)
	@touch $(FW_BUILD)/__ip_done

# Synthesize design, dependent on BD, IP, and sources
$(FW_BUILD)/__synth_done: $(FW_BUILD)/__ip_done $(FW_BUILD)/__bd_done $(FW_SRCS)
	@echo "Synthesizing Design"
	@cd $(FW_BUILD); $(VIV_CMD) SYNTH $(FW_SIMS) $(FW_SRCS)
	@touch $(FW_BUILD)/__synth_done

# Implement design, dependent on synthesis
$(FW_BUILD)/__impl_done: $(FW_BUILD)/__synth_done
	@echo "Implementing Design"
	@cd $(FW_BUILD); $(VIV_CMD) IMPL $(FW_SRCS)
	@touch $(FW_BUILD)/__impl_done
	@cp $(FW_BUILD)/$(FW_TARG).runs/impl_1/$(FW_TARG).bit $(FW_BUILD)/$(FW_TARG).bit

# Friendly target name for creating project
project: |$(FW_BUILD)/$(FW_TARG).xpr

# Friendly target name for generating IP blocks
ip: project $(FW_BUILD)/__ip_done

# Friendly target name for generating block designs
bd: project $(FW_BUILD)/__bd_done

# Friendly target name for sythesis
synth: $(FW_BUILD)/__synth_done

# Friendly target name for implementation
impl: project $(FW_BUILD)/__impl_done

# Friendly target name for linting design
lint: $(FW_BUILD)/__ip_done $(FW_BUILD)/__bd_done $(FW_SRCS)
	@echo "Linting Design"
	@cd $(FW_BUILD); \
	$(VIV_CMD) LINT $(FW_SIMS) $(FW_SRCS); \

# Friendly target name for simulating design
sim: $(FW_BUILD)/__ip_done $(FW_BUILD)/__bd_done
	@echo "Simulating Design"
	@cd $(FW_BUILD); \
	$(VIV_CMD) SIM;

program: $(FW_BUILD)/__impl_done
	@echo "Programming device"
	@if [ ! -f $(FW_BUILD)/$(FW_TARG).bit ]; then \
		echo "Bitstream not found, run implementation first"; \
	else \
		cd $(FW_BUILD); $(VIV_CMD) PROG $(DEV_IP); \
	fi

# Friendly target name for opening design in GUI
vivado: project
	@cd $(FW_BUILD); \
	vivado $(FW_TARG).xpr &
	
# Remove FW output files
clean_fw:
	rm -r $(FW_BUILD)

## -----------------------------------------------------------------------------
## SoC App variables
## -----------------------------------------------------------------------------
PLAT_TARG ?= $(FW_TARG)_platform

SOC_BUILD := $(BUILD_DIR)/$(SOC_TARG)
SOC_SRCS  := $(patsubst %,$(SOC_TARG)/%,$($(SOC_TARG)_SRCS))
SOC_INCS  := $(patsubst %,$(SOC_TARG)/%,$($(SOC_TARG)_INCS))
SOC_LNK   := $(patsubst %,$(SOC_TARG)/%,$($(SOC_TARG)_LNK))
SOC_ELF   := $(SOC_BUILD)/$(SOC_TARG).elf

BIT_STR   := $(subst /,\/,$(FW_BUILD)/$(FW_TARG).bit)

## -----------------------------------------------------------------------------
## SoC targets
## -----------------------------------------------------------------------------
.PHONY: vitis platform app run load clean_soc

# Generate platform, dependent on FW target's XSA
$(BUILD_DIR)/__platform_$(FW_TARG)_done: $(FW_BUILD)/__impl_done |$(SOC_BUILD)/
	@echo "Building SoC Platform"
	@if [ -d $(SOC_BUILD)/$(FW_TARG)_platform ]; then \
		echo "Removing existing platform"; \
		rm -r $(SOC_BUILD)/$(FW_TARG)_platform;\
	fi
	@cd $(SOC_BUILD);\
	vitis -s $(GEN_DIR)/scripts/xil_vitis.py CREATE_PLAT $(FW_TARG)
	@touch $(BUILD_DIR)/__platform_$(FW_TARG)_done

# Generate SoC application, dependent on platform
$(BUILD_DIR)/BOOT.bin: $(BUILD_DIR)/__platform_$(FW_TARG)_done $(SOC_SRCS) $(SOC_INCS)
	@echo "Building app $(SOC_TARG)"
	@if [ ! -d $(SOC_TARG) ]; then \
		echo "ERROR: SoC Application target $(SOC_TARG) does not exist"; \
		exit 1; \
	fi
	@cd $(SOC_BUILD); \
	vitis -s $(GEN_DIR)/scripts/xil_vitis.py BUILD_APP $(FW_TARG) $(SOC_TARG) $(SOC_TEMPLATE) $(SOC_LNK) $(SOC_INCS) $(SOC_SRCS)
	@sed -i 's/: ".*\.bit/: "$(BIT_STR)/' $(SOC_BUILD)/$(SOC_TARG)/_ide/launch.json
	@cp $(SOC_BUILD)/$(SOC_TARG)/build/$(SOC_TARG).elf $(SOC_BUILD)/$(SOC_TARG).elf
	@bootgen -w -arch zynq -image $(SOC_BUILD)/$(SOC_TARG)/build/$(SOC_TARG).bif -o $(BUILD_DIR)/BOOT.bin

# Friendly target to generate SoC hardware platform
platform: |$(SOC_BUILD)/ $(BUILD_DIR)/__platform_$(FW_TARG)_done

# Friendly target to generate SoC app
app: |$(SOC_BUILD)/ $(BUILD_DIR)/BOOT.bin

run: $(BUILD_DIR)/BOOT.bin
	@echo "Running application $(SOC_TARG)"
	@cd $(BUILD_DIR); \
	xsdb $(GEN_DIR)/scripts/load_elf.tcl $(SOC_BUILD)/$(FW_TARG)_platform/hw/sdt/ps7_init.tcl $(SOC_BUILD)/$(SOC_TARG).elf $(FW_BUILD)/$(FW_TARG).bit
	
load: $(BUILD_DIR)/BOOT.bin
	@echo "Loading boot image"
	@cd $(BUILD_DIR);\
		program_flash -f $(BUILD_DIR)/BOOT.bin -offset 0 -flash_type qspi-x4-single -fsbl \
			$(SOC_BUILD)/$(FW_TARG)_platform/zynq_fsbl/build/fsbl.elf -url TCP:127.0.0.1:3121 

# Friendly target name for opening design in GUI
vitis: $(SOC_BUILD)
	@cd $(SOC_BUILD); \
	vitis -w . &
	
clean_soc:
	rm -rf $(SOC_BUILD)
	rm -f $(BUILD_DIR)/__platform*
	rm -f $(BUILD_DIR)/BOOT.bin

