Welcome to this new project, where I'm trying to reverse engineer the assets files of the series of games of **Who Wants To Be A Millionaire** developed by Hothouse Creations.

Why?
----------
One of the first games I played was *Chi Vuol Essere Miliardario* for PlayStation, which is the italian version of the game. One day I decided I wanted to add my own questions to the game and, maybe, do more things.

Important notes
----------
* I'm italian and I'm not 100% familiar with the english language, so there will be mistakes. I hope i'll write this as correct as possible.
* I started reverse engineering the files without taking a look to the game code. I'll try to specify that in the code, but consider, most of them, as working hypothesis.
* This is a project which I don't want really focus too much
* I'm not an expert writing on GitHub and Reverse Engineering code, especially on PlayStation and Dreamcast
* All the values seems to use a Little endian byte ordering

Main goals
----------
The main goals of this project are:
* Decode the STY file, which seems to be a unique file created by Hothouse Creations
* Decode the files containing the database with all the questions and their metadata
* Get the high quality voice samples, as heard during the gameplay
* Get all the other media

Tested on...
----------
Unfortunately is very difficult to test every game in the series, with all their variants. Even if they seems very similar, but there are some differences.

### Tested on this version of the game
* Chi Vuol Essere Miliardario (Prima edizione) - PlayStation. This is the first edition of the italian version of WWTBAM
* The PC and Dreamcast Versions seems to use a similar approach. I haven't checked the GBA, PS2 and MacOS Variants

Which files are on the PlayStation disc?
----------
*Note: There's a STY file on the Dreamcast version too. I haven't check yet, but the file should be very similar, only with the cutscenes available inside separate files. In this section I will only talk about the Playstation version.

### The Playstation disk contains only four files
* **SLES_035.82** - Will change based on the edition, this is for the italian version
* **SYSTEM.CNF** - This is a file that the PlayStations looks to execute the game code
* **SILENCE.WAV** - I don't know why this file exists, but it's a 4 minute, silent WAV file
* **PSX.STY**

The structure of the STY file
----------
:white_check_mark: **Decoded**

*Note: I don't think I can put the data here to make an example, so I'll make a picture with the scheme, just to be more clear.
The STY file is what contains any other file. This file is not encrypted or compressed, but it seems a non-standard format. This seems to be how it's structured sequentially:
* **4 bytes** - The size of the header containing data about the files in the list. More on that later. The count includes these 4 bytes.
* **4 bytes** - The number of files in the next succession. If the value is - for example - 7, there will be 7 subsequential files metadata. This value doesn't indicate all the files available inside the STY, as there may be other successions of files.
* **24 bytes** - The file name. Seems to be compliant with the ANSI format and they're readable easily with a text editor
* **4 files** - The size of the file in bytes. There are folders too in this list, which are indicated as big 0 bytes
* **4 bytes** - The offset of the file relative to the beginning of the file. This value seems to be in "blocks", with 1 block big 2048 bytes. So, to get the right offset, you've to multiply the value by 2048

After the header, all the files are subsequential until the end of the file.

The PSM file format
----------
:thinking: **Hypotetical use**
:x: **Unkown format**

Unfortunately I haven't found which format are these encoded, altough I'm pretty sure these are the video clips.

The 3Minutes file
----------
:x: **Unknown usage**

This file is a mystery at the moment, since contains a random pattern with the names of the developers.

The SSL File
----------
:white_check_mark: **Known use**
:thinking: **Partially Decoded**

This contains the audio clips, including both the speech and the music. Sincie it's on PlayStation, this should be a PS-ADPCM format.
A tool like PSound can play and convert this file to wav.
* **4 bytes** - The size of the header. These 4 bytes are excluded from the count.
* A list of two values

  * **4 bytes** - The offset in number of blocks. A block is 2048 bytes
  * **4 bytes** - The size of the sample. :thinking: *Not sure if this value is in number of blocks or bytes*
  
* **? bytes** - there are some bytes containing ANSI text. They seems referencing some images
* All the audio clip data, in sequential order.

The EGG Files
----------
:white_check_mark: **Known use**
:thinking: **Partially Decoded**

This is a container with mixed image formats inside, used for sprites and background images. Fonts are included in another file format.
We can divide the structure of this file in three areas:
* The header for the entire file, divided in two parts
* The data for the TIM-like sprites. I'll refer to these as TIM files even though they doesn't seem to contain any header for the "standard" TIM file. More on that later.
* The data for the SB-like backgrounds. I'll refer to these as SB files

