# Copyright (c) 2026 Eclipse Foundation
# SPDX-License-Identifier:Apache-2.0

# TODO: seed management

# Run recipes with bash + pipefail so `cmd | tee` surfaces cmd's exit status
# (rather than tee's), and run_all aborts on the first failed test.
SHELL := /bin/bash
.SHELLFLAGS := -o pipefail -c

# Variables
BANNER     = "*******************************************************************************"
VERILATOR  = verilator
COV_TOOL   = verilator_coverage
TOP_MODULE = uvm_pkg
SV_SOURCES = ../sv/pipe_pkg.sv \
             ../sv/pipe_if.sv \
             ../rtl/pipe.v \
             ../tb/top.sv

# UVM variables
# Accelera's UVM library fetched with `wget https://www.accellera.org/images/downloads/standards/uvm/Accellera-1800.2-2017-1.0.tar.gz`
# Use ?= so the container image can inject UVM_HOME via its environment.
UVM_HOME       ?= /opt/accellera/1800.2-2017-1.0/src
UVM_PKG        = $(UVM_HOME)/uvm_pkg.sv
UVM_TESTNAME  ?= data0_test
ALL_TESTNAMES ?= data0_test data1_test random_test many_random_test

# Output Directories
OBJ_DIR  = obj_dir
LOG_DIR  = logs
CMP_LOG  = $(LOG_DIR)/verilator_compile.log
COV_DIR  = $(LOG_DIR)/coverage
COV_FILE = $(COV_DIR)/$(UVM_TESTNAME).cov

# Verilator Flags
# --coverage means all of --coverage-line, --coverage-toggle, and --coverage-user
VERILATOR_FLAGS = --main \
                  --binary \
                  --build \
                  --cc \
                  --Mdir $(OBJ_DIR) \
                  --error-limit 5 \
                  --hierarchical \
                  --coverage \
                  -j 4 \
                  -Wall \
                  --timescale 1ns/1ns

# Deliberately disabled warnings (all from the UVM class library)
# TODO: eliminate these one-by-one.
DISABLED_WARNINGS= -Wno-DECLFILENAME \
                   -Wno-CONSTRAINTIGN \
                   -Wno-MISINDENT \
                   -Wno-VARHIDDEN \
                   -Wno-WIDTHTRUNC \
                   -Wno-CASTCONST \
                   -Wno-WIDTHEXPAND \
                   -Wno-UNDRIVEN \
                   -Wno-UNUSEDSIGNAL \
                   -Wno-UNUSEDPARAM \
                   -Wno-SYMRSVDWORD \
                   -Wno-ZERODLY \
                   -Wno-CASEINCOMPLETE \
                   -Wno-SIDEEFFECT \
                   -Wno-fatal \
                   -Wno-REALCVT

INCDIRS = +incdir+"$(UVM_HOME)" \
          +incdir+../rtl \
          +incdir+../sv \
          +incdir+../tb

DEFINES = +define+UVM_REPORT_DISABLE_FILE_LINE \
          +define+UVM_NO_DPI \
          +define+SVA_ON

# --- iSim (True North Silicon) ---------------------------------------------
# iSim is a SystemVerilog + VHDL simulator that SHIPS a pre-compiled UVM:
# `-uvm [version]` maps the library out of the install's own
# lib/isim/uvm/<version>/ into the work library and resolves uvm_macros.svh
# from the shipped flattened header.  So the iSim targets below drop EXACTLY
# TWO things from what the Verilator line compiles, and they are the same one
# thing: $(UVM_PKG) and +incdir+"$(UVM_HOME)".  No UVM source reaches an iSim
# command line and UVM_HOME is never read.
#
# EVERYTHING ELSE IS READ FROM THE VARIABLES ABOVE, NOT RETYPED -- the same
# $(SV_SOURCES), the same $(INCDIRS) minus the UVM entry, the same $(DEFINES)
# including +define+UVM_NO_DPI.  That define is a Verilator capability
# workaround and is inert under iSim (it appears nowhere in the macro headers
# that expand inside this compilation unit), but it is upstream's define and
# these targets do not get to decide which of upstream's defines they like.
# If a source, an include path or a define is added above, these targets pick
# it up unchanged.
ISIM_VLOG ?= isim-vlog
ISIM_ELAB ?= isim-elab
ISIM      ?= isim
# TOP_MODULE above is `uvm_pkg`, which names the C++ class Verilator emits for
# its --main wrapper -- it is NOT the HDL top.  The elaboration top is `top`,
# in tb/top.sv.
ISIM_TOP  ?= top
ISIM_WORK ?= isim_work
ISIM_LOG   = $(LOG_DIR)/isim_compile.log

