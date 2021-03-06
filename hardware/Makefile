############################################################################
############################################################################
##
## Copyright 2016-2018 International Business Machines
##
## Licensed under the Apache License, Version 2.0 (the "License");
## you may not use this file except in compliance with the License.
## You may obtain a copy of the License at
##
##     http://www.apache.org/licenses/LICENSE#2.0
##
## Unless required by applicable law or agreed to in writing, software
## distributed under the License is distributed on an "AS IS" BASIS,
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
## See the License for the specific language governing permissions AND
## limitations under the License.
##
############################################################################
############################################################################

## Variables required by other scripts.
## This Makefile is contained in the hardware directory.
## So, the root directory is one level above.
##
SHELL=/bin/bash
PLATFORM ?= $(shell uname -i)

export SNAP_ROOT=$(abspath ..)
export SNAP_HARDWARE_ROOT=$(SNAP_ROOT)/hardware
export LOGS_DIR=$(SNAP_HARDWARE_ROOT)/logs
export MAKE_TARGET=$(MAKECMDGOALS)
export SNAP_CFLAGS

snap_config_cflags = $(SNAP_ROOT)/.snap_config.cflags
snap_config_sh     = $(SNAP_ROOT)/.snap_config.sh
snap_env_sh        = $(SNAP_ROOT)/snap_env.sh

-include $(snap_config_cflags)
-include $(snap_config_sh)
-include $(snap_env_sh)

SIMULATOR ?= nosim

FPGA_CARD := $(shell echo $(FPGACARD) | tr '[:upper:]' '[:lower:]')
ifeq "$(CAPI20)" "y"
	export CAPI_VER=capi20
	export PSL9_IP_ARCHIVE=ibm.com_CAPI_PSL9_WRAP_2.00.zip
	export PSL9_IP_DIR=$(SNAP_HARDWARE_ROOT)/capi2-bsp/psl
else
	export CAPI_VER=capi10
endif

export DCP_ROOT ?= $(SNAP_ROOT)/dcp

export IGNORE_CHECK_PSL ?= $(USE_PRFLOW)

## variables just for this Makefile
SETUP_DIR=$(SNAP_HARDWARE_ROOT)/setup
BUILD_DIR=$(SNAP_HARDWARE_ROOT)/build
SIM_DIR=$(SNAP_HARDWARE_ROOT)/sim
SNAP_HDL_CORE=$(SNAP_HARDWARE_ROOT)/hdl/core
SNAP_HDL_HLS=$(SNAP_HARDWARE_ROOT)/hdl/hls
SNAP_SIM_CORE=$(SIM_DIR)/core
SNAP_SIM_NVME=$(SIM_DIR)/nvme

SNAP_SIMULATORS += irun xsim questa modelsim xcelium
SNAP_MODELS += .model_irun .model_xsim

SNAP_PP_FILES_VHD=$(SNAP_HDL_CORE)/psl_fpga.vhd        \
                  $(SNAP_HDL_CORE)/psl_accel.vhd       \
                  $(SNAP_HDL_CORE)/psl_accel_types.vhd \
                  $(SNAP_HDL_CORE)/snap_core.vhd       \
                  $(SNAP_HDL_CORE)/dma_buffer.vhd      \
                  $(SNAP_HDL_CORE)/mmio.vhd            \
                  $(SNAP_SIM_CORE)/top.sv              \
                  $(SNAP_HDL_HLS)/action_wrapper.vhd   \
                  $(SNAP_HDL_CORE)/snap_core_types.vhd

SNAP_TMP_FILES =  $(SIM_DIR)/README.txt

SNAP_ACTION_DCP=$(DCP_ROOT)/user_action_synth.dcp
SNAP_BASE_DCP=$(DCP_ROOT)/snap_static_region_bb.dcp
PSL_DCP_TYPE=$(shell $(SNAP_HARDWARE_ROOT)/snap_check_psl "$(FPGACARD)" "$(PSL_DCP)")

#
# FIXME Targets in this Makefile are to be build in the correct order.
#      Parallelization e.g. by using -jN will not work at this point in time
#      due to missing dependencies between the different targets.
#

ifeq ($(PLATFORM),x86_64)

.PHONY: all snap_config check_snap_settings check_simulator check_nvme prepare_project snap_preprocess_start snap_preprocess_execute snap_preprocess patch_version patch_NVMe action_hw create_hls_ip create_project hw_project_start hw_project .hw_project_done config add_psl_dcp check_bsp image cloud_enable cloud_base cloud_action cloud_merge pslse software app model xsim irun nosim sim clean

