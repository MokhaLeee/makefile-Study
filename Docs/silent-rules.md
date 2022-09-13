# u-boot 静默编译原理

​	正常情况下，对 c 语言做编译所需的 makefile 大抵书写如下：

```makefile
%o: %c
	aarch64-linux-gnu-gcc -c -o $@ $<

main.elf: main.o lib.o
	aarch64-linux-gnu-gcc -o main.elf main.o lib.o
```

​	在 shell 中输入指令 `make main.elf` 获得输出如下：

```shell
> make main.elf
aarch64-linux-gnu-gcc -c -o main.o main.c
aarch64-linux-gnu-gcc -c -o lib.o lib.c
aarch64-linux-gnu-gcc -o main.elf main.o lib.o
```

​	然而在编译 u-boot 以及 kernel 时，大抵获得如下输出：

```shell
> make main.elf V=0
CC		main.o
CC		lib.o
CC		main.elf
```

​	今天通过对 u-boot makefile 的分析，尝试阐述一下该种输出的 makefile 编写原理。



## 一、顶层配置

​	对 u-boot 的静默编译配置由 make 时添加附加指令 `V` 来实现，如果配置 `V=1` 则按照复杂情况进行输出，反之则输出静默版本。顶层 makefile 通过如下代码实现：

```shell
# u-boot/makefile
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
```

​	顶层通过 `origin` 函数来判定输入参数中是否有参数 `V`，这样一来也可以直接在 makefile 中配置缺省值。此处需要关注参数 `quiet`，若传参 `V=1`，则配置 `quiet=`，否则配置 `quiet=quiet_`。

​	传统的教程至此就结束了，他们会告诉你需要在 shell 中编译 code 的时候添加额外参数 `V=1` 用来查看编译过程，但是却不会告诉你具体的实现过程。

​	接下来，我们会逐步对 makefile 中编译 C 的过程进行细化，以最终实现静默输出。



## 二、原理解释

​	首先，我们使用 `call` 函数来替换对 C 的直接编译：

```makefile
cmd_cc_o_c = aarch64-linux-gnu-gcc -c -o $(2) $(1)

%o: %c
	$(call, cmd_cc_o_c, $<, $@)
```

​	`call` 函数会将变量 `cmd_cc_o_c` 的值作为指令进行运行，在其中进行传参，`$(1) = $<`，`$(2) = $@`。此处在 shell 中编译 `main.o`，得到与原本相同的结果：

```shell
> make main.o
aarch64-linux-gnu-gcc -c -o main.o main.c
```

​	接下来，构建一个函数 `cmd`：

```makefile
cmd = $(cmd_$(1))
cmd_cc_o_c = aarch64-linux-gnu-gcc -c -o $@ $<

%o: %c
	$(call cmd,cc_o_c)
```

​	在上述 makefile 中，调用`$(call cmd,cc_o_c)` 后，`cmd` 的值变为：`cmd_cc_o_c`，正对应上述函数，从而获得执行对 C 的编译工作。
​	接下来，从 `cmd` 的位置开始施展魔法：


```makefile
cmd = @$(if $($(quiet)cmd_$(1)),\
	echo '$($(quiet)cmd_$(1))' &&) $(cmd_$(1))

quiet_cmd_cc_o_c = CC		$@
cmd_cc_o_c = aarch64-linux-gnu-gcc -c -o $@ $<

%o: %c
	$(call cmd,cc_o_c)
```

​	在 makefile 调用 `$(call cmd,cc_o_c)` 后，此时 `cmd` 的值替换为：

```makefile
@$(if $($(quiet)cmd_cc_o_c), 				\
	echo '$($(quiet)cmd_cc_o_c)' &&)		\
$(cmd_cc_o_c)
```

​	此时 `call` 函数实际调用了两个函数：

- `if` 函数调用 `echo '$(quiet)cmd_cc_o_c'` ，实现静默输出
- 静默调用 `$(cmd_cc_o_c)`，即实质编译 C 的工作



​	通过对 `V` 的传参，可以实现对 `quiet` 的配置。若 `quiet=quiet_`，则 `if` 判定的值是 `quiet_cmd_cc_o_c`，正是对应于 `CC		$@`。否则，`if` 判定 `cmd_cc_o_c`，输出完整编译过程。

## 三、makefile 编写

​	在了解了静默输出原理后，可将静默编译过程进一步扩展，最终完成 makefile 书写如下：

```makefile
# config silent-rule
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

# tools
profix := aarch64-linux-gnu-
CC := $(profix)gcc
AS := $(profix)as

cmd = @$(if $($(quiet)cmd_$(1)),\
	echo '$($(quiet)cmd_$(1))' &&) $(cmd_$(1))

# compile
quiet_cmd_cc_asm_c = CC		$@
	cmd_cc_asm_c = $(CC) $(c_flags) -S $< -o $@
quiet_cmd_as_o_S = AS		$@
	cmd_as_o_S = $(AS) $(c_flags) $< -o $@
quiet_cmd_cc_o_c = CC		$@
	cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<

%.asm: %.c
	$(call cmd,S)
%.o: %.c
	$(call cmd,cc_o_c)
%.o: %.S
	$(call cmd,as_o_S)

# target
quiet_cmd_main_elf = CC		$@
	cmd_main_elf = $(CC) -static -o $@ main.o

main.elf: main.o
	$(call cmd,main_elf)

# files
CC_FILES := $(shell find -type f -name '*.c')
AS_FILES := $(shell find -type f -name '*.s')
CC_GEN_FILES := $(patsubst %.c, %.o, $(CC_FILES))
CC_GEN_FILES += $(patsubst %.c, %.asm, $(CC_FILES))
AS_GEN_FILES := $(patsubst %.s, %.o, $(AS_FILES))

# clean
CLEAN_FILES :=
CLEAN_FILES += $(CC_GEN_FILES)
CLEAN_FILES += $(AS_GEN_FILES)

clean:
	@rm -f $(CLEAN_FILES)
	@echo 'All cleaned!'
.PHONY: clean

```

​	基于上述 makefile，编译 main.elf：

```shell
> make main.elf
CC		main.o
CC		lib.o
CC		main.elf
```