* The first part of the header seems to be always 16 byte long and is structured like this
  * **4 bytes** - The file signature, always 45 47 47 00 or EGG
  * **4 bytes** - The number of bytes for some data about the TIM images
  * **4 bytes** - The number of bytes for the offsets of the BS images
  * **2 bytes** - Unknown. It's always 0x00 inside all the files I analyzed
  * **2 bytes** - The number of TIM images multiplied by 2. Maybe it's indicating another thing, but the data corresponds with this idea
 
* The second part of the header contains some additional info about the TIM and the BS images
  * The data about the TIM files are currently unknown
  * The SB data contains only the offset for all the SB files. These are a series of 4 bytes long values. The first one is always 00 00 00 00
 
* Immediately after the header data, we can see the TIM files in series. Any TIM file is big 131072 bytes, which is the same number as the number of pixels of the game resolution (512x256)
  Like a TIM file, the images use a CLUT system, where any pixel has the index to the color in a palette (CLUT = Color Look Up Table), also defined inside the same 512x256 bytes. I don't have the exact structure of this file for here.

* Immediately after the TIM files, we can see the SB files in series. These are also known as the single frames inside the .STR video files. These are compressed 256x256 images.
  The SB files contains all the necessary header of the file and can be read with an official tool such as MC32 (AKA Movie Converter). I got almost no issue reading them with the Version 3.4 of the tool.

Even if there's a file signature EGG, this format is a non-standard and non-documented format.

The PFF Files
----------
:white_check_mark: **Known use**
:thinking: **Partially Decoded**

These seems to include the fonts with some metadata. There are multiple files, each for different sizes.

The TSB File
----------
:white_check_mark: **Known use**
:thinking: **Partially Decoded**

The first important file for the question database. This contains the character map and, maybe, other data. This seems ANSI compliant too.
The letters inside the db files don't use an ANSI encoding, but refers - with an offset - to this file.
I think there's more to this file, but it's yet to be discovered.
The letter with index 1 starts at the offset 0xA and is one word big. The letters aren't sorted alphabetically. There are numbers and symbols too in the mix.

For example, if in the db file is used the word `02 00`, the letter to get is at the address `0x0C`

The DB Files
----------
:white_check_mark: **Known use**
:thinking: **Partially Decoded**

This contains the question database, alongsize the answers and soem metadata. This is the structure (Not complete at the moment)
* 2048 bytes for the header. I haven't fully decoded these, but I can find some familiar values

  * The words at 0x02 and 0x08 value is equal to the block size. Maybe one indicates the size of the header and the other the size of the question area
  * The word at 0x06 contains the number of the questions. In the italian version it seems there are 1011 questions.
  * The word at 0x04 is 14, the maximum value for the difficulty (0-14 -> 15 values)
  * From 0x0E to the end of the header, there are multiple words in sequence, ranging from 1 to 16384. I think this is one indicator for the difficulty of the question, since the range is from log2(1) = 0 to log2(16384) = 14. 15 different values in total. There are some 0x00 values until the end of the header should serve as padding. 

* 2048 bytes for each question in sequence.

  * **4 bytes** - Seems an unique numeric ID for the question
  * **2 bytes** - The difficulty of the question? This seems a redundant value found inside the header.
  * **1024 bytes** - The question. Any word is a letter. The values are indexes to be used as offset in the TDB file.
  * **128 bytes** - The answer A
  * **2 bytes** - The percentage of the help of the public for the answer A. It seems there are two possible values, one for each byte.
  * **128 bytes** - The answer B
  * **2 bytes** - The percentage of the help of the public for the answer B. It seems there are two possible values, one for each byte.
  * **128 bytes** - The answer C
  * **2 bytes** - The percentage of the help of the public for the answer C. It seems there are two possible values, one for each byte.
  * **128 bytes** - The answer D
  * **2 bytes** - The percentage of the help of the public for the answer D. It seems there are two possible values, one for each byte.
  * **498 bytes** - The metadata of the question. Only six bytes seems to indicate something. All the other bytes are 0x00, with a 0x01 to end the section.
  
    * **1byte** - The correct answer, in range 0 - 3
    * **1byte** - The answer given by the phone call, in range 0 - 4. This is almost always identical to the answer. A value of 4 indicates that the helper doesn't know how to answer.
    * **1byte** - The remaining option - alongside the correct answer - while using a 50:50 help, in range 0 - 3. All the values seems different from the correct answer, which led me to think this is what it is.
    * **1byte** - Probably which person will respond to the phone. In the italian version there are 10 different persons.

Conclusion
----------
I'll update this document more in the future. All of these assumptions were made without looking at the game code.