all: model image

snap_config:
	@$(MAKE) -C $(SNAP_ROOT) snap_config

check_snap_settings:
	@if [ -z `which git 2> /dev/null` ]; then \
		echo "                        The SNAP framework make process relies on git."; \
		echo "                        Please make sure that you have installed git and that"; \
		echo "                        the environment variable PATH points to its executable."; \
		exit -1; \
	fi
	@cd $(SNAP_HARDWARE_ROOT); if [ -z `git describe HEAD 2> /dev/null` ]; then \
		echo "                        $(SNAP_HARDWARE_ROOT) does not belong to a git repository."; \
		echo "                        Please check out the SNAP framework as git clone from git@github.com:open-power/snap.git"; \
		exit -1; \
	fi
	@if [ ! -e "$(snap_config_sh)" ]; then \
		echo "                        Missing snap config setup file \"$(snap_config_sh)\""; \
		echo "                        Please make sure that the snap configuration is prepared by calling 'make snap_config'"; \
		exit -1; \
	fi
	@if [ ! -e "$(snap_env_sh)" ]; then \
		echo "                        Missing snap environment setup file \"$(snap_env_sh)\""; \
		echo "                        Please make sure that the snap environment is prepared by calling 'make snap_config'"; \
		exit -1; \
	fi
	@source $(SNAP_ROOT)/snap_env $(ignore_action_root);
	@echo "                        =============================="
	@echo "                        hardware/Makefile called with:"
	@echo "                        ACTION_ROOT        = $(ACTION_ROOT)"
	@if [ "$(CAPI_VER)" == "capi10" ]; then \
		echo "                        PSL_DCP            = $(PSL_DCP)"; \
	elif [ "$(CAPI_VER)" == "capi20" ]; then \
		echo "                        PSL9_IP            = $(PSL9_IP_CORE)"; \
		echo "                        FPGA ACTION CLK    = $(FPGA_ACTION_CLK)"; \
	fi
	@echo "                        FPGACARD           = $(FPGACARD)"
	@echo "                        FPGACHIP           = $(FPGACHIP)"
	@echo "                        NUM_OF_ACTIONS     = $(NUM_OF_ACTIONS)"
	@echo "                        HLS_SUPPORT        = $(HLS_SUPPORT)"
	@echo "                        SDRAM_USED         = $(SDRAM_USED)"
	@echo "                        HBM_USED           = $(HBM_USED)"
	@echo "                        BRAM_USED          = $(BRAM_USED)"
	@echo "                        NVME_USED          = $(NVME_USED)"
	@echo "                        ETHERNET_USED      = $(ETHERNET_USED)"
	@echo "                        ETH_LOOP_BACK      = $(ETH_LOOP_BACK)"
	@echo "                        ILA_DEBUG          = $(ILA_DEBUG)"
	@if [ `echo "$(ILA_DEBUG)" | tr a-z A-Z` == "TRUE" ]; then \
		echo "                        ILA_SETUP_FILE     = $(ILA_SETUP_FILE)"; \
	fi
	@echo "                        SIMULATOR          = $(SIMULATOR)"
	@echo "                        USE_PRFLOW         = $(USE_PRFLOW)"
	@if [ `echo "$(USE_PRFLOW)" | tr a-z A-Z` == "TRUE" ]; then \
		echo "                        CLOUD_USER_FLOW = $(CLOUD_USER_FLOW)"; \
		echo "                        DCP_ROOT        = $(DCP_ROOT)"; \
	fi
	@echo "                        =============================="

check_nvme:
	@if [ "$(SIMULATOR)" != "nosim" ] && [ "$(NVME_USED)" == "TRUE" ] && [ "$(DENALI_USED)" == "TRUE" ]; then \
		if [ "$(SIMULATOR)" != "irun" ] && [ "$(SIMULATOR)" != "xcelium" ]; then  \
			echo "                        NVMe simulation with Denali model is not supported for SIMULATOR $(SIMULATOR)"; \
			exit -1; \
		fi; \
		if [ !  -d "$(DENALI)" ]; then \
			echo "                        Missing or invalid environment variable for NVMe simulation with Denali."; \
			echo "                        Please check the setting of:"; \
			echo "                        DENALI:   $(DENALI)"; \
			exit -1; \
		fi \
	fi

check_simulator:
	@if [ "$(SIMULATOR)" != "irun" ] && [ "$(SIMULATOR)" != "xsim" ] && [ "$(SIMULATOR)" != "questa" ] && [ "$(SIMULATOR)" != "xcelium" ] && [ "$(SIMULATOR)" != "modelsim" ] && [ "$(SIMULATOR)" != "nosim" ]; then \
		echo "                        unknown simulator=$SIMULATOR"; \
		exit -1; \
	fi

