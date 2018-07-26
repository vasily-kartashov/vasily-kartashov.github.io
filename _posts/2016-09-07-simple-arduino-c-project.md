---
layout: post
title: Arduino with C / make on ubuntu
tags: c arduino
---

As I am not really a C / make person, let's document rather trivial findings on how to compile and deploy arduino based project without using Arduino IDE. It's a well known fact that most arduinos out there are exclusively utilized for LEDs blink, so who I am to argue? This is how you do it in C:


```c
#include <avr/io.h>
#include <util/delay.h>

#define BLINK_DELAY_MS 2000

int main (void) {
    /* set pin 5 of PORTB for output*/
    DDRB |= _BV(DDB5);

    while(1) {
        /* set pin 5 high to turn led on */
        PORTB |= _BV(PORTB5);
        _delay_ms(BLINK_DELAY_MS);

        /* set pin 5 low to turn led off */
        PORTB &= ~_BV(PORTB5);
        _delay_ms(BLINK_DELAY_MS);
    }
}

```

To compile and deploy let's use the following `Makefile`

```makefile
name = led

eeprom = target/$(name)
hex = $(eeprom).hex
source_files = $(wildcard src/*.c)
object_files = $(patsubst src/%.c, target/%.o, $(source_files))

.PHONY: clean

all: clean build

clean:
        @rm -rf target

build: $(hex)

run: $(hex)
        @avrdude -F -V -c arduino -p ATMEGA328P -P /dev/ttyUSB0 -b 115200 -U flash:w:$(hex)

$(hex): $(object_files)
        @avr-gcc -mmcu=atmega328p $< -o $(eeprom)
        @avr-objcopy -O ihex -R .eeprom $(eeprom) $(hex)

target/%.o: src/%.c
        @mkdir -p $(shell dirname $@)
        @avr-gcc -Os -DF_CPU=16000000UL -mmcu=atmega328p -c -o $@ $<

```

Which contains quite a few magical constants, but it's ok for now. Now we only need to install few command line tools

```bash
sudo apt-get install gcc-avr binutils-avr gdb-avr avr-libc avrdude
```

That's it, connect your Arduino to USB0 (if not, fix the Makefile), and run `make run`. Et voilà, another blinking LED. Before we proceed with drawing of the rest of the owl, let's see how we could use CLion, to make our work a bit more comfortable.

First, let's add the following `CMakeLists.txt` to our project (stolen mostly from [this guy](https://github.com/MultiMote/avr-clion/blob/master/CMakeLists.txt)):

```cmake
cmake_minimum_required(VERSION 3.0)
project(led)

set(DEVICE "atmega328p")
set(FREQ "16000000")

set(CMAKE_C_COMPILER avr-gcc)
set(CMAKE_CXX_COMPILER avr-g++)

include_directories(/usr/lib/avr/include)

set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")
set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")

set(CMAKE_C_FLAGS  "-Os -mmcu=${DEVICE} -DF_CPU=${FREQ}UL -std=gnu99 -Wl,--gc-sections")
set(CMAKE_CXX_FLAGS "-Os -mmcu=${DEVICE} -DF_CPU=${FREQ}UL -Wl,--gc-sections")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/target")

set(SOURCE_FILES src/led.c)

add_executable(${PROJECT_NAME} ${SOURCE_FILES})

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD COMMAND avr-objcopy -O ihex -R .eeprom ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${PROJECT_NAME} ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${PROJECT_NAME}.hex)
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD COMMAND avr-objcopy -O ihex -j .eeprom --set-section-flags=.eeprom="alloc,load" --change-section-lma .eeprom=0 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${PROJECT_NAME} ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${PROJECT_NAME}.eep)
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD COMMAND avr-size ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${PROJECT_NAME} --mcu=${DEVICE} --format=avr)
```

Run build, and it compiles from the IDE. For deployment create a simple run configuration in CLion IDE (not sure if there's a way to do that with CMake). Anyhow, now you have a proper IDE to support you in LED blinking endeavour.