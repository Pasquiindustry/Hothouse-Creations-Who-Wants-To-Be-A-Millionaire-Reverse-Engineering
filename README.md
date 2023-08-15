Welcome to this new project, where I'm trying to reverse engineer the assets files of the series of games of **Who Wants To Be A Millionaire** developed by Hothouse Creations.

Why?
----------
One of the first games I played was *Chi Vuol Essere Miliardario* for PlayStation, which is the italian version of the game. One day I decided I wanted to add my own questions to the game and, maybe, do more things.

Important notes
----------
* I'm italian and I'm not 100% familiar with the english language, so there will be mistakes. I hope i'll write this as correct as possible.
* I started reverse engineering the files without taking a look to the game code. I'll try to specify that in the code, but consider, most of them, as educated hypothesis.
* This is a project which I don't want really focus too much. Update: I'm focusing too much to this project.
* I don't have a lot of experience with both writing on GitHub and reverse engineer PlayStation and Dreamcast games.
* Most of the values - if not all - uses a little endian notation. This is standard for the PlayStation hardware.
* To avoid Copyright issues, I'll avoid writing the bytes of the game here. This may impact the readability of this document. I may find a better way to represent the data.

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
|Game name|Platform|Game code|Description|
|---|---|---|---|
|Chi Vuol Essere Miliardario|PlayStation|SLES-03582|The first edition of the italian version, using Lira Italiana as currency|
|Who Wants to Be a Millionaire?|PlayStation|SLES-02988|This is the first edition of the european / uk version of WWTBAM|


Which files are on the PlayStation disc?
----------
*Note: There's a STY file on the Dreamcast version too. I haven't check yet, but the file should be very similar, only with the cutscenes available inside separate files. In this section I will only talk about the Playstation version.

### The Playstation disk contains only four files
|File name|Description |
|---|---|
| **SLES_035.82** | The name of this file depends on the edition. This is valid for the first italian edition. This is the executable |
| **SYSTEM.CNF** | This is a standard file for multiple PlayStation games. The Bios will look for this file, which contains the entry point for the main executable |
| **SILENCE.WAV** | :x: Unknown. It's a 4 minute silent WAV file, complete with a standard signature and header|
| **PSX.STY** | This contains all the assets of the game. This doesn't contain game code.

The STY file
----------
| | |
|---|---|
|Status|:white_check_mark: **Decoded**|
|Type|Archive|
|Standard|No|

This is the only file containing all the assets of the game. This file is not encrypted or compressed per se. Consider this as a virtual hard-disk.

### File structure
This file is divided into two main parts: the header and the actual data.
This archive supports directory hierarchy.
If you want to modify the assets of the game, it's better to change only the size and the offset of each file, since the game code knows which file wants to load by their name and the file/folder hierarchy is not needed.

* **4 bytes** - The first 4 bytes of data contains the size of the entire header. The number ncludes this 4 bytes too, so the count starts from offset 0.

Then there's a loop. Every cycle it's data from a different directory. This will continue for the entirety of the header
* **4 bytes** - The number of files inside the current folder. If this value is 7, there are 7 file in succession. The first succession of files are considered to be placed inside the root directory

Each file have these values

* **24 bytes** - The name of the storage item with the extension. Seems to be compliant with the ANSI format and they're readable easily with a text editor. The name of the parent directory is absent.
* **4 files** - The size of the storage item in bytes. We can both identify files and folders with this value, since directories have a value of 0.
* **4 bytes** - This value indicates two different things if the item is a file or a folder.
    * For files (Size > 0) this value indicates the location of the data inside the STY. This number is in number of blocks, so you've to multiply this by 2048 to get the exact offset.
    * For directories (Size == 0) this value indicates the offset of the succession of files available inside the folder. This should point to the 4 bytes containing the number of files inside the folder.

After the header, all the files are subsequential until the end of the file.

The PSM file format
----------
| | |
|---|---|
|Status|:white_check_mark: **Almost entirely decoded**|
|Type|Video|
|Standard|No. Raw data contains VAG and BS files|

This is the video clip file format.

Although there are some similarities with the STR file format, these are not directly compatible with the STR format.

All the videos inside the game seems to use a 256x256 resolution and they're played at about 12.5 fps with stereophonic audio at 22050Hz

All the things available in this file are in their 2048 bytes block
A single block can contain only one of these data

|Type|Description|Format|
|---|---|---|
|Header|More on the structure of the header later. In a single file, multiple headers can be present|
|Audio|Two channels of audio data are stored indipendently each other|Headerless VAG / PlayStation ADPCM, Mono, 22050Hz|
|Image frames|This type of block can contain a single image frame|BinaryStream / A frame of an STR file / Playstation MDEC|

### The header(s)
This is the structure of the headers
* **2 bytes** - Currently unknown. The first byte may be a signature since it's the same across all files. The second byte is slightly different.
    * IMPORTANT: These two bytes are only available on the first block!
    
Then, in succession, until the end of the header (2046 or 2048 blocks) you'll find this succession of values:
* **1 byte** - The type of data inside the next blocks. These are the possible values
    * 0x00 - Nothing
    * 0x03 - Audio channel
    * 0x04 - Audio channel
    * 0x02 - Image frame
    * 0x01 - The type may be available on the next block. Since each block is maximum 2048 bytes, if a file is bigger than this value, it will use more blocks. A 0x01 value will appear until the file ends, with the last block indicated with the correct type. If you find 01 01 01 02, you knwo that there's a frame that occupies 4 blocks.

Since the header can contain only 2046 on the first block or 2048 on the subsequent blocks, you can only represent a limited amount of data. If the total of blocks is, for example, 3000 blocks, you'll find a second header and subsequent files. 

0x01 blocks can fall between two headers.

Here's a table explaining the structure in a different way. Note that this is just an example, not a real file.
|Offset in bytes|Type of data|Indicated as this value inside the header|Info|
|---|---|---|---|
|0|Header (With the first two bytes)|
|2048|Audio|0x03|
|4096|Audio|0x04|
|6144|Frame|0x02|
|8192|Frame|0x02|
|10240|Audio|0x03|
|12288|Audio|0x04|
|14336|Look for the next block|0x01|
|16384|Look for the next block|0x01|
|18432|Look for the next block|0x01|
|20480|Frame|0x02|This BS takes 4 blocks total|
|22528|Frame|0x02|This BS takes 1 block total|
|...| | |
|4188160|Audio|0x04|
|4190208|Look for the next block|0x01|
|4192256|Header| |
|4194304|Look for the next block|0x01|
|4196352|Frame|0x02|This BS takes 3 blocks total|

The 3Minutes file
----------
| | |
|---|---|
|Status|:x: **Unknown**|
|Type|Plain text|
|Standard|?|

This file is a mystery at the moment, since contains a random pattern with the names of the developers.

The SSL File
----------
| | |
|---|---|
|Status|:white_check_mark: **Almost entirely decoded**|
|Type|Audio|
|Standard|No. Contains multiple headerless VAG audio files|

This contains the audio clips, including both the speech and the music. It doesn't contain the sound for the videos, which are embedded inside their respective PSM files.

All the clips, once extracted, seems to respect the PlayStation ADPCM / headerless VAG format.

Since we don't have any header, we've to guess which format the game expects. Some details are still to be decoded.

### File structure
* **4 bytes** - The first 4 bytes of this file contains the size of the header. These 4 bytes are excluded from the count. Be careful since this is different from the STY file!

Then there's a succession of these values inside the header

* **4 bytes** - The offset in number of blocks. A block is still 2048 bytes.
* **2 bytes** - The size of the sample in number of blocks.
* **1 byte** - A value that indicates different "variants" of the audio.

    The most common value is 0, which indicates a 22050Hz 1 channel audio clip.
    
    Other values found are 1 and 3, which are currently unknown. It seems they both define a stereo audio clip. It's possible to still play the clips without major issues from inside the game if we change this value to 0.
    * Sounds with a value of 1 seems dedicated to intros and idents
    * Sounds with a value of 3 seems dedicated to background music.
    
Then, between the header and the actual succession of raw data, there's this:
  
* **? bytes** - some bytes containing ANSI text. They seems referencing some images that are nowhere found inside the game disk.

Finally...

* All the audio clip data.

The EGG Files
----------
| | |
|---|---|
|Status|:white_check_mark: **Almost entirely decoded**|
|Type|Images|
|Standard|No. Contains two types of media: TIM-Like, 8 bit with CLUT and BitStream / MDEC / Playstation STR video frame|

This is a container with mixed image formats inside, used for sprites and background images. Fonts are not included, since they're available in another file format.
We can divide the structure of this file in three main areas:
* The header for the entire file.
* The data for the TIM-like sprites. I'll refer to these as TIM files even though they doesn't seem to contain any header for the "standard" TIM file. More on that later.
* The data for the BS-like backgrounds. I'll refer to these as BS files

### The header

This file is divided into multiple parts. This is because there are - at least on the PlayStation version - Two different type of data: multiple TIM-like images and multiple BS images.
If you want to edit this file, consider that we only need to change the pixels and, if you want to go deeper, the CLUT on the Tim-like images.

#### The first part

On the two versions I tested, I found that there's a small but critical difference. The file signature may or may not be present at the start of the file.

The first part of the header seems to be always 12 or 16 byte long and is structured like this:
  * **Optional 4 bytes** - The file signature, always 0x45 0x47 0x47 0x00 or EGG in plain text.
  * **4 bytes** - The number of bytes for some data about the TIM images
  * **4 bytes** - The number of bytes for the offsets of the BS images
  * **2 bytes** - Unknown. It's always 0x00 inside all the files I analyzed
  * **2 bytes** - The number of TIM images multiplied by 2. Maybe it's indicating another thing, but the data corresponds with this idea
 
#### The second part: The header for the TIM-like data

The next part of the header contains some additional info about the TIM-like data.
This section is a succession of 8 byte values. Every item on this list is a portion of the 512x256 TIM-like image
  
The meaning of each value is:
* The X coordinate of the sprite
* The Y coordinate of the sprite
* The width of the sprite
* The height of the sprite
* Currently unknown
* The x coordinate of the CLUT
* The Y coordinate of the CLUT
* Currently unknown
    
As we can see, even though the width of the TIM is 512 pixels, the maximum value a single byte can have is 255. To find the exact reference, we have to double the X coordinates and the Width values.

Another thing: we don't have the size of the CLUT. We can consider that the height is 1 pixel and the width is 512 pixels. Any color is 16 bit wide with the following structure (Each letter = one bit):

    RRRRRGGG GGBBBBBT

Where
* R = Red
* G = Green
* B = Blue
* T = Threat this index as a transparent color
    
#### The third part: The offsets of the BS files

As written inside the title, here we can only find a series of offsets. Here we'll know where we can find all of our backgrounds and still-images.

The first one is always 0x00 0x00 0x00 0x00, since the offsets are relative to only the BS data portion. This portion starts immetiately after the raw pixels of the TIM-like files.
 
#### The final parts: the raw data

Immediately after the header data, we can see the TIM files in series.
Any TIM file is exactly 131072 bytes, which is the same number as the number of pixels of the game resolution (512x256).

Like a TIM file, the images use a CLUT system, where any pixel has the index to the color in a palette (CLUT = Color Look Up Table), also defined inside the same 512x256 bytes.

Be careful: the color data inside a CLUT is 2 pixels wide or 16 bit, where the pixels are effectively 1 byte.

As written before, immediately after all the Tim-like files, there are the BS files. These are known as multiple names, since it's only used on the first PlayStation. Some examples are: BitStream, MDEC Images, STR still frames.

All the images are 256x256 pixels. We can convert this to a known format by using community-made tools like jpsxdec or very old but official tools like MC32.

The PFF Files
----------
| | |
|---|---|
|Status|:thinking: **There's work to do**|
|Type|Font metadata|
|Standard|No|

These seems to include the fonts with some metadata. There are multiple files, each for different sizes.

The TSB File
----------
| | |
|---|---|
|Status|:thinking: **There's work to do**|
|Type|Glue logic between fonts and database|
|Standard|No|

The first important file for the question database. This contains the character map and, maybe, other data. This seems ANSI compliant too.
The letters inside the db files don't use an ANSI encoding, but refers - with an offset - to this file.
I think there's more to this file, but it's yet to be discovered.
The letter with index 1 starts at the offset 0xA and is one word big. The letters aren't sorted alphabetically. There are numbers and symbols too in the mix.

For example, if in the db file is used the word `02 00`, the letter to get is at the address `0x0C`

The DB Files
----------
| | |
|---|---|
|Status|:white_check_mark: **Almost entirely decoded**|
|Type|Question database|
|Standard|No.|

This contains the question database, alongsize the answers and soem metadata. This is the structure (Not complete at the moment)