prepare_logs:
	@mkdir -p $(LOGS_DIR);

prepare_project: check_snap_settings prepare_logs
	@echo -e "[PREPARE PROJECT.....] start `date +"%T %a %b %d %Y"`";
	@mkdir -p $(BUILD_DIR);
	@mkdir -p $(BUILD_DIR)/Reports;
	@mkdir -p $(BUILD_DIR)/Images;
	@mkdir -p $(BUILD_DIR)/Checkpoints;
	@ln -f -s $(SNAP_HDL_CORE)/psl_fpga_$(FPGA_CARD).vhd_source   $(SNAP_HDL_CORE)/psl_fpga.vhd_source;
	@ln -f -s $(SNAP_HDL_CORE)/psl_accel_$(FPGA_CARD).vhd_source  $(SNAP_HDL_CORE)/psl_accel.vhd_source;
	@ln -f -s $(SNAP_SIM_CORE)/top_$(CAPI_VER).sv_source          $(SNAP_SIM_CORE)/top.sv_source;
	@ln -f -s $(SNAP_HDL_CORE)/dma_$(CAPI_VER).vhd_source         $(SNAP_HDL_CORE)/dma.vhd;
	@ln -f -s $(SNAP_HDL_CORE)/dma_types_$(CAPI_VER).vhd_source   $(SNAP_HDL_CORE)/dma_types.vhd;
	@ln -f -s $(SNAP_HDL_CORE)/dma_buffer_$(CAPI_VER).vhd_source  $(SNAP_HDL_CORE)/dma_buffer.vhd;
	@ln -f -s $(SNAP_HDL_CORE)/dma_rams_$(CAPI_VER).vhd_source    $(SNAP_HDL_CORE)/dma_rams.vhd;
	@echo -e "[PREPARE PROJECT.....] done  `date +"%T %a %b %d %Y"`";

snap_preprocess_start: prepare_project
	@echo -e "[SNAP PREPROCESS.....] start `date +"%T %a %b %d %Y"`";
	$(MAKE) -C $(SNAP_ROOT)/hardware/sim/nvme_lite

$(SNAP_PP_FILES_VHD):
	@if [ -e "$(snap_config_cflags)" ]; then \
		echo -e "                        generating $(notdir $@)"; \
	else \
		echo -e "                        Need to run \"make snap_config\" first"; exit -1; \
	fi
	@$(CC) -E -xc $(SNAP_CFLAGS) $@_source 2> $(LOGS_DIR)/$(notdir $@)_pp.log > $@_tmp;
	@if [ $$? -ne 0 ]; then \
		$(RM) $@_tmp; \
		echo -e "                        Error: please look into $(LOGS_DIR)/$(notdir $@)_pp.log"; exit -1; \
	fi
	@grep -v "^#" $@_tmp > $@
	@$(RM) $@_tmp

$(SNAP_SIM_NVME)/endp_4_pipe32.v:
	@if [ "$(DENALI_USED)" == "TRUE" ]; then \
	  echo -e "                        generating $(notdir $@)"; \
	  $(DENALI)/bin/pureview -batch -generate all . -genoutput  $(SNAP_SIM_NVME)/endp_4_pipe32.v $(SNAP_SIM_NVME)/endp_4_pipe32.soma > $(LOGS_DIR)/pureview.log 2>&1; \
	  if [ $$? -ne 0 ]; then \
		  echo -e "                        Error: please look into $(LOGS_DIR)/pureview.log"; exit -1; \
	  fi; \
	fi

snap_core_config:
	@$(SNAP_HARDWARE_ROOT)/setup/snap_config.sh  $(SNAP_HDL_CORE)/snap_core_types.vhd

%.vhd: %.vhd_source

snap_preprocess_execute: $(SNAP_PP_FILES_VHD) $(SNAP_SIM_NVME)/endp_4_pipe32.v snap_core_config
	@echo -e "[SNAP PREPROCESS.....] done  `date +"%T %a %b %d %Y"`";

snap_preprocess: snap_preprocess_start
	@$(MAKE) -s snap_preprocess_execute

patch_version:
	@echo -e "                        patching SNAP version and build date registers"; \
	$(SNAP_HARDWARE_ROOT)/setup/patch_version.sh $(SNAP_HDL_CORE) snap_core_types.vhd

patch_NVMe:
	@if [ -e "$(SNAP_HARDWARE_ROOT)/setup/patch_NVMe.sh" ] && [ "$(NVME_USED)" == "TRUE" ]; then \
		echo -e "                        patching NVMe PCIe Root Complex sim files"; \
		cd $(SNAP_HARDWARE_ROOT)/setup && ./patch_NVMe.sh && cd .. ; \
	fi

action_hw: prepare_logs
	@echo -e "[CONFIG ACTION HW....] start `date +"%T %a %b %d %Y"`";
	@if [ ! -d "$(ACTION_ROOT)" ]; then \
		echo "                        Error: Path to ACTION_ROOT is not defined properly"; exit -1; \
	fi
	@echo "Calling make -C $(ACTION_ROOT) hw" > $(LOGS_DIR)/action_make.log
	@$(MAKE) -C $(ACTION_ROOT) hw; \
		if [ $$? -ne 0 ]; then \
			echo -e "                        Error: please look into $(LOGS_DIR)/action_make.log"; exit -1; \
		fi
	@if [ "$(HLS_SUPPORT)" == "TRUE" ]; then \
	        echo "Calling make -C $(ACTION_ROOT)/hw vhdl" >> $(LOGS_DIR)/action_make.log; \
		$(MAKE) -kC $(ACTION_ROOT)/hw vhdl >> $(LOGS_DIR)/action_make.log; \
		if [ $$? -ne 0 ]; then \
			echo -e "                        Error: please look into $(LOGS_DIR)/action_make.log"; exit -1; \
		fi; \
	fi
	@echo -e "[CONFIG ACTION HW....] done  `date +"%T %a %b %d %Y"`";

.create_ethernet_done: prepare_logs
	@echo -e "[CREATE ETHERNET IP..] start `date +"%T %a %b %d %Y"`";
	@rm -rf $(SNAP_HARDWARE_ROOT)/ip/eth_100G
	echo "Calling create_eth_100G TCL script" >> $(LOGS_DIR)/eth_100G_make.log; 
	@echo -e "			generating 100G Ethernet IP";
	@cd $(BUILD_DIR)  &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_eth_100G_ip.tcl -notrace -log $(LOGS_DIR)/eth_100G_make.log -journal $(LOGS_DIR)/eth_100G_make.jou >> $(LOGS_DIR)/eth_100G_make.log; 
	@echo -e "[CREATE ETHERNET IP..] done `date +"%T %a %b %d %Y"`";
	@touch .create_ethernet_done

.create_snap_ip_done:
	@cd $(BUILD_DIR)  &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_snap_ip.tcl -notrace -log $(LOGS_DIR)/create_snap_ip.log -journal $(LOGS_DIR)/create_snap_ip.jou
	@touch .create_snap_ip_done

.create_nvme_done:
	@echo -e "[CREATE NVME.........] start `date +"%T %a %b %d %Y"`";
	@cd $(BUILD_DIR)  &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_nvme_host.tcl -notrace -log $(LOGS_DIR)/create_nvme_host.log -journal $(LOGS_DIR)/create_nvme_host.jou
	@$(MAKE) -s patch_NVMe
	@echo -e "[CREATE NVME.........] done  `date +"%T %a %b %d %Y"`";
	@touch .create_nvme_done

# HBM part - call a different bd if simulation or HBM (HBM is not supported by xsim)
.create_hbm_done:
	@echo -e "[CREATE HBM..........] start `date +"%T %a %b %d %Y"`";
	@if [ "$(BRAM_USED)" == "TRUE" ]; then \
		cd $(BUILD_DIR)  &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_hbm_for_simu.tcl -notrace -log $(LOGS_DIR)/create_hbm.log -journal $(LOGS_DIR)/create_hbm.jou; \
	else \
		cd $(BUILD_DIR)  &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_hbm.tcl -notrace -log $(LOGS_DIR)/create_hbm.log -journal $(LOGS_DIR)/create_hbm.jou; \
	fi
	@echo -e "[CREATE HBM..........] done  `date +"%T %a %b %d %Y"`";
	@touch .create_hbm_done

create_hls_ip:
	@cd $(BUILD_DIR) &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_hls_ip.tcl -notrace -log $(LOGS_DIR)/create_hls_ip.log -journal $(LOGS_DIR)/create_hls_ip.jou

