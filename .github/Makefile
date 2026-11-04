TARGET := FauxM0n
BUILD := build
SOURCES := .
CFILES := $(notdir $(wildcard $(SOURCES)/*.c))
OFILES := $(CFILES:.c=.o)

ARCH := -mthumb -mthumb-interwork -mcpu=arm7tdmi
CFLAGS := -g -Wall -O2 $(ARCH)
LDFLAGS := $(ARCH)

LIBS := -lgba

include $(DEVKITARM)/gba_rules
