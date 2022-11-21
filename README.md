Welcome to this new project, where I'm trying to reverse engineer the series of games of **Who Wants To Be A Millionaire** developed by Hothouse Creations.

Why?
----------
One of the first games I played was *Chi Vuol Essere Miliardario* per PlayStation, which is the italian version of the game. One day I decided I wanted to add my own questions to the game and, maybe, do more things.

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
* Get the videos as played on the various platforms

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
Unfortunately I haven't found which format are these encoded, altough I'm pretty sure these are the video clips.

The 3Minutes file
----------
This file is a mystery at the moment, since contains a random pattern with the names of the developers.

The SSL File
----------
This contains the audio clips. The clips are placed sequentially inside this file, but I haven't found a way to decode it properly yet. I think there isn't only raw audio data here. 
I tried to put this file in Audacity and, with some tweaks, I can barely listen to the presenter voice (Gerry Scotti in my case) with a lot of noise

The EGG Files
----------
This seems to contains part of the textures of the game. I haven't found the extact codec, but it should be a raw bitmap. Just need to find the size of the file and the bit depth

The PFF Files
----------
These seems to include the fonts with some metadata. There are multiple files, each for different sizes

The TSB File
----------
The first important file for the question database. This contains the character map and, maybe, other data. This seems ANSI compliant too.
The letters inside the db files don't use an ANSI encoding, but refers - with an offset - to this file.
I think there's more to this file, but it's yet to be discovered.
The letter with index 1 starts at the offset 0xA and is one word big. The letters aren't sorted alphabetically. There are numbers and symbols too in the mix.

For example, if in the db file is used the word `02 00`, the letter to get is at the address `0x0C`

The DB Files
----------
This contains the question database, alongsize the answers and soem metadata. This is the structure (Not complete at the moment)
* 2048 bytes for the header. I haven't fully decoded these, but I can find some familiar values
* * The words at 0x02 and 0x08 value is equal to the block size. Maybe one indicates the size of the header and the other the size of the question area
* * The word at 0x06 contains the number of the questions. In the italian version it seems there are 1011 questions.
* * The word at 0x04 is 14, the maximum value for the difficulty (0-14 -> 15 values)
* * From 0x0E to the end of the header, there are multiple words in sequence, ranging from 1 to 16384. I think this is one indicator for the difficulty of the question, since the range is from log2(1) = 0 to log2(16384) = 14. 15 different values in total. There are some 0x00 values until the end of the header should serve as padding. 
* 2048 bytes for each question in sequence.
* * **4 bytes** - Seems an unique numeric ID for the question
* * **2 bytes** - The difficulty of the question? This seems a redundant value found inside the header.
* * **1024 bytes** - The question. Any word is a letter. The values are indexes to be used as offset in the TDB file.
* * **128 bytes** - The answer A
* * **2 bytes** - Unknown. This doesn't seem to be part of the letters
* * **128 bytes** - The answer B
* * **2 bytes** - Unknown. This doesn't seem to be part of the letters
* * **128 bytes** - The answer C
* * **2 bytes** - Unknown. This doesn't seem to be part of the letters
* * **128 bytes** - The answer D
* * **500 bytes** - The metadata of the question. Only six bytes seems to indicate something. All the other bytes are 0x00, with a 0x01 to end the section.
* * * **1byte** - Unknown
* * * **1byte** - Unknown
* * * **1byte** - The correct answer, in range 0 - 3
* * * **1byte** - Maybe the answer given by the phone call, in range 0 - 3. This is almost always identical to the answer.
* * * **1byte** - Maybe the remaining option - alongside the correct answer - while using a 50:50 help, in range 0 - 3. All the values seems different from the correct answer, which led me to think this is what it is.
* * * **1byte** - Unknown

Conclusion
----------
I'll update this document more in the future. All of these assumptions were made without looking at the game code.

