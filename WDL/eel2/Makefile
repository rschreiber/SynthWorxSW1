CC=gcc
CFLAGS=-g -DWDL_FFT_REALSIZE=8  -Wall -Wno-unused-function -Wno-multichar -Wno-unused-result -Wshadow -Wtype-limits
LFLAGS=
CXX=g++

ifdef DEBUG
CFLAGS += -D_DEBUG -O0 -DWDL_CHECK_FOR_NON_UTF8_FOPEN
else
CFLAGS += -DNDEBUG -O
endif

CFLAGS += -D_FILE_OFFSET_BITS=64

OBJS=nseel-caltab.o nseel-compiler.o nseel-eval.o nseel-lextab.o nseel-ram.o nseel-yylex.o nseel-cfunc.o fft.o

SWELL_OBJS=
LICE_OBJS=

OBJS2=

UNAME_S := $(shell uname -s)
ARCH := $(shell uname -m)

ifeq ($(ARCH), aarch64)
  ifeq ($(shell $(CC) -dumpmachine | cut -f 1 -d -), arm)
    # helper for armv7l userspace on aarch64 cpu
    ARCH := armv7l
  endif
endif

ifeq ($(UNAME_S),Darwin)
  CC=clang
  CXX=clang++
  CFLAGS += -arch $(ARCH)
endif

ifeq ($(ARCH),arm64)
  CFLAGS += -fsigned-char
else
  ifneq ($(filter arm%,$(ARCH)),)
    CFLAGS += -fsigned-char -mfpu=vfp -march=armv6t2 -marm
  endif
  ifeq ($(ARCH),aarch64)
    CFLAGS += -fsigned-char
  endif
endif

ifndef ALLOW_WARNINGS
  ifneq ($(UNAME_S),Darwin)
    CFLAGS += -Werror
  endif
endif
ifndef DEPRECATED_WARNINGS
  CFLAGS +=  -Wno-deprecated-declarations
endif


default: loose_eel eel_pp

nseel-compiler.o: glue*.h ns-eel*.h
nseel-cfunc.o:  asm*.c ns-eel*.h
loose_eel.o: eel*.h ns-eel*.h
nseel-*.o: ns-eel*.h

vpath %.cpp ../lice ../swell
vpath %.mm ../swell
vpath %.c ../

ifdef MAXLOOP
  CFLAGS += -DNSEEL_LOOPFUNC_SUPPORT_MAXLEN=$(MAXLOOP)
else
  CFLAGS += -DNSEEL_LOOPFUNC_SUPPORT_MAXLEN=0
endif

ifdef DISASSEMBLE
  CFLAGS += -DEELSCRIPT_DO_DISASSEMBLE
endif

ifndef NO_GFX
    LICE_OBJS += lice.o lice_image.o lice_line.o lice_ico.o lice_bmp.o lice_textnew.o lice_text.o lice_arc.o
    CFLAGS += -DEEL_LICE_WANT_STANDALONE 

  ifeq ($(UNAME_S),Darwin)
    CLANG_VER := $(shell clang --version|head -n 1| sed 's/.*version \([0-9][0-9]*\).*/\1/' )
    CLANG_GT_9 := $(shell [ $(CLANG_VER) -gt 9 ] && echo true )
    ifeq ($(CLANG_GT_9),true)
      CFLAGS += -mmacosx-version-min=10.7 -stdlib=libc++
    else
      CFLAGS += -mmacosx-version-min=10.5
    endif
    SWELL_OBJS += swell-wnd.o swell-gdi.o swell.o swell-misc.o swell-dlg.o swell-menu.o swell-kb.o
    LFLAGS += -lobjc -framework Cocoa -framework Carbon
  else

    CFLAGS += -DSWELL_LICE_GDI -DSWELL_EXTRA_MINIMAL
    ifdef GDK2
      CFLAGS += -DSWELL_TARGET_GDK=2 $(shell pkg-config --cflags gdk-2.0)
      LFLAGS += $(shell pkg-config --libs gdk-2.0) -lX11 -lXi
    else
      CFLAGS += -DSWELL_TARGET_GDK=3 $(shell pkg-config --cflags gdk-3.0)
      LFLAGS += $(shell pkg-config --libs gdk-3.0) -lX11 -lXi
    endif
    ifndef NOFREETYPE
      CFLAGS += -DSWELL_FREETYPE $(shell pkg-config --cflags freetype2)
      LFLAGS += $(shell pkg-config --libs freetype2)
    endif

    SWELL_OBJS += swell-wnd-generic.o swell-gdi-lice.o swell.o swell-misc-generic.o \
            swell-dlg-generic.o swell-menu-generic.o swell-kb-generic.o \
            swell-gdi-generic.o swell-ini.o swell-generic-gdk.o

    LFLAGS += -ldl -lGL
  endif

endif

ifdef PORTABLE
  CFLAGS +=  -DEEL_TARGET_PORTABLE
else
  ifeq ($(UNAME_S),Darwin)
    ifeq ($(ARCH),x86_64)
      ASM_FMT = macho64
      NASM_OPTS = --prefix _
      OBJS2 += asm-nseel-x64-sse.o
    endif
  endif
  ifeq ($(UNAME_S),Linux)
    ifeq ($(ARCH),x86_64)
      ASM_FMT = elf64
      NASM_OPTS =
      OBJS2 += asm-nseel-x64-sse.o
    endif
  endif

asm-nseel-x64-sse.o: asm-nseel-x64-sse.asm
	nasm -D AMD64ABI -f $(ASM_FMT) $(NASM_OPTS) asm-nseel-x64-sse.asm

endif
CXXFLAGS=$(CFLAGS)

ifeq ($(CXX),g++)
  GCC_VER := $(shell $(CXX) --version|head -n 1| sed 's/.* \([0-9][0-9]*\)[.][0-9.]*/\1/' )
  GCC_GT_10 := $(shell [ "$(GCC_VER)" -gt 10 ] && echo true )
  ifeq ($(GCC_GT_10),true)
    CXXFLAGS += -std=c++03
  endif
endif

gen-yacc:
	yacc -v -d eel2.y

gen-lex:  # the output of this, lex.nseel.c, is unused because we have a handwritten parser instead
	flex eel2.l

%.o : %.mm
	$(CXX) $(CXXFLAGS) -c -o $@ $^

loose_eel: loose_eel.o $(OBJS) $(OBJS2) $(SWELL_OBJS) $(LICE_OBJS)
	g++ -o $@ $^ $(CXXFLAGS) $(LFLAGS)

eel_pp: eel_pp.o $(OBJS) $(OBJS2)
	g++ -o $@ $^ $(CXXFLAGS) $(LFLAGS)

clean:
	-rm -f -- loose_eel loose_eel.o eel_pp.o eel_pp $(OBJS) $(SWELL_OBJS) $(LICE_OBJS)

.PHONY: clean gen-lex gen-yacc
