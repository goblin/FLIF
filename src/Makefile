PREFIX := $(DESTDIR)/usr
CXXFLAGS := $(shell pkg-config --cflags zlib libpng)
LDFLAGS := $(shell pkg-config --libs libpng)

OSNAME := $(shell uname -s)
SONAME = -soname
ifeq ($(OSNAME),Darwin)
  SONAME = -install_name
endif

# for running interface-test
export LD_LIBRARY_PATH=$(shell pwd):$LD_LIBRARY_PATH

FILES_H := maniac/*.hpp maniac/*.cpp image/*.hpp transform/*.hpp flif-enc.hpp flif-dec.hpp common.hpp flif_config.h fileio.hpp io.hpp io.cpp config.h compiler-specific.hpp
FILES_CPP := maniac/chance.cpp maniac/symbol.cpp image/crc32k.cpp image/image.cpp image/image-png.cpp image/image-pnm.cpp image/image-pam.cpp image/image-rggb.cpp image/color_range.cpp transform/factory.cpp common.cpp flif-enc.cpp flif-dec.cpp io.cpp

all: flif libflif_dec.so libflif.so viewflif
lgpl: libflif_dec.so viewflif dflif

# options to consider: -fvect-cost-model=unlimited -funroll-all-loops
OPTIMIZATIONS := -DNDEBUG -O2 -ftree-vectorize
# there are often problems with clang and lto also it doesn't seem to know -fwhole-program
ifeq ($(CXX), g++)
	OPTIMIZATIONS := $(OPTIMIZATIONS) -flto -fwhole-program
endif

LIB_OPTIMIZATIONS := -DNDEBUG -O2
ifeq ($(CXX), g++)
	LIB_OPTIMIZATIONS := $(LIB_OPTIMIZATIONS) -flto
endif

# Command-line FLIF encoding/decoding tool - GPL
flif: $(FILES_H) $(FILES_CPP) flif.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(OPTIMIZATIONS) -g0 -Wall $(FILES_CPP) flif.cpp $(LDFLAGS) -o flif

# Command-line FLIF decoding tool - LGPL
dflif: $(FILES_H) $(FILES_CPP) flif.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(OPTIMIZATIONS) -DDECODER_ONLY -g0 -Wall $(FILES_CPP) flif.cpp $(LDFLAGS) -o dflif

# Decoder-only library - LGPL
libflif_dec.so: $(FILES_H) $(FILES_CPP) library/flif_dec.h library/flif-interface-private_dec.hpp library/flif-interface_dec.cpp
	echo $(OS)
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(LIB_OPTIMIZATIONS) -DDECODER_ONLY -g0 -Wall -shared -fPIC $(FILES_CPP) library/flif-interface_dec.cpp $(LDFLAGS) -Wl,$(SONAME),libflif_dec.so.0 -o libflif_dec.so.0
	ln -sf libflif_dec.so.0 libflif_dec.so

# Encoder-only library - GPL - probably not that useful (not built by default)
libflif_enc.so: $(FILES_H) $(FILES_CPP) library/flif_enc.h library/flif-interface-private_enc.hpp library/flif-interface_enc.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(LIB_OPTIMIZATIONS) -g0 -Wall -shared -fPIC $(FILES_CPP) library/flif-interface_enc.cpp $(LDFLAGS) -Wl,$(SONAME),libflif_enc.so.0 -o libflif_enc.so.0
	ln -sf libflif_enc.so.0 libflif_enc.so

# Decoder + encoder library - GPL
libflif.so: $(FILES_H) $(FILES_CPP) library/*.h library/*.hpp library/*.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(LIB_OPTIMIZATIONS) -g0 -Wall -shared -fPIC $(FILES_CPP) library/flif-interface.cpp $(LDFLAGS) -Wl,$(SONAME),libflif.so.0 -o libflif.so.0
	ln -sf libflif.so.0 libflif.so

libflif.dbg.so: $(FILES_H) $(FILES_CPP) library/*.h library/*.hpp library/*.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) -O1 -ggdb3 -Wall -shared -fPIC $(FILES_CPP) library/flif-interface.cpp $(LDFLAGS) -Wl,$(SONAME),libflif.so.0 -o libflif.dbg.so.0
	ln -sf libflif.dbg.so.0 libflif.dbg.so

# Example application: simple FLIF viewer - public domain
viewflif: libflif_dec.so viewflif.c
	$(CC) -std=gnu11 -O3 -ggdb3 $(shell sdl2-config --cflags) -Wall -Ilibrary/ viewflif.c -L. -lflif_dec $(shell sdl2-config --libs) -o viewflif


install: all
	install -d $(PREFIX)/bin $(PREFIX)/lib $(PREFIX)/share/man/man1
	install -s -m 755 flif viewflif $(PREFIX)/bin
	install -s -m 755 libflif_dec.so.* $(PREFIX)/lib
	install -s -m 755 libflif.so.* $(PREFIX)/lib
	install -m 644 ../doc/flif.1 $(PREFIX)/share/man/man1
	install -m 755 ../tools/gif2flif $(PREFIX)/bin
	install -m 755 ../tools/apng2flif $(PREFIX)/bin

install-dev:
	install -m 644 library/*.h $(PREFIX)/include

magic:
	if ! grep -q FLIF /etc/magic; then cat ../doc/flif.magic >> /etc/magic; fi

uninstall:
	rm -f $(PREFIX)/bin/flif
	rm -f $(PREFIX)/bin/viewflif
	rm -f $(PREFIX)/bin/gif2flif
	rm -f $(PREFIX)/bin/apng2flif
	rm -f $(PREFIX)/lib/libflif.so
	rm -f $(PREFIX)/lib/libflif_dec.so
	rm -f $(PREFIX)/lib/libflif.so.0
	rm -f $(PREFIX)/lib/libflif_dec.so.0
	rm -f $(PREFIX)/share/man/man1/flif.1

clean:
	rm -f flif dflif libflif*.so* viewflif flif.asan flif.dbg flif.prof flif.stats test-interface


# The targets below are only meant for developers

test-interface: libflif.dbg.so ../tools/test.c
	$(CC) -O0 -ggdb3 -Wall -Ilibrary/ ../tools/test.c -L. -lflif.dbg  -o test-interface

test: flif test-interface
	mkdir -p ../tmp-test
	./test-interface ../tmp-test/dummy.flif
	../tools/test-roundtrip.sh ../tools/2_webp_ll.png ../tmp-test/2_webp_ll.flif ../tmp-test/decoded_2_webp_ll.png
	../tools/test-roundtrip.sh ../tools/kodim01.png ../tmp-test/kodim01.flif ../tmp-test/decoded_kodim01.png
	../tools/test-roundtrip_anim.sh ../tools/endless_war.gif ../tmp-test/endless_war.flif
	../tools/test-roundtrip_anim_framedir.sh ../tools/bouncing_ball_frames ../tmp-test/bouncing_ball.flif

flif.stats: $(FILES_H) $(FILES_CPP) flif.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) -DSTATS $(OPTIMIZATIONS) -g0 -Wall $(FILES_CPP) flif.cpp $(LDFLAGS) -o flif.stats

flif.prof: $(FILES_H) $(FILES_CPP) flif.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(OPTIMIZATIONS) -g0 -pg -Wall $(FILES_CPP) flif.cpp $(LDFLAGS) -o flif.prof

flif.dbg: $(FILES_H) $(FILES_CPP) flif.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) -Og -ggdb3 -Wall $(FILES_CPP) flif.cpp $(LDFLAGS) -o flif.dbg

flif.asan: $(FILES_H) $(FILES_CPP) flif.cpp
	$(CXX) -std=gnu++11 $(CXXFLAGS) $(OPTIMIZATIONS) -fsanitize=address,undefined -fno-omit-frame-pointer -g3 -Wall $(FILES_CPP) flif.cpp $(LDFLAGS) -o flif.asan