**Important note**: this is mostly valid for the question database. The fast-finger database - for example - seems to slightly differ. I don't have an exact structure for that at the moment.

* 2048 bytes for the header. I haven't fully decoded these, but I can find some familiar values

  * The words at 0x02 and 0x08 value is equal to the block size. Maybe one indicates the size of the header and the other the size of the question area
  * The word at 0x06 contains the number of the questions. In the italian version it seems there are 1011 questions.
  * The word at 0x04 is 14, the maximum value for the difficulty (0-14 -> 15 values)
  * From 0x0E to the end of the header, there are multiple words in sequence, ranging from 1 to 16384. I think this is one indicator for the difficulty of the question, since the range is from log2(1) = 0 to log2(16384) = 14. 15 different values in total. There are some 0x00 values until the end of the header should serve as padding. 

* 2048 bytes for each question in sequence.

  * **4 bytes** - Seems an unique numeric ID for the question
  * **2 bytes** - The difficulty of the question? This seems a redundant value found inside the header.
  * **1024 bytes** - The question. Any word is a letter. The values are indexes to be used as offset in the TDB file.
  * **128 bytes** - The text for the answer A
  * **2 bytes** - The percentage of the help of the public for the answer A. It seems there are two possible values, one for each byte.
  * **128 bytes** - The text for the answer B
  * **2 bytes** - The percentage of the help of the public for the answer B. It seems there are two possible values, one for each byte.
  * **128 bytes** - The text for the answer C
  * **2 bytes** - The percentage of the help of the public for the answer C. It seems there are two possible values, one for each byte.
  * **128 bytes** - The text for the answer D
  * **2 bytes** - The percentage of the help of the public for the answer D. It seems there are two possible values, one for each byte.
  * **498 bytes** - The metadata of the question. Only six bytes seems to indicate something. All the other bytes are 0x00, with a 0x01 to end the section.
  
    * **1byte** - The correct answer, in range 0 - 3
    * **1byte** - The answer given by the phone call, in range 0 - 4. This is almost always identical to the answer. A value of 4 indicates that the helper doesn't know how to answer.
    * **1byte** - The remaining wrong possible answer while using a 50:50 help, in range 0 - 3.
    * **1byte** - Probably which person will respond to the phone. In the italian version there are 10 different persons.
    
How to read different formats / codecs
----------
### Headerless VAG / PlayStation ADPCM
Tools like PSound, Vgmstream, Vag2Wav and Wav2Vag can play and convert this file to wav.

#### More about Vgmstream
To convert or play a VAG file, it's possible to use this txth with Vgmstream.

Note that this will work well only with the clips marked with a 0x00 type byte.

The interleave value is random since I don't know the correct value. Any value seems to work at the moment.

    codec = PSX
    interleave = 0x1000
    sample_rate = 22050
    channels = 1
    padding_size = auto-empty
    num_samples = data_size
    
### PlayStation BitStream image / PlayStation STR Video Frame
It wasn't easy to find tools that will work with a raw BitStream image.
This is currently my setup:

#### From BS to a known image format via a terminal
I use Jpsxdec, which was extremely helpful with this project.
Here's the command to convert a BS to a bitmap:

    java -jar <Path to the jpsxdec jar> -f <Path to the BS file (With the file name)> -static bs -dim 256x256 -fmt bmp
    
Remember to replace the parameters between the <>. The output will be saved to the working directory.
#### From a known image format to a BS via a terminal
Unfortunately - at the moment - it seems that Jpsxdec doesn't support the creation of a BS file.
Luckly I found an old tool made by Sony called MC32, also called Movie Converter or MOVCONV.

With the version 3.4 of this tool, it's possible to convert a raw RGB image into a BS image. It also supports TIM and YUV image formats.

Calling MC32 via a terminal is a little bit different than usual, since you need to provide a script file to the program. Here's a script I made which I found works without issues

    Rgb2bsV(<Path to the input rgb file (With the file name)>, 256, 256, <Path to the destination BS file (With the file name)> , x2, 15fps, 2, 3, FALSE);

Save this script as a .scr file, just like a plain text file.
To execute this script with MC32, just call this command

    MC32.EXE -s <Path to the scr file>

Conclusion
----------
I'll update this document more in the future. All of these assumptions were made without looking at the game code.
