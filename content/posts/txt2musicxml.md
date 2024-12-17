---
date: '2024-11-27T00:12:40+01:00'
title: 'Converting simple text into beautiful chord sheets'
draft: false
tags:
    - music
    - antlr
    - cli
---
Whenever I learn or write a new song, I am usually next to my computer, and I found that the easiest way for me to write down the chords, is to just open up a simple text editor and write it quickly with a super simple syntax.

<!--more-->
---
> #### TL;DR:
> Turn this:
> ```
> Cmaj7 | A7 | Dm9 | G7b9 |
> ```
> Into this:
> ![](/txt2musicxml/chords.png)

---

The "rules" of the syntax are:
- Roots of chords are capital letters (`A`, `C` `G`)
- Accidentals are `#` `b`
- All suffixes (or chord types) are be lowercase ascii letters, symbols and numbers (`maj7`, `-7b5`, `7b9,13`)
- Bass notes use a slash `/`

Full example:
```
Cmaj7 | A7 | Dm9 | G7b9 |
```

The problem is that, although quite simple and fast, it is very ugly and not super comfortable to read while playing it back. So I built a CLI tool that parses this input and turns into something more meaningful: MusicXML.

MusicXML is a pretty good standard, mainly because it is widely accepted. I don't know of any music software that doesn't have MusicXML import/export. It's not an easy standard, but the end user doesn't need to know the details. The important thing is that with my new tool I can turn my simple chord syntax into MusicXML, and then import it into my favorite software to render it. Luckily, MuseScore is open source, and has a CLI tool that can turn MusicXML into pdf in 1 cli command.

### Antlr
Well, I had no idea how to do that. Luckily, I found Antlr, which is a really cool open source software to create lexers and parsers (and more) in your favorite programming language. You basically define your language's grammar in a single file in a straightforward syntax, that is very much similar to regex. The hard part was learning the theory behind it. What is a language, what is a lexer, what is a parser, etc. So here are some of the important concepts I learned:
![](/txt2musicxml/antlr_process.png)

#### Tokens
A token is the "atom" of a language. The smallest significant part of it. I already defined it above! For example a chord root is a token (`D`), an accidental is a token (`#`), a suffix is a token (`maj7`). But also a barline is a token `|` and a newline is a token `\n`.

#### Lexer
A lexer is the part of the program that turns raw text (an input stream) into tokens. So basically it turns:
```
Cmaj7 | A7 | Dm9 | G7b9 |
```
into:
```
['C', 'maj7', '|', 'A', '7', '|', 'D', 'm9', '|', 'G', '7b9', '|']
```

#### Parser
A parser gives meaning to tokens. It takes each token in the stream and assigns it a meaning. It also turns it into something called a parse-tree.
In our examples, it would be something like this:
![](/txt2musicxml/parsetree.png)

> We can start seeing patterns and meaning in our language :)

#### Visitor (AST)
A Parser Tree still contains all the original text, but a lot of this is not necessarily important to be able to do something with it. We need to simplify our tree to a model that makes more sense. Generically this is called an AST (Abstract Syntax Tree). In our case I will call it a "Sheet". Antlr has a concept of a Visitor, which walks over all the nodes of the parse tree, and does something with them. In my case, it turned the parse-tree into python classes that have significant relationships with them. A Sheet contains lines, lines contain bars, bars contain chords, a chord has a root, and possibly a suffix and a bass, etc. In our example, it would be something like this (simplified):
```python
Sheet(lines=[
    Line(bars=[
        Bar(chords=[Chord(root='C', suffix='maj7',)]),
        Bar(chords=[Chord(root='A', suffix='7',)]),
        Bar(chords=[Chord(root='D', suffix='m9',)]),
        Bar(chords=[Chord(root='G', suffix='7b9',)]),
    ])
])
```

#### XML Generator
Now that we are in python-land, it's quite easy (yet very tedious), to create a MusicXML generator, which is basically a class that accepts a sheet, walks down the nodes and generates the correct xml according to the [MusicXML specifications](https://www.w3.org/2021/06/musicxml40/).


The result is this:
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE score-partwise PUBLIC "-//Recordare//DTD MusicXML 4 Partwise//EN" "http://www.musicxml.org/dtds/partwise.dtd">
<score-partwise version="4"><part-list><score-part id="P1"><part-name>Chords</part-name></score-part></part-list><part id="P1"><measure number="1"><attributes><divisions>1</divisions><key><fifths>0</fifths><mode>major</mode></key><time><beats>4</beats><beat-type>4</beat-type></time><clef><sign>G</sign><line>2</line></clef></attributes><harmony><root><root-step>C</root-step></root><kind use-symbols="yes">major-seventh</kind></harmony><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note><harmony><root><root-step>A</root-step></root><kind use-symbols="yes">dominant</kind></harmony><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note></measure><measure number="2"><attributes><divisions>1</divisions></attributes><harmony><root><root-step>D</root-step></root><kind use-symbols="yes">minor-ninth</kind></harmony><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note><harmony><root><root-step>G</root-step></root><kind use-symbols="yes">major</kind></harmony><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note><note><pitch><step>B</step><octave>4</octave></pitch><duration>1</duration><type>quarter</type><stem>none</stem><notehead>slash</notehead></note></measure></part></score-partwise>
```
> Yikes!

### What to do with this XML???
Save it as a file and you can open it with any music software that supports it (all of them basically).
MuseScore is an open source software, and it's pretty feature rich. Luckily, they have a CLI tool that let's you import/export files. I believe it was originally intended to help with batch conversions, but we can use it for our own benefits. The CLI I wrote accepts chord text in stdin, and spits XML in stdout, it's fully pipe-able and redirectable. So you can do whatever you want in one line.

Pipe a string of chords into the cli and get the xml:
```shell
echo -n 'Cmaj7 A7 | Dm9 G7b913 |' | txt2musicxml
```

or redirect input/output from/to a file:
```shell
txt2musicxml < path/to/Thriller.crd > path/to/Thriller.musicxml
```

or convert it directly to pdf using Musescore:
```shell
TMPSUFFIX=.musicxml; mscore -o path/to/output.pdf =(txt2musicxml < path/to/input.crd)
```