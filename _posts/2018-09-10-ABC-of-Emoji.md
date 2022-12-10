---
layout:     post
title:      The ABC's of Emoji
subtitle:   Unicode, character encoding, etc.
date:       2018-09-10
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Encoding
    - Emoji
---
> 技术分享时总结的关于emoji和字符编码相关的知识，这里只展示了文字部分，后续会把Keynote内容放出来。

# Quiz
- Which language can use US-ASCII to encode all its characters?
- How many characters can *char* represent in Java?
- Can we use *char* to represent ‘😀’ in Java?
- What will return if you call “😀”.length() and “😀”.getBytes() in Java? 
- Can we get the emoji calling “😀c”.substring(0,1)? 
- Can we execute *insert into tb(‘name’) values(‘😀’)* in MySQL?

# How to Define Character
- Character set - defines all `readable characters`
- Coded character set - use a `code point` to delegate a character in character repertoire
- Character encoding - the `transformation` between code point and its own code units

# Code Point
- A unique number assigned to each Unicode character
- Usually expressed in Hexadecimal as *U+xxxx*, e.g. code point for A is *U+0041*

# Planes
- 17 Planes
- 136755 characters defined
- U+0000~U+10FFFF, 21 bit
- Support over 1.1M possible characters 

# BMP
- Basic Multilingual Plane, *U+0000~U+FFFF*, 65535 in total

# Supplementary Characters
- Code points between *U+10000* and *U+10FFFF* are the supplementary characters
- Can not be described as a single 16-bit entity

# Character Encoding
- A *mapping* from the numbers of one ore more coded character sets to sequences of one or more 
fixed-width code units
- The most commonly used code units are *bytes*, but 16-bit, 32-bit integers can also be used for 
*internal processing*
- UTF-32, UTF-16 and UTF-8 are character encoding schemas for the Unicode standard

# UTF-32
- UTF-32 encodes each Unicode character as one 32-bit code units, e.g. 00 00 00 41 represent A
- It’s the most convenient representation for internal processing
- But it’s memory-wasting
- Try *"a".getBytes("utf-32")* in Java, you'll get *byte[4]{0, 0, 0, 97}*

# UTF-16
- UTF-16 encodes each `Unicode` character as `one or two` 16-bit code units, *U+0000~U+FFFF*, *0~65535*
- Each character is encoded using 2 or 4 bytes
- The internal Java encoding
- Code points between *U+0000* and *U+FFFF* are represented as a 16-bit Java char value
- e.g. U+4E2D -  中, 2 bytes, char c = ‘中’
- Code points between *U+10000* and *U+10FFFF* are the `supplementary characters` which char in Java can
 not hold
- Try *"HELLO".getBytes()* in Java, you'll see it's encoded using UTF-16

# Endianness
- Big Endian, e.g. *"a".getBytes("utf-16be")*->*byte[2]{0, 97}*
- Little Endian, e.g. *"a".getBytes("utf-16be")*->*byte[2]{97, 0}*

# BOM
- BOM = Byte Order Mark
- Big Endian starts with *U+FEFF*, e.g. *"a".getBytes("utf-16")*->*byte[4]{0xFE, 0xFF, 0, 97}*
- Little Endian starts with *U+FFFE*
- UTF-16 and UTF-32 have to deal with the issue of BE and LE, because they use *multi-byte* code units

# Java and Supplementary Characters
- Unicode was originally designed as a fixed-width 16-bit character encoding
- Java used to hold all Unicode characters using *char*
- But later Unicode 3.1 has been extended up to *1,114,112*, 21-bit character encoding
- Since J2SE5.0, JDK supports version 4.0 of Unicode standard

# UTF-8
- 8-bit, variable-width encoding
- Encodes each Unicode character using 1 to 4 bytes
- .class files is encode using UTF-8
- No BOM needed, try *"a".getBytes("utf-8be")*, you'll get *UnsupportedEncodingException*

# UTF-8 to UTF-16
- For those bytes：
    - starts with 0
        - e.g. 0xxxxxxx ==> 00000000 0xxxxxxxx
    - starts with 110,
        - e.g. 110xxxxx 10yyyyyy ==> 00000xxx xxyyyyyy
    - starts with 1110
        - e.g. 1110xxxx 10yyyyyy 10zzzzzz ==> xxxxyyyy yyzzzzzz
- e.g. “中”
    - Unicode U+4E2D: 01001110 00101101 
    - UTF-8 4E B8 AD : *1110*0100 *10*111000 *10*101101  

