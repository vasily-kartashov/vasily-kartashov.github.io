---
layout: post
title: Arduino with C / make on ubuntu
---

As I am not really a C / make person, let's document rather trivial findings on how to compile and Aeploy arduino based project without using Arduino IDE. It's a well known fact that most arduinos out there are deployed to make LEDs blink, so who I am to argue? This is how you do it in C:


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

That's it, connect your Arduino to USB0 (if not, fix the Makefile), and run `make run`. Et voilà, another blinking LED. Now draw the rest of the owl.
