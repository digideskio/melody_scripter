.. |--| unicode:: U+2013   .. en dash

Melody Scripter
===============

**Melody Scripter** is a Python application which parses melody files written
in an easy-to-write textual format into an internal Python object model.

For the purposes of **Melody Scripter**, a "melody" consists of a sequence
of notes on the standard Western musical scale, together with bar lines
(which must match the specified time signature) and chords, with optional
bass notes where different from the chord root note.

The internal object model can generate a Midi file for the melody.

Currently Midi output is the only functionality provided by the object model,
but the object model also provides a convenient representation of melody information
for the purposes of scientific analysis.

Melody Scripter File Format
===========================

The Melody Script file format is line-oriented. Currently there are two types
of input lines:

* **Command** lines. Any line starting with the character '*' (with possibly
  preceding whitespace) is parsed as a command line.
* **Song Item** lines. All other lines are parsed as sequences of whitespace-separated Song Items.

Command Lines
-------------

A command line starts with a **command** (after the initial '*' character).

The command is followed by a ':' character, and then one or more
comma separated arguments.

Currently there are four commands, which are **song**, and three 'track' commands:
**track.melody**, **track.chord** and **track.bass**.

All four commands take arguments which are **property settings**, consisting 
in each case of a **key** and a **value**.

Currently all property settings can be executed only prior to any song items,
but in future they may be allowed during the song (or additional commands may
be defined which are valid within the song).

Song Property Settings
----------------------

Available property settings for the **song** command are:

+----------------+--------------------------------------+------------+--------------+
| key            | value                                | default    | valid values |
+================+======================================+============+==============+
| tempo_bpm      | Tempo in beats per minute            | 120        | 1 to 1000    |
+----------------+--------------------------------------+------------+--------------+
| beats_per_bar  | Beats per bar                        | 4          | 1 to 32      |
+----------------+--------------------------------------+------------+--------------+
| ticks_per_beat | Ticks per beat                       | 4          | 1 to 2000    |
+----------------+--------------------------------------+------------+--------------+

The **tempo_bpm** and **ticks_per_beat** values both determine corresponding values when
a Midi file is generated. "Ticks" are the unit of time in the song, and every note
or rest length must be a whole number of ticks |--| if not, an error occurs.