# UTF-16 to UTF-8
- For those bytes：
    - Less than 0x007F(00000000 01111111)
        - e.g. 0x00000000 xxxxxx ==> 0xxxxxxxx
    - Less than 0x07FF(00000111 11111111),
        - e.g. 00000aaa bbbbbbbb ==> 110aaabb 10bbbbbb
    - Others
        - e.g. aaaaaaaa bbbbbbbb ==> 1110aaaa 10aaaabb 10bbcccccc 
- e.g. “中”
    - Unicode U+4E2D: 01001110 00101101 
    - UTF-8 4E B8 AD : *1110*0100 *10*111000 *10*101101  

# Supplementary Encoding in UTF-16
- UTF-16 covers *U+0000~U+FFFF* using 2 bytes
- For Unicode U(*U+10000~U+10FFFF*)：
    - Minus 0x10000, get U’(0x00000~0xFFFFF), 20 bits
        - e.g. U’ = yyyyyyyyyyxxxxxxxxxx
    - Using W1 to represent the first 10 bits,
        - e.g. W1 = 110110yyyyyyyyyy, W1 in D800~DBFF
    - Using W2 to represent the second 10 bits,
        - e.g. W2 = 110111xxxxxxxxxx, W2 in DC00~DFFF

 
# Emoji History
- In 1999, *Shigetaka Kurita* created the first 180 emoji collection for a Japanese mobile web 
platform
- Sounds /ɪˈmoʊdʒi/ from Japanese
- "e"(picture), "moji"(character)

# Text&Color Shape
Emoji character can have two main kinds of presentation:
- An emoji presentation, with colorful and perhaps whimsical shapes, even animated
- A text presentation, such as black & white

# Emoji Modifiers
- **Emoji modifier** - A character that can be used to modify the appearance of a preceding emoji in an emoji modifier sequence
- **Emoji modifier base** - A character whose appearance can be modified by a subsequent emoji modifier in an emoji modifier sequence
- **Emoji modifier sequence** - A sequence of the following form: emoji_modifier_sequence := emoji_modifier_base emoji_modifier

# Fitzpatrick Modifiers
- When one of these characters follows certain characters, then a font should show the sequence as a single glyph with the specified skin tone
- If the font doesn’t show the combined character, the user can still see that a skin tone was intended

# Variation Selectors
- VS is a Unicode block containing 16 Variation Selector format characters(designated VS1 through VS16)
- They are used to specify a specific glyph variant for a Unicode character
- At present only standardized variation sequences with VS1, VS15 and VS16 have been defined

## VS-15
An *invisible code point* which specifies that the preceding character should be *rendered* in a textual fashion

## VS-16
- An *invisible code point* which specifies that the preceding character should be *displayed* with 
emoji presentation
- Only required if the preceding character defaults to text presentation
- Often used in Emoji `ZWJ Sequences`, where one or more characters in the sequence have text and Emoji presentation

# Emoji ZWJ Sequences
- ZERO WIDTH JOINER, U+0x200D
- Joining characters as a single glyph if available
- Behave like single emoji character, even though *internally* they are sequences

# Surrogate Pair
- It is possible to *combine* two code points defined in the `BMP` to express another code point that 
 lies outside of the first 65635 code points. This combination is called surrogate pair.
- `Leading Surrogate`: *U+D800~U+DB7F*
- `Trailing Surrogate`: *U+DC00~U+DFFF*
- The values from *U+D800* to *U+DFFF* are *reserved* for used in UTF-16, no characters are assigned to 
them as code points

# Java API
- String.codePointAt(int index):int
- Character.highSurrogate(int codePoint):char
- Character.lowSurrogate(int codePoint):char
- Character.charCount(int codePoint):int
- Character.isSupplementaryCodePoint(int codePoint):boolean
- Character.isSurrogate(char):boolean
- Character.isSurrogatePair(char, char):boolean
- …

# References
- https://en.wikipedia.org/wiki/Emoji
- http://stn.audible.com/abcs-of-unicode
- https://twitter.github.io/twemoji/preview.html
- http://www.unicode.org/reports/tr51/
- https://en.wikipedia.org/wiki/Fitzpatrick_scale
- http://www.oracle.com/technetwork/articles/javase/supplementary-142654.html
- https://en.wikipedia.org/wiki/UTF-8
- https://vinoit.me/2016/10/07/codePoint-in-java-and-utf16/
- https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/

> 本文首次发布于 [S.L's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,转载请保留原文链接.