# $(INCDIRS) minus the UVM one.  filter-out matches on the EXPANDED token, so
# this keeps working if UVM_HOME is redefined or injected from the container.
ISIM_INCDIRS = $(filter-out +incdir+"$(UVM_HOME)",$(INCDIRS))

# The timescale is SCRAPED from VERILATOR_FLAGS rather than repeated, because
# a timescale changes what `#5` means and a second copy would drift.
ISIM_TIMESCALE := $(shell printf '%s\n' $(VERILATOR_FLAGS) | grep -A1 -x -e '--timescale' | tail -n1)
ifneq ($(filter isim isim_analyse isim_compile isim_run_one isim_run_all isim_one isim_all,$(MAKECMDGOALS)),)
ifeq ($(strip $(ISIM_TIMESCALE)),)
$(error VERILATOR_FLAGS no longer carries --timescale -- every timestamp a run \
        produces depends on it, so the iSim targets refuse to guess one)
endif
endif

# --- Targets ---

.PHONY: all clean run report epilog
.PHONY: isim isim_analyse isim_compile isim_run_one isim_run_all isim_one isim_all isim_clean isim_epilog

targets:
	@echo "Available targets..."
	@echo "  $(OBJ_DIR)/V$(TOP_MODULE): compile and build"
	@echo "  run: run UVM test $(UVM_TESTNAME) (depends on $(OBJ_DIR)/V$(TOP_MODULE))."
	@echo "  cov_report: generate a coverage report (depends on run)."
	@echo "  epilog: print location of objs, logs and coverage files (no dependents)."
	@echo "  clean: delete all generated files."
	@echo "  one: clean, run_one, cov_report and epilog (in that order)."
	@echo "  all: clean, run_all, cov_report and epilog (in that order)."
	@echo "  --- iSim (True North Silicon) ---"
	@echo "  isim_analyse: analyse with iSim's shipped UVM (-uvm)."
	@echo "  isim_compile: analyse + elaborate with iSim's shipped UVM (-uvm)."
	@echo "  isim_run_one: run UVM test $(UVM_TESTNAME) under iSim."
	@echo "  isim_run_all: run every test in ALL_TESTNAMES under iSim."
	@echo "  isim_one / isim_all: isim_clean, compile, run_one/run_all, epilog."
	@echo "  isim_clean: delete iSim generated files."

one: clean $(OBJ_DIR)/V$(TOP_MODULE) run_one cov_report epilog
all: clean $(OBJ_DIR)/V$(TOP_MODULE) run_all cov_report epilog

# Compile and Build
$(OBJ_DIR)/V$(TOP_MODULE): $(SV_SOURCES)
	@echo $(BANNER)
	@echo "Compiling..."
	@echo $(BANNER)
	@mkdir -p $(LOG_DIR)
	$(VERILATOR) $(VERILATOR_FLAGS) $(DISABLED_WARNINGS) $(DEFINES) $(INCDIRS) $(UVM_PKG) $(SV_SOURCES) 2>&1 | tee $(CMP_LOG)

# Run one testcase (as determined by $UVM_TESTNAME)
run_one: $(OBJ_DIR)/V$(TOP_MODULE)
	@echo $(BANNER)
	@echo "Running $(UVM_TESTNAME)..."
	@echo $(BANNER)
	@mkdir -p $(COV_DIR)
	./$(OBJ_DIR)/V$(TOP_MODULE) \
		+UVM_TESTNAME="$(UVM_TESTNAME)" \
		+verilator+coverage+file+$(COV_DIR)/$(UVM_TESTNAME).cov \
		2>&1 | tee $(LOG_DIR)/$(UVM_TESTNAME).log

## Run all testcases
run_all: $(OBJ_DIR)/V$(TOP_MODULE)
	@echo $(BANNER)
	@echo "Running _all_ testcases..."
	@echo $(BANNER)
	@mkdir -p $(COV_DIR)
	@set -e; for testname in $(ALL_TESTNAMES); do \
		echo "Creating $$testname..."; \
		./$(OBJ_DIR)/V$(TOP_MODULE) \
			+UVM_TESTNAME="$$testname" \
			+verilator+coverage+file+$(COV_DIR)/$$testname.cov \
			2>&1 | tee $(LOG_DIR)/$$testname.log; \
	done

# Generate Coverage Report
# Annotate creates a copy of the source code with coverage markers
# This feature is not yet fully supported by Verilator.
cov_report: run
	@echo $(BANNER)
	@echo "Generating coverage report..."
	@echo $(BANNER)
	$(COV_TOOL) --annotate $(LOG_DIR)/annotated_src $(COV_DIR)/*.cov
	$(COV_TOOL) --rank $(COV_DIR)/*.cov > $(LOG_DIR)/coverage_ranking.txt

epilog:
	@echo $(BANNER)
	@echo "Compilation logfiles in $(CMP_LOG)"
	@echo "Simulation logfiles in $(LOG_DIR)/<UVM_TESTNAME>.log"
	@echo "Annotated coverage reports in $(LOG_DIR)/annotated_src"
	@echo "Coverage ranking report in $(LOG_DIR)/coverage_ranking.txt"
	@echo $(BANNER)

clean:
	@echo $(BANNER)
	@echo "Delete _all_ generated files..."
	@echo $(BANNER)
	rm -rf $(OBJ_DIR) $(LOG_DIR)

# --- iSim targets ----------------------------------------------------------
# Mirrors the Verilator ladder above: compile+elaborate, run one, run all.
#
# WHAT THESE DO TODAY, MEASURED 2026-09-02: `isim_compile` ANALYSES the whole
# environment clean and then isim-elab refuses ONE call -- tb/test_lib.sv:33's
# `printer = new("my_printer")`.  uvm_table_printer gained a name argument in
# 1800.2-2017; in UVM 1.2, the version iSim ships, its constructor takes none.
# That is a LIBRARY VERSION difference, not a simulator defect, and the oracle
# agrees: dsim renders the identical refusal from its own UVM 1.2 tree and
# runs this bench clean from its 1800.2 tree.  The rung is tracked upstream of
# here as True North Silicon github#1592 (ship an 1800.2 library); until then
# `isim_compile` is EXPECTED to stop at that call, loudly and by name.  It is
# committed anyway because a build surface that names the flow is how the day
# it starts passing gets noticed.

# `isim_analyse` is the analysis half on its own (the compilation rung of a
# ladder that grades analyse / elaborate / run separately); `isim_compile`
# is analyse + elaborate, as before.
isim_analyse: $(SV_SOURCES)
	@echo $(BANNER)
	@echo "Analysing with iSim (shipped UVM, -uvm)..."
	@echo $(BANNER)
	@mkdir -p $(LOG_DIR)
	$(ISIM_VLOG) -work $(ISIM_WORK) -uvm -timescale $(ISIM_TIMESCALE) \
		$(ISIM_INCDIRS) $(DEFINES) $(SV_SOURCES) 2>&1 | tee $(ISIM_LOG)

isim_compile: isim_analyse
	@echo $(BANNER)
	@echo "Elaborating with iSim..."
	@echo $(BANNER)
	$(ISIM_ELAB) -work $(ISIM_WORK) -uvm -timescale $(ISIM_TIMESCALE) \
		-sir -top $(ISIM_TOP) 2>&1 | tee -a $(ISIM_LOG)

# `isim` is the short spelling of the compile+elaborate rung.
isim: isim_compile

# Run one testcase (as determined by $UVM_TESTNAME)
isim_run_one: isim_compile
	@echo $(BANNER)
	@echo "Running $(UVM_TESTNAME) under iSim..."
	@echo $(BANNER)
	$(ISIM) -work $(ISIM_WORK) -image $(ISIM_WORK)/a.out.so \
		+UVM_TESTNAME="$(UVM_TESTNAME)" \
		2>&1 | tee $(LOG_DIR)/isim_$(UVM_TESTNAME).log

# Run all testcases
isim_run_all: isim_compile
	@echo $(BANNER)
	@echo "Running _all_ testcases under iSim..."
	@echo $(BANNER)
	@set -e; for testname in $(ALL_TESTNAMES); do \
		echo "Running $$testname..."; \
		$(ISIM) -work $(ISIM_WORK) -image $(ISIM_WORK)/a.out.so \
			+UVM_TESTNAME="$$testname" \
			2>&1 | tee $(LOG_DIR)/isim_$$testname.log; \
	done

isim_one: isim_clean isim_compile isim_run_one isim_epilog
isim_all: isim_clean isim_compile isim_run_all isim_epilog

isim_epilog:
	@echo $(BANNER)
	@echo "iSim compile log in $(ISIM_LOG)"
	@echo "iSim simulation logfiles in $(LOG_DIR)/isim_<UVM_TESTNAME>.log"
	@echo "iSim work library in $(ISIM_WORK)"
	@echo $(BANNER)

isim_clean:
	@echo $(BANNER)
	@echo "Delete iSim generated files..."
	@echo $(BANNER)
	rm -rf $(ISIM_WORK) $(LOG_DIR)/isim_*.log $(ISIM_LOG)
