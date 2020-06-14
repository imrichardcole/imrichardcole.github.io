---
title: Nessie - learning to write a NES emulator in C
layout: post
summary: This post, and hopefully series of posts, documents my journey with writing (yet another) NES emulator
---

Why am I writing a NES emulator?

1.  I'd like to learn C
2.  I grew up playing on the NES
3.  It's fun :)

### Some background

I've got about 30 years programming experience, 15 of those in professional development roles.

My background is Java which makes me a feel a bit 'spoilt' in terms of programming concepts.  It's kind of like being given a Zippo lighter as opposed to a stick and some dry leaves.  Yes java has it's complexities, but I've always felt I never really had to deal with some of the issues that lower level languages dealt with.  Hence the desire to learn C.

I'm not a graphical/UI developer.  The closest thing I've come to a 'UI' is some very basic websites.  

I've never done any game development.

### Getting started

So the purpose of this task, and the whole series in fact, is to keep things simple.  The whole possiblity of getting a working NES emulator with graphics, audio, input etc seems pretty far off at this point, but that's ok.  I want to try and break each task down into the simplest possible parts.  So to that end, let's start with the ROM and specfically the header

### ROM, ROM, ROM...

I'm working with a ROM that's in [iNES](https://wiki.nesdev.com/w/index.php/INES) format.  This dictates that the first 16 bytes contain the header.  So first task, let's write some code to open the ROM file and extract these 16 bytes:

{% highlight c %}
	#include <stdio.h>

	void read_header(FILE * rom_file) {
		const int header_size = 16;
		unsigned char buffer[header_size];
		size_t read = fread(buffer, sizeof(buffer), 1, rom_file);
		printf("Read %lu bytes from header\n", (read * sizeof(buffer)));
		for (int i = 0; i < 16; ++i) {
			const char byte = buffer[i];
			printf("%d = %d/0x%02X (%c)\n", i, byte, byte, byte);
		}
	}

	int main() {
		FILE * rom_file = fopen("mario.nes", "rb");
		read_header(rom_file);
	}
{% endhighlight %}

Which, when run, will give us:

		Read 16 bytes from header
		0 = 78/0x4E (N)
		1 = 69/0x45 (E)
		2 = 83/0x53 (S)
		3 = 26/0x1A ()
		4 = 2/0x02 ()
		5 = 1/0x01 ()
		6 = 1/0x01 ()
		7 = 0/0x00 ()
		8 = 0/0x00 ()
		9 = 0/0x00 ()
		10 = 0/0x00 ()
		11 = 0/0x00 ()
		12 = 0/0x00 ()
		13 = 0/0x00 ()
		14 = 0/0x00 ()
		15 = 0/0x00 ()

The first column after the index gives us the numerical value in both decimal and hex of each byte and then, where possible, the value in the brackets shows it as an ASCII character.  Nice to see the NES value in 0-3!  So now lets focus on the actual content, starting with bytes 4 and 5:

		4 = the size of the PRG ROM (where the actual game is) in 16384 byte blocks
		5 = the size of the CHR ROM (where the sprite data is) in 8192 byte blocks

In the example above we can then see that we have 2 * 16384 bytes of PGR and 1 * 8192 of CHR data.  Now the next two interesting bytes are in 6 and 7.  At this point the byte isn't treated as a whole but is being split up into individual bits to represent flag type values i.e on/off.

#### Flags 6/7

The upper 4 bits (or nybble as it's known) are uses to represent half of a byte used for the 'mapper'.  The other half is stored in flag 7.  The lower four bits are used to represent various flags.  To collect this data, I need a couple of functions that use [bitwise operations](https://en.wikipedia.org/wiki/Bitwise_operation#Bit_shifts) to get at that data:

{% highlight c %}

	int get_nth_bit_from_byte(char byte, int bit) {
		return (byte >> bit) & 1;
	}

	int get_lower_nibble(char byte) {
		return (byte & 0x0F);
	}

	int get_upper_nibble(char byte) {
		return (byte >> 4);
	}
{% endhighlight %}

And some examples of using the above:

{% highlight c %}

	flag_six = 47;
	printf("7th: %d\n", get_nth_bit_from_byte(flag_six, 7));
	printf("6th: %d\n", get_nth_bit_from_byte(flag_six, 6));
	printf("5th: %d\n", get_nth_bit_from_byte(flag_six, 5));
	printf("4th: %d\n", get_nth_bit_from_byte(flag_six, 4));
	printf("3rd: %d\n", get_nth_bit_from_byte(flag_six, 3));
	printf("2nd: %d\n", get_nth_bit_from_byte(flag_six, 2));
	printf("1st: %d\n", get_nth_bit_from_byte(flag_six, 1));
	printf("0th: %d\n", get_nth_bit_from_byte(flag_six, 0));

	const int upper = get_upper_nibble(45);
	printf("Upper: %d\n", upper);

	const int lower = get_lower_nibble(45);
	printf("Lower: %d\n", lower);

{% endhighlight %}

Giving:

		7th: 0
		6th: 0
		5th: 1
		4th: 0
		3rd: 1
		2nd: 1
		1st: 1
		0th: 1
		Upper: 2
		Lower: 13

Given the values I have for flags 6 and 7 (1 and 0) are pretty unexiting, the only useful value I get out is the 0th bit in flag 6 (set to 1) giving us:

		Mirroring:	0: horizontal (vertical arrangement) (CIRAM A10 = PPU A11)
	              		1: vertical (horizontal arrangement) (CIRAM A10 = PPU A10)

Which we'll come to at a later date.

### Wrapping it up

So now I've parsed what I needed, I've created a small struct to hold the data:

{% highlight c %}

struct Header {
   	int prg_banks;
   	int chr_banks;
   	char flag_six;
   	char flag_seven;
};

{% endhighlight %}