create_project: snap_preprocess
	@echo -e "[CREATE PROJECT......] start `date +"%T %a %b %d %Y"`";
	@echo -e "                        using `vivado -version |grep Vivado`"
	@$(MAKE) -s .create_snap_ip_done
	@$(MAKE) -s create_hls_ip
	@if [ "$(NVME_USED)" == "TRUE" ]; then $(MAKE) -s .create_nvme_done; fi
	@if [ "$(ETHERNET_USED)" == "TRUE" ] && [ "$(ETH_LOOP_BACK)" == "FALSE" ]; then $(MAKE) -s .create_ethernet_done; fi
	@if [ "$(HBM_USED)" == "TRUE" ]; then $(MAKE) -s .create_hbm_done; fi
	@cd $(BUILD_DIR)  &&  vivado -quiet -mode batch -source $(SETUP_DIR)/create_framework.tcl -notrace -log $(LOGS_DIR)/create_framework.log -journal $(LOGS_DIR)/create_framework.jou
	@echo -e "[CREATE PROJECT......] done  `date +"%T %a %b %d %Y"`"

hw_project_start:
	@echo -e "======================================================="
	@echo -e "== Precompiling the Action logic: `cat $(SNAP_ROOT)/snap_env.sh|grep actions|cut -f3- -d/`"
	@echo -e "======================================================="
	@echo -e "[HW PROJECT..........] start `date +"%T %a %b %d %Y"`"

hw_project: hw_project_start action_hw
	@$(MAKE) -s create_project
	@touch .hw_project_done
	@echo -e "[HW PROJECT..........] done  `date +"%T %a %b %d %Y"`"

.hw_project_done:
	$(MAKE) -s hw_project

# Adding target 'config' for backward compatibility
config: hw_project

# Adding CAPI2 board support if required
capi2-bsp/$(FPGACARD)/.create_ip_done:
	@if [ "$(CAPI_VER)" == "capi20" ]; then \
		if [ ! -e $(SNAP_HARDWARE_ROOT)/capi2-bsp/Makefile ]; then \
			cd $(SNAP_ROOT); \
			git submodule update --init --recursive hardware/capi2-bsp;  \
		fi; \
		if [ ! -e "$(PSL9_IP_DIR)/$(PSL9_IP_ARCHIVE)" ] && [ -e "$(PSL9_IP_CORE)" ]; then \
			ln -s $(PSL9_IP_CORE) $(PSL9_IP_DIR)/$(PSL9_IP_ARCHIVE); \
		fi; \
		if [ -e "$(PSL9_IP_DIR)/$(PSL9_IP_ARCHIVE)" ]; then \
			$(MAKE) -s -C $(SNAP_HARDWARE_ROOT)/capi2-bsp $(FPGACARD);  \
		fi \
	fi

