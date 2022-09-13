.PHONY: test all clean

test: main.elf


# config V

ifeq ("$(origin V)", "command line")
  KBUILD_VERBOSE = $(V)
endif
ifndef KBUILD_VERBOSE
  KBUILD_VERBOSE = 0
endif

ifeq ($(KBUILD_VERBOSE),1)
  quiet =
  Q =
else
  quiet=quiet_
  Q = @
endif



cmd = @$(if $($(quiet)cmd_$(1)),\
	echo '$($(quiet)cmd_$(1))' &&) $(cmd_$(1))

CLEAN_FILES :=

profix := aarch64-linux-gnu-
CC := $(profix)gcc
AS := $(profix)as

c_flags  := -O1

%.asm: %.c
	$(call cmd,S)
%.o: %.c
	$(call cmd,cc_o_c)
%.o: %.S
	$(call cmd,as_o_S)



# in scripts/Makefile.build
quiet_cmd_cc_asm_c = CC		$@
	cmd_cc_asm_c = $(CC) $(c_flags) -S $< -o $@

quiet_cmd_as_o_S = AS		$@
	cmd_as_o_S = $(AS) $(c_flags) $< -o $@

quiet_cmd_cc_o_c = CC		$@
	cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<

quiet_cmd_main_elf = CC		$@
	cmd_main_elf = $(CC) -o $@ main.o lib.o

CC_FILES := $(shell find -type f -name '*.c')
AS_FILES := $(shell find -type f -name '*.s')

CC_GEN_FILES := $(patsubst %.c, %.o, $(CC_FILES))
CC_GEN_FILES += $(patsubst %.c, %.asm, $(CC_FILES))
AS_GEN_FILES := $(patsubst %.s, %.o, $(AS_FILES))

main.elf: main.o lib.o
	$(call cmd,main_elf)

all: $(CC_GEN_FILES) $(AS_GEN_FILES) main.elf

CLEAN_FILES += $(CC_GEN_FILES) $(AS_GEN_FILES)
CLEAN_FILES += main.elf

clean:
	@rm -f $(CLEAN_FILES)
	@echo all cleaned!