The **beats_per_bar** value defines the required length of each complete bar. It has no effect on Midi
output, but if the contents of a bar do not have the correct total length, it's an error.
(It's OK to have partial bars at the start of end of the song.)


Track Property Settings
-----------------------

The three tracks, **melody**, **chord** and **bass**, correspond to three Midi tracks generated in the Midi output file. 
Each track has its own settings:

+----------------+--------------------------------------+------------+--------------+
| key            | value                                | default    | valid values |
+================+======================================+============+==============+
| instrument     | Midi instrument number               | 0          | 0 to 127     |
+----------------+--------------------------------------+------------+--------------+
| volume         | Midi volume (velocity)               | 0          | 0 to 127     |
+----------------+--------------------------------------+------------+--------------+
| octave         | Octave for initial melody note, and  | 3, 1, 0    | -1 to 10     |
|                | for all chord root notes and all     |            |              |
|                | bass notes.                          |            |              |
+----------------+--------------------------------------+------------+--------------+

(The octave defaults are for **melody**, **chord** and **bass** respectively.)

The **instrument** and **volume** settings define the Midi settings for each track. Midi instrument numbers
range from 0 to 127, and the actual sounds depend entirely on the SoundFont used to play the Midi song,
although there is a standard **GM** set of Midi instruments definitions (where the default of **0** 
corresponds to a Acoustic Grand Piano).

Currently Melody Scripter does not have any provision for per-note volume (velocity) specification. In
practice there is no easy way to determine appropriate volume values, for example when typing in from
sheet music. For playback it is recommended to choose suitable instrument sounds that work well with 
constant volume (for example see choices made in the sample song files in this project).

The **octave** setting determines which Midi octave the first melody note belongs to, and for
the **chord** and **bass** tracks, it determines the octave of all root notes and bass notes respectively.
(Melody note octave values are determined relatively, as will be described in the Song Items section next.)

Although octave values are allowed from -1 to 10, not all Midi notes in the 10th octave are allowed,
and an error will occur if a note occurs with a value greater than 127.

Song Items
----------

There are six types of song item that can be parsed:

* Notes
* Ties
* Rests
* Chords
* Bar Lines
* A Cut

All song items are represented by tokens that don't contain any whitespace, and song items in a line must
be separated from each other by whitespace.


Notes
-----

The components of a note are, in order:

Continued marker:
  If provided, specified as "~". This indicates that a note is a continuation
  of the previous note.
Ups or downs:
  If provided, specified as one or more "^" for up, or one or more "V" for down.
Note letter:
  A lower case letter from "a" to "g". For the purposes of defining an octave,
  the octave starts at "c" (this is a standard convention).
Sharp or flat:
  Represented by "+" or "-", and only one is allowed.
Duration:
  The note duration is specified as a number of beats, with optional qualifiers.
  The default number of beats is either 1, for the first note, and the first note
  in each bar. Possible qualifiers are "h" and "q", which can both occur zero or
  more times, and representing a halfing and quartering of length in each case,
  "t", (for triplet), which divides the note length by three, and "." which multiples
  the note length by 1.5. "t" and "." can only occur once. Any note duration must
  be a whole number of ticks, and an error will occur if a note length is defined
  which is a fractional number of ticks. (In such a case, if the note length is
  correct, you will need to increase or change the specified **ticks_per_beat**
  song property.)
To-be-continued marker:
  If provided, specified as "~". This indicates that a note will be continued
  by the next note.

Except for the very first note, Melody Scripter does not provide for each note to
specify its octave. Instead, pitch values are specified relatively to the previous note.
If no "ups" or "downs" markers are specified, the rule is to always choose the closest
possibility. If this choice is ambiguous, ie when going from 'f' to 'b', then an error occurs.

If one up or one down is specified, then the next note should be the first note above
or below the previous note, respectively. If more than one up or down marker is given, 
the go an extra octave up or down for each extra marker.

So, for example, "c" followed by "e" means go up to the next "e", and "c" followed
by "^e" *also* means go up to the next "e". Whereas "^^e" means go up 9 notes to the "e"
above that, "Ve" means go down to the first "e" below, and "VVe" means go to the "e" 
below that one.

Ties, and Note Continuations
----------------------------

A **continuation** is where one note is represented by the joining of two or more
note items in the melody script. Because bar lines have to occur in the right place,
notes that cross bar lines *have* to be represented using continuations. There may
also be some note lengths that cannot be represented using the Duration format
specified above, so they have to be constructed from multiple notes.

In other situations, the use of continuations is optional.

There are two ways to specify that one note is to be continued by a second note:

* Either, the first note ends with "~" and the second note starts with "~",
* Or, a "~" **Tie** item occurs between the two notes.

It is entirely possible for more than two notes to form a continuation |--| the
required joinings just need to be indicated in each case. This would be necessary,
for example, to specify a note that filled more than two bars.

Rests
-----

A **Rest** consists of the letter "r" followed by a duration specification. The duration
specification for rests is very similar to that for notes, but there is no default
duration, and at least one part of the duration specification must be given. If
only qualifiers are given, then they are applied to a value of 1. So, for example,
"rh" is a valid rest, representing half a beat.

Chords
------

**Chords** are specified by enclosing their contents in "[" and "]". Currently there 
are two formats:

Root note plus descriptor
  The root note is given as an upper-case letter with an optional "+" or "-" for sharp or flat,
  and one of several standard "descriptors" from "" (for a major chord), "7", "m",
  "m7" and "maj7". So, for example, "[Cm]" represents a C minor chord.
Root note plus other chord notes.
  Prefixed with a ":", the notes are given as upper-case letters with optional "+"/"-" sharp
  or flat, with the root note first. So, for example, "[:CE-G]" represents a C minor chord.

In each case, chords may contain an optional bass note specifier, to specify a bass note
different from the root note. This is given as a "/" character, followed by an upper-case
letter and optional sharp or flat. So, for example, "[:A+m/F+]" represents A sharp minor
with an F sharp bass.

Bar Lines
---------

**Bar Lines** are represented by "|". Bar lines are used to check that the total lengths of notes
and rests in each bar have the correct values. They also effectively reset the default note
duration to 1 beat. Bar lines do not have any direct effect on Midi output.

Cuts
----

A **Cut** is represented by "!". **Cut** means "cut out all previous song items". This song item
is very useful when you want to play part of the song without starting all the way from the beginning.


Playback
========

The **main()** method of **play_song.py** generates a Midi file from the Song file whose name is
given as the first argument. After generating the Midi file, this method also plays it using 
the "/usr/bin/cvlc" command, if that command is available. **cvlc** is the command line version of VLC, 
as installed on an  Ubuntu system, and it only plays Midi files if the **vlc-plugin-fluidsynth** VLC plugin is installed.

(An alternative playback option on Ubuntu is **timidity**, however even with the **--output-24bit**
option, on my system, the sound quality is poor at the beginning of the song.)