# checking card type for PSL design checkpoint
check_bsp: capi2-bsp/$(FPGACARD)/.create_ip_done
	@if [ "$(IGNORE_CHECK_PSL)" != "TRUE" ]; then \
		if [ "$(CAPI_VER)" == "capi10" ] && [ "$(PSL_DCP_TYPE)" != "$(FPGACARD)" ]; then \
			if [[ "$(PSL_DCP_TYPE)" == *"OUTDATED"* ]]; then \
				echo "### ERROR ### PSL_DCP for $(FPGACARD) is pointing to an outdated design checkpoint."; exit -1; \
			elif [ "$(PSL_DCP_TYPE)" == "" ]; then \
				echo "### ERROR ###  PSL_DCP must point to the CAPI PSL Checkpoint file (b_route_design.dcp)."; \
				echo "       Please refer to https://github.com/open-power/snap/tree/master/hardware/README.md#power8"; exit -1; \
			else  \
				echo "### ERROR ### PSL_DCP is pointing to a checkpoint for a $(PSL_DCP_TYPE) card while FPGACARD is set to $(FPGACARD)."; exit -1; \
			fi; \
		elif [ "$(CAPI_VER)" == "capi20" ]; then \
		    if [ ! -e "$(SNAP_HARDWARE_ROOT)/capi2-bsp/$(FPGACARD)/build/ip/capi_bsp_wrap.xcix" ] && \
		       [ ! -e "$(SNAP_HARDWARE_ROOT)/capi2-bsp/$(FPGACARD)/build/ip/capi_bsp_wrap/capi_bsp_wrap.xci" ]; then \
		        echo "### ERROR ### CAPI board support IP for $(FPGACARD) is not prepared."; \
		        echo "       This is a requirement for image build."; \
		        if [ ! -e  "$(PSL9_IP_DIR)/$(PSL9_IP_ARCHIVE)" ] && [ ! -e "$(PSL9_IP_CORE)" ]; then \
		            echo "       Path to $(PSL9_IP_ARCHIVE) file cannot be found."; \
		            echo "       Please set or check PSL9_IP_CORE variable in snap_env.sh"; \
		            echo "       points to the right path or make sure this zip file is in"; \
		            echo "         $(PSL9_IP_DIR)"; \
		            echo "       This zip file is mandatory for building an image/FPGA with CAPI2.0."; \
		            echo "       All information where to get this zip file can be found"; \
		            echo "       from https://github.com/open-power/snap/tree/master/hardware/README.md#power9"; \
		        fi; \
		        exit -1; \
		    elif [ -e "$(SNAP_HARDWARE_ROOT)/capi2-bsp/$(FPGACARD)/.action_clk_225MHZ" ] \
                          && [ $(FPGA_ACTION_CLK) == "250MHZ" ]; then                    \
			echo "PSL for $(FPGACARD) card is being recreated for 250MHZ Action clock (Last compilation done with 225MHZ)";         \
			$(MAKE) -s -C $(SNAP_HARDWARE_ROOT)/capi2-bsp clean $(FPGACARD); \
		    elif [ -e "$(SNAP_HARDWARE_ROOT)/capi2-bsp/$(FPGACARD)/.action_clk_250MHZ" ] \
                          && [ $(FPGA_ACTION_CLK) == "225MHZ" ]; then                    \
			echo "PSL for $(FPGACARD) card is being recreated for 225MHZ Action clock (Last compilation done with 250MHZ)";         \
			$(MAKE) -s -C $(SNAP_HARDWARE_ROOT)/capi2-bsp clean $(FPGACARD); \
		    elif [ ! -e "$(SNAP_HARDWARE_ROOT)/capi2-bsp/$(FPGACARD)/.action_clk_225MHZ" ] \
		      && [ ! -e "$(SNAP_HARDWARE_ROOT)/capi2-bsp/$(FPGACARD)/.action_clk_250MHZ" ]; then  \
			echo "PSL for $(FPGACARD) card is being (re)created (No logs states with which FPGA clock BSP was compiled earlier)";         \
			$(MAKE) -s -C $(SNAP_HARDWARE_ROOT)/capi2-bsp clean $(FPGACARD); \
		    fi; \
		fi; \
	fi

image: check_bsp
	$(MAKE) .hw_project_done
	@echo -e "[BUILD IMAGE.........] start `date +"%T %a %b %d %Y"`\n"
ifeq ($(USE_PRFLOW),TRUE)
	snap-cloud-build ${SNAP_ROOT}/snap_env.sh
else
	@echo "A complete FPGA bitstream build got kicked off.";
	@echo "This might take more than an hour depending on the machine used";
	@echo "The process may be terminated by pressing <CTRL>-C at any time.";
	@echo "After termination it can be restarted later."; echo
	@$(MAKE) -s patch_version
	@cd $(BUILD_DIR) && vivado -quiet -mode batch -source $(SETUP_DIR)/snap_build.tcl -notrace -log $(LOGS_DIR)/snap_build.log -journal $(LOGS_DIR)/snap_build.jou
	@$(RM) -r .bitstream_name.txt
endif
	@echo -e "[BUILD IMAGE.........] done  `date +"%T %a %b %d %Y"`"

allow_%:
	@if [ "$(USE_PRFLOW)" != "TRUE" ]; then \
		echo; echo "Error: Makefile target $* is only allowed for PR flow!"; exit -1; \
	elif [ "$(CLOUD_USER_FLOW)" == "TRUE" ] && [ "$*" == "cloud_base" ]; then \
		echo; echo "Error: Makefile target $* is not allowed for cloud user flow!"; exit -1; \
	fi

cloud_enable: allow_cloud_action
	@echo -e "[ENABLE CLOUD FLOW...] start `date +"%T %a %b %d %Y"e`"
	@$(MAKE) -s patch_version
	@cd $(BUILD_DIR) && env CLOUD_RUN= vivado -quiet -mode batch -source $(SETUP_DIR)/snap_cloud_build.tcl -notrace -log $(LOGS_DIR)/cloud_action.log -journal $(LOGS_DIR)/cloud_action.jou
	@echo -e "[ENABLE CLOUD FLOW...] done  `date +"%T %a %b %d %Y"`"

cloud_base: allow_cloud_base .hw_project_done
	@echo -e "[BUILD CLOUD BASE....] start `date +"%T %a %b %d %Y"`"
	@$(MAKE) -s patch_version
	@cd $(BUILD_DIR) && env CLOUD_RUN=BASE vivado -quiet -mode batch -source $(SETUP_DIR)/snap_cloud_build.tcl -notrace -log $(LOGS_DIR)/snap_cloud_build.log -journal $(LOGS_DIR)/snap_cloud_build.jou
	@echo -e "[BUILD CLOUD BASE....] done  `date +"%T %a %b %d %Y"`"

$(SNAP_ACTION_DCP):
	@echo -e "                        Need to run cloud_action first"
	@exit -1

$(SNAP_BASE_DCP):
	@echo -e "                        Need to run cloud_base first"
	@exit -1

cloud_action: allow_cloud_action .hw_project_done
	@echo -e "[CLOUD ACTION SYNTH..] start `date +"%T %a %b %d %Y"e`"
	@$(MAKE) -s patch_version
	@echo -e "                        using `vivado -version |grep Vivado`"
	@cd $(BUILD_DIR) && env CLOUD_RUN=ACTION vivado -quiet -mode batch -source $(SETUP_DIR)/snap_cloud_build.tcl -notrace -log $(LOGS_DIR)/cloud_action.log -journal $(LOGS_DIR)/cloud_action.jou
	@echo -e "[CLOUD ACTION SYNTH..] done  `date +"%T %a %b %d %Y"`"

cloud_merge: allow_cloud_merge $(SNAP_BASE_DCP) $(SNAP_ACTION_DCP)
	@ignore_action_root=ignore_action_root $(MAKE) prepare_project
	@echo -e "[CLOUD MERGE ........] start `date +"%T %a %b %d %Y"`"
	@echo -e "                        using `vivado -version |grep Vivado`"
	@cd $(BUILD_DIR) && vivado -quiet -mode batch -source $(SETUP_DIR)/snap_cloud_merge.tcl -notrace -log $(LOGS_DIR)/snap_cloud_merge.log -journal $(LOGS_DIR)/snap_cloud_merge.jou
	@echo -e "[CLOUD MERGE ........] done  `date +"%T %a %b %d %Y"`"

# Add DEBUG=1 (before -C) in make pslse line to get more details from PSLSE
pslse: prepare_logs
	@echo -e "[COMPILE PSLSE ......] start `date +"%T %a %b %d %Y"`"
	@$(SNAP_ROOT)/check_pslse.sh > $(LOGS_DIR)/compile_pslse.log 2>&1
#        @$(MAKE) DEBUG=1 -C $(PSLSE_ROOT) >> $(LOGS_DIR)/compile_pslse.log 2>&1; \
	@$(MAKE) -C $(PSLSE_ROOT) >> $(LOGS_DIR)/compile_pslse.log 2>&1; \
	if [ $$? -ne 0 ]; then echo -e "                        Error: please look into $(LOGS_DIR)/compile_pslse.log"; exit -1; fi
	@echo -e "[COMPILE PSLSE ......] done  `date +"%T %a %b %d %Y"`"

software: pslse
	@echo -e "[COMPILE SOFTWARE....] start `date +"%T %a %b %d %Y"`"
	@$(MAKE) -C $(SNAP_ROOT)/software       > $(LOGS_DIR)/compile_sw.log 2>&1; \
	if [ $$? -ne 0 ]; then \
		echo -e "                        Error: please look into $(LOGS_DIR)/compile_sw.log"; exit -1; \
	fi
	@echo -e "[COMPILE SOFTWARE....] done  `date +"%T %a %b %d %Y"`"

app: software
	@echo -e "[COMPILE APPLICATION.] start `date +"%T %a %b %d %Y"`"
	@$(MAKE) -C $(ACTION_ROOT)/sw        >> $(LOGS_DIR)/compile_sw.log 2>&1; \
	if [ $$? -ne 0 ]; then \
		echo -e "                        Error: please look into $(LOGS_DIR)/compile_sw.log"; exit -1; \
	fi
	@echo -e "[COMPILE APPLICATION.] done  `date +"%T %a %b %d %Y"`"

nosim:
	@echo -e "[BUILD $@ MODEL....] start `date +"%T %a %b %d %Y"`"
	@echo -e "                        Info: Not building a simulation model, since SIMULATOR is set to \"nosim\"";
	@echo -e "[BUILD $@ MODEL....] done  `date +"%T %a %b %d %Y"`"
	@echo "-----------"
	@echo "  Suggested next step: to run a simulation, execute: make sim"

$(SNAP_SIMULATORS): check_nvme .hw_project_done app
	@if [ "$(SIMULATOR)" != "$@" ]; then \
		echo; echo "Error: Makefile target $@ called with SIMULATOR set to \"$(SIMULATOR)\""; \
		exit -1; \
	fi
	@echo -e "[BUILD $@ MODEL....] start `date +"%T %a %b %d %Y"`"
	@$(MAKE) -s patch_version
	@cd $(SIM_DIR) && vivado -quiet -mode batch -source $(SETUP_DIR)/export_$@.tcl -notrace -log $(LOGS_DIR)/compile_$@.log -journal $(LOGS_DIR)/compile_$@.jou
	@$(SETUP_DIR)/patch_sim.sh $(SIM_DIR)/$(subst irun,ies,$@) top.sh >> $(LOGS_DIR)/compile_$@.log 2>&1
	@cd $(SIM_DIR)/$(subst irun,ies,$@) && ln -sf $(PSLSE_ROOT)/afu_driver/src/libdpi.so;
	@echo -e "                        build $@ model";
	@cd $(SIM_DIR)/$(subst irun,ies,$@); \
		./top.sh >> $(LOGS_DIR)/compile_$@.log 2>&1; \
		if [ $$? -ne 0 ]; then \
			echo -e "                        Error: please look into $(LOGS_DIR)/compile_$@.log"; exit -1; \
		fi
	@touch .model_$@
	@echo -e "[BUILD $@ MODEL....] done  `date +"%T %a %b %d %Y"`"
	@echo "-----------"
	@echo "  Suggested next step: to run a simulation, execute: make sim"

$(SNAP_MODELS):
	@$(MAKE) -s $(subst .model_,,$@)

model: check_simulator
	@$(MAKE) -s $(SIMULATOR)

sim: check_simulator
	@$(MAKE) -s .model_$(SIMULATOR)
	@echo -e "[SIMULATION........] start `date +"%T %a %b %d %Y"`"
	@echo "                        SIMULATOR is set to $(SIMULATOR)"
	@if [ "$(SIMULATOR)" != "nosim" ]; then cd sim; ./run_sim; fi
	@echo -e "[SIMULATION........] done  `date +"%T %a %b %d %Y"`"
	@echo "-----------"
	@echo "  Suggested next step: to display waveforms, execute: ./snap_trace"

sim_tmux:
	@tmux new-session "$(MAKE) -s sim"

else #noteq ($(PLATFORM),x86_64)
.PHONY: wrong_platform all model sim image

wrong_platform:
	@echo; echo "\nSNAP hardware builds are possible on x86 platform only\n"; echo;

all model sim image: wrong_platform
endif

.PHONY: clean
clean:
	@echo -e "[CLEAN ENVIRONMENT...] start `date +"%T %a %b %d %Y"`"
	@$(RM)    .hw_project_done .create_snap_ip_done .create_nvme_done .create_ethernet_done .create_hbm_done
	@$(RM) -r $(SNAP_PP_FILES_VHD)                   \
	          $(SNAP_SIM_CORE)/top.sv_source         \
	          $(SNAP_SIM_NVME)/endp_4_pipe32.v       \
	          $(SNAP_TMP_FILES)                      \
	          $(SNAP_HDL_CORE)/psl_fpga.vhd_source   \
	          $(SNAP_HDL_CORE)/psl_accel.vhd_source  \
	          $(SNAP_HDL_CORE)/dma.vhd               \
	          $(SNAP_HDL_CORE)/dma_types.vhd         \
	          $(SNAP_HDL_CORE)/dma_buffer.vhd        \
	          $(SNAP_HDL_CORE)/dma_rams.vhd          \
	          $(SNAP_MODELS)                         \
	          *build
	@echo -e "                        vivado project";
	@$(RM) -r viv_project*
	@echo -e "                        IPs";
	@$(RM) -r ip
	@echo -e "                        sim files";
	@$(RM) -r sim/ies sim/xsim sim/modelsim sim/questa sim/xcelium hdl/nvme/component.xml hdl/nvme/xgui
	@echo -e "                        log files";
	@$(RM) -r logs
	@echo -e "                        action / application";
	@if [ -e "$(ACTION_ROOT)/Makefile" ]; then	    \
		$(MAKE) -C $(ACTION_ROOT) $@ > /dev/null;   \
		if [ $$? -ne 0 ]; then                      \
			echo -e "                        Error: [make "$@"] failed for action/application in $(ACTION_ROOT)"; exit -1; \
		fi                                          \
	fi
	$(MAKE) -C $(SNAP_ROOT)/hardware/sim/nvme_lite $@
	@echo -e "[CLEAN ENVIRONMENT...] done  `date +"%T %a %b %d %Y"`"
