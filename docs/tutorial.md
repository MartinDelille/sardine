---
hide:
    - navigation
---

# Using Sardine

At least! You have now installed and configured Sardine. You are up and running, but you don't know how to use **Sardine**! This tutorial will hopefully help you to understand the different layers that are composing the performance/composition/improvisation system that **Sardine** is. I will not dive deep into details or materials dedicated to contributors or developers but give you the bare minimum to become a proficient **Sardine** user.

## Sardine Clock

### MIDI Clock

When **Sardine** is imported using the command `from sardine import *`, an instance of `Clock` will automatically start to run in the background and will be referenced by the variable `c`. `Clock` is the main MIDI Clock. By default, if you haven't touched to the configuration, the clock will be running in `active` mode: it will send a **MIDI** clock signal for every tick on the default MIDI port. The default MIDI port will either be a virtual port named `Sardine` if your OS supports virtual MIDI ports or the first available MIDI Port declared by your OS. It can also be `passive` and made to listen to the default MIDI port if you prefer. Never override the `c` variable. You won't have to worry a lot about the internals. You will likely find the following commands interesting:

* `c.bpm`: current tempo in beats per minute.
* `c.ppqn`: current [PPQN](https://en.wikipedia.org/wiki/Pulses_per_quarter_note) (Pulses per Quarter Note, used by MIDI gear).
  - be careful. The tempo might fluctuate based on the PPQN you choose. Assume that 24 is a default sane PPQN for most synthesizers/drum machines.
* `c.accel`: an acceleration factor for the clock, from `0` to `100` (double tempo) %. 
* `c.nudge`: nudge the clock forward in time by the given amount. Usually pinged randomly until you fall back on the external click track you wish to follow. 

### Ableton Link Clock

The **Link** protocol is a novel open source protocol released by **Ableton** which allows users to synchronise their musical tempo seamlessly on a local network. While still pretty new, this method of synchronisation is now supported by a fair amount of by music software and apps, including other live coding libraries.

**Sardine** can be made to start or follow an **Ableton Link** Clock that will be shared by all users on the local network. To do so, you will need to join/start a session using the `c.link()` method. Be mindful that the regular behavior of the clock will be altered and that you won't be able to change the tempo or alter time the way you want. **Link** is a collaborative clocking protocol, and there is no "main" tempo originating somewhere and followed by everyone, unlike MIDI. To resume the regular behavior of the clock, use the command `c.unlink()`. The `c.link_log()` function can be used to monitor the **Ableton Link** Clock state. 

The **Ableton Link** Clock is a bit weird. It will disrupt the regular behavior of the Sardine Clock. It will stop emitting a MIDI clock signal because it cannot ensure that the clock will be steady. **MIDI** Clocks and **Ableton Link** Clock does not go hand in hand. It is preferable to kill every running pattern before attempting the switch from regular time to Link time.

### Some clock things to know

- Sardine will not behave nicely if no external clock is running while in `passive` mode. Time is simply frozen and events will not trigger, suspended somewhere in time.

- You can introspect the current state of the clock using clock attributes or using the very verbose `debug` mode. Be careful, your terminal will be flooded by messages. After running `c.debug = True`, you should see something that looks like:

```python
...
BPM: 130.0, PHASE: 15, DELTA: 0.001726 || TICK: 495 BAR:2 3/4
BPM: 130.0, PHASE: 16, DELTA: 0.001729 || TICK: 496 BAR:2 3/4
BPM: 130.0, PHASE: 17, DELTA: 0.001475 || TICK: 497 BAR:2 3/4
BPM: 130.0, PHASE: 18, DELTA: 0.000634 || TICK: 498 BAR:2 3/4
BPM: 130.0, PHASE: 19, DELTA: 0.000614 || TICK: 499 BAR:2 3/4
BPM: 130.0, PHASE: 20, DELTA: 0.001333 || TICK: 500 BAR:2 3/4
...
```

- Some interesting clock attributes can be accessed:
    * `c.beat`: current clock beat since start.
    * `c.tick`: current clock tick since start.
    * `c.bar`: current clock bar since start (`4/4` bars by default).
    * `c.phase`: current phase.

These clock attributes are used everywhere in **Sardine** as they provide the most basic interface to the clock for every component in the system. You can use them if you wish to compose more complex pieces and sequences. They are still really useful tools to craft conditionals and random number generators even though there are better and more controlled ways to access them (for instance, *via* the *patterning system*).

#### The meaning of sleep

If you are already familiar with *Python*, you might have heard about or used the `sleep()` function. This function will halt the execution of a program for a given amount of time and resume immediately after. **Sardine** does not rely on Python's `sleep` because it is *unreliable* for musical purposes! Your OS can decide to introduce micro-delays, to resume the execution too late or even not sleep for the precise duration you wanted. 

**Sardine** proposes an alternative to regular Python `sleep`, backed by the clock system previously described, crafted by @thegamecracks. The `sleep()` function has been overriden to allow you to have a safe and sane, similarly working alternative for musical contexts. You can use it to stop and resume a **swimming function** while keeping synchronization and timing accuracy.

```python
@swim
def sleeping_demo(d=1):
    print("Doing something...")
    sleep(1)
    print("Doing something else...")
    sleep(1)
    again(sleeping_demo, d=2)

@swim
def limping(d=4):
    S('hh').out()
    sleep(3)
    S('bd').out()
    again(limping, d=4)
```

The **swimming function** `sleeping_demo()` will recurse after a delay of `2`. Think of the time you have in-between a recursion as spare time you can use and consume using `sleep()`. You can use that time sending the instructions that compose your swimming function. You can also do nothing for most of your time just like in `limping()`. You can write code in an imperative fashion, something that you might have already encountered in live coding systems such as [Sonic Pi](https://sonic-pi.net/) or **SuperCollider** `Tdefs`.

**Be careful**! You can oversleep and trigger a recursion while your function is still running, effectively overlapping different versions of your **swimming functions**:   

```python
@swim
def oversleep(d=4):
    S('hh').out()
    sleep(3)
    S('bd').out()
    again(oversleep, d=0.5) # Changed the value to oversleep
```

If you are not yet familiar with the concept of recursion, or with the meaning of some of the facts presented here, keep patience. The meaning of all this will become clear after a few sections and some tests on your side :)


#### Swimming functions

We already used the term **swimming function** before without taking the time to explain it! In **Sardine** parlance, a **swimming function** is a function that is scheduled to be repeated by recursion. The function will call itself when it ends. The newly called function will call itself, and again, and again... This is one way computer scientists like to think about loops and structures like lists. To define a function as a **swimming function**, use the `@swim` decorator. The opposite of the `@swim` decorator is the `@die` decorator that will release a function from this dreadful recursive temporal loop.

```python
@swim # replace me by 'die'
def bd(d=1):
    """Loud bassdrum"""
    S('bd', amp=2).out()
    again(bd, d=1) # again == anew == cs
```

If you don't manually add the recursion to the designated **swimming function**, the function will run once and stop. Recursion must be explicit and you should not forget about it! Forgetting the recursion loop call is another way to make a **swimming function** stop but not the recommended one!

```python
# Boring
@swim 
def bd(d=1):
    S('bd', amp=2).out()
```

The recursion can (and should) be used to update your arguments between each call of the **swimming function**. Here is an example of a very simple iterator:

```python
@swim # or die 
async def iter(d=1, nb=0):
    """A simple recursive iterator"""
    print(f"{nb}")
    again(iter, d=1, nb=nb+1)
# 0
# 1
# 2
# 3
# 4
```

This is an incredibly useful feature to keep track of state between each call of your function. **Swimming functions** and its handling of arguments are the most basic thing you have to learn in order to use **Sardine** proficiently. This is the base to improvise music with variation, nuance and finesse. Temporal recursion makes it very easy to manually code LFOs, musical sequences, randomisation, etc... It will gradually become like a second nature for you to write them.

**Swimming functions** are great but they have one **BIG** difference compared to a classic recursion: they are *temporal* recursive. They must be given a `delay` argument. The `delay` argument is actually `d` (shorter is better). If you don't provide it, Sardine will assume that your function uses `d=1`. If you forget it on one side while using it on the other side, **Sardine** will jump up to your neck and try to kill you. If you ever try to give a delay of `0`, your function will immediately stop and an error message will be printed in your terminal. Not waiting is simply not an option, otherwise we would equally be able to travel back in time :)

### Making sound / sending information

#### Sender objects

Sender objects are the most frequent objects you will be interacting with while playing with **Sardine**. Senders are objects that compose a single message that can be sent out using the `.out(iter=0)` method. They are your main interface to the outside world (*SuperCollider*/*SuperDirt*, *MIDI* or *OSC*). These objects can receive various and/or arbitrary parameters depending on their purpose and specialty. These arguments can be *integers* (`1`, `2`), *floats* (`1.23`, `0.123123`) or *strings* (`"baba"`, `"dada/43/baba/"`):

- *int*/*float*: parameters are sent as is, they are numbers!
- *string* : interpreted by **Sardine** and transformed into a pattern of values.

When you import **Sardine**, `MIDISender`, `SuperDirtSender` and `OSCSender` will already be available under the name `M()` (for **MIDI**),  `S()` (for **sound** or *SuperDirt*) and `O()` (for **OSC**). These objects are preconfigured objects that must be prefered to custom senders you can declare yourself (more on this later). `M`, `O` and `S` are three ways of interacting with the synthesis engine, your synths or other equipment/softwares.

#### SuperDirt output

The easiest way to trigger a sound with **Sardine** is to send an OSC message to **SuperDirt**. **SuperDirt** is designed as a tool that will convert control messages into the an appropriate action without having to deal with **SuperCollider** itself. Most people will use the **SuperDirt** output instead of plugging multiple synthesizers along with **Sardine**, or craft musical patches listening to OSC messages. The interface to **SuperDirt** is crude but fully functional. People already familiar with [**TidalCycles**](https://tidalcycles.org/) will feel at home using the `S()` (for `SuperDirt`) object. The syntax is extremely similar for the purpose is similar, and names often match between the two systems:  

```python
# A bassdrum (sample 0 from folder 'bd')
S('bd').out() 
# Fourth sample, way louder!
S('bd', n=3, amp=2).out() 
# Patterning a parameter (read the appropriate section) 
S('bd', n=3, amp=1, speed='1,0.5').out(i) 
# Introducing some Python in our parameters
from random import random, randint
S('bd' if random() > 0.5 else 'hh', speed=randint(1,5)) 
```

#### Delayed messages

You can pre-declare a sound before sending it out. This allows you to build your messages incrementely before sending them out using the `.out()` method.

```python
@swim
def delayed(d=0.5):
    sound = S('bd')
    if sometimes():
        sound.shape(0.5)
    else:
        sound.speed(4)
    sound.out()
    again(delayed)
```

Do not use the assign operator (`=`). Call the attribute directly (eg: `amp()`). Any attribute can be set but they are not checked for validity. This is an useful feature if you prefer to write your code in an imperative fashion. There are other things to know about delayed composition:

- attributes can be chained: `S('cp').speed('1 2').room(0.5).out()`
- attributes **will** be parsed. You can write patterns just like you do when you send the objet out directly.

Attributes are not checked for validity. You can really write anything so be careful: spell out the *SuperDirt* attribute names correctly.

This technique is also really interesting if you like to pre-compose things before playing them. You can write some additional code to store libraries of prepared audio samples matching with some custom parameters.

#### Orbits

You will soon find out that you can assign effects to audio samples played using `S()` such as a reverb, a low-pass filter or a bitcrusher. Some of these effects are **local**. They only affect the sound you are currently playing. Some other effects are **global**. They will affect all the sounds running through the same audio bus. This is an important distinction to keep in mind! To specify which bus you would like to use for a given sound, use the `orbit` argument.

`orbit` is a very important argument. Actually, it is the only argument that is specified by default. You can run a sound through a very heavy and long-tailed reverb while having, concurrently, another dry sound by switching orbits:
```python3
S('clap', room=0.9, dry=0.1, size=0.9, orbit=0).out()
S('bd', orbit=3).out()
```

The number of orbits at your disposition is declared on the **SuperDirt** side in your configuration file. More on this later on! In the meantime, check out the **SuperDirt** GitHub repository if you would like to learn more about it.

#### MIDI Output

The `MIDISender` object is structurally similar to the `SuperDirtSender` object. It is specialized in writing/sending MIDI notes and MIDI notes only. Other MIDI events are handled differently by specific methods such as `cc()` (for control changes) or `pgch()` (for program changes). MIDI Notes messages need a duration, a velocity, a note number and a channel:

- **duration** (`seconds`): time between a *note-on* and *note-off* event (pressing a key on an imaginary keyboard).
- **velocity** (`0-127`): think of it as the volume amplitude of a note.
- **channel** (`0-16`): the MIDI channel to send that note to on your default MIDI port.
- **note** (`0-127`): note from the lowest possible octave on a keyboard to the highest.

```python
M(delay=0.2, note=60, velocity=120, channel=0)
```

The `.out()` method is still used to carry a note out and to inform the sender of the value you would like to select in a pattern (see patterning). All of these values have a given default. If you don't specify anything (`M()`), you will hear a very loud middle-C on channel 0. See the `MIDI` section to learn more about sending out other message types such as *control changes* or *program changes*.

#### OSC output

The `OSCSender` is the weirdest of all the senders. It behaves and works just like the other ones but will require more work from your part. By default, the object cannot assume what OSC connexion you would like to use. For that reason, you will need to feed him one with an aditional parameter. Likewise, the sender cannot assume what address you would like to carry your message to. You will need to feed the object an address everytime you wish to send a message out. There is no default OSC connexion (except for the one used by **SuperDirt** and **Sardine** internally).

To use the `OSCSender` object, you will have to open manually an OSC connexion before feeding it into the object:
```python
my_osc = OSC(ip="127.0.0.1", port= 12000, name="super_connexion", ahead_amount=0.25)
O(my_osc, 'loulou', value='1,2,3,4').out()
```

Argument names do not matter when composing OSC messages. You can name arguments, but this will only make it easier for you to name and find values in your code. The message being caried out will not feature the name of the argument you specified. I'm still pondering if that is a nice thing or if I should find another system. If you have an opinion about it, please voice it!

### Composing patterns

The most intriguing aspect of **Sardine** is that you can write small patterns of values that can be used to make your functions more lively on their own. These patterns can be used to generate rhythms, streams of notes/addresses/numbers, random values, etc... Instead of feeding static values such as `0.5` to an argument like `amp`, you can make it change dynamically on-the-fly by feeding sequence and/or patterns of values: `0.5, 0.3, 0.2` or `{0_1(0.1)}`. There is a whole syntax dedicated to crafting patterns and I plan to add more and more in subsequent versions of **Sardine**!

Think of all this as a glorified time-dependant calculator that can also do some arithmetics on lists, with new operators such as `_`, `!` or `:`. The pattern system also works with musical notes and names (for samples and OSC addresses). It can be used to summon musical scales, musical chords and some funny musical transformations (such as `.disco` or `.explode`). All of this is *maïeutic*, aka meant to fuel your imagination and let you explore a world of dynamic algorithmic musical patterns. The grammar in itself is a bit complex, but can be found living in the deepest corner of the source code.

There are three basic types of values that can be used to compose a **Sardine** pattern:

- **notes** : musical notes.

- **names** : names denoting an address or an audio sample file name.

- **numbers**: floating-point numbers or integers.

All these values, even if not of the same type, can interact with one another, which means that you can write a scale down and transpose it by a given factor just by writing an addition: `C->penta + 2`.

If you are familiar with *live coding* already, this idea of playing with algorithmic patterns might ring a bell. If you are not, let me start now with more detailed explanations.

#### Sardine grammar

Think of the pattern language as a novel way to write lists. Fundamentally, **Sardine** patterns are just list of values, separated each by commas: `1,2,3,4`. Whitespace don't matter. Each element written between commas can be a more complex expression: `1+2, 2*4, r, {5_10}`. Some values are atomic, they only are one thing. `1` is nothing but `1`. Some values are more complex, and can represent lists of values: `C-min7` is a four-note structure made of the notes composing the C minor 7 chord.

Lists are defined as a list of operations **separated by whitespaces**. I will present some patterns in gradual order of complexity before detailing the list of available operators.

```python
# parser('1 2 3'): a list of numbers
[1.0, 2.0, 3.0]

# parser('1 2 3*3'): multiplication on last number
[1.0, 2.0, 9.0]

# parser('r+1 r/4 r*2'): random numbers and math
[1.691424279818424, 0.12130662023101241, 0.39367309061991507]

# parser('1_5 2_7'): list operators, generating lists
[1, 2, 3, 4, 5, 2, 3, 4, 5, 6, 7]

# parser('1_5/2'): you can do math on lists
[0.5, 1.0, 1.5, 2.0, 2.5]

# parser('1_5/2!!2'): and apply a transformer
[0.5, 0.5, 1.0, 1.0, 1.5, 1.5, 2.0, 2.0, 2.5, 2.5]

#  parser('1_5/2!![2,3]'): but the transformer can be a list
[0.5, 0.5, 1.0, 1.0, 1.0, 1.5, 1.5, 2.0, 2.0, 2.0, 2.5, 2.5]

#  parser('1_5/2!![2,3]|1'): because of |, it's either a long list or 1. 
[1.0]

# parser('1|2|3|4|5') # random picking
[2.0]

# parser('dada|baba|lala:2') # some operators work on names! 
['lala:1', 'lala:2', 'lala:3']
```
You get it, the pattern system can become quite complicated even if it only follows simple mathemetical rules. Here is a list of possible operators:

- math unary operators: `+2`, `-2`
- math binary operators: `+`, `-`, `*`, `/`
- `r` for a random number between `0.0` and `1.0`.
- parenthesis for complex precedence
- brackets and comma for lists
- **Sardine** binary operators:
    - `x!y`: replication, `y` times `x`.
    - `x!!y`: replication for lists, repeat `y` times each in `x`.
    - `x:y`: range, pick one between `x` and `y`.
    - `x|y`: choice. Choose one between `x` and `y`.
* **Sardine** list operators:
    - `x_y`: generate a ramp from `x` to `y`.

Try them and see if you find something interesting. Some operators might now always yield the same result depending on the result of chance based operators. Here are some patterns to get you started. I hope that they will help you to understand how all of this works:
```python
parser('1_3+(1_3|10)')
parser('r*8!4')
parser('[r,r,r]/[1,2,3]*2')
parser('1_5/2_5!!2')
```
#### A grammar for notes

There is another special grammar that you will only encounter for some specific arguments such as `note` or `midinote`. This grammar is specialized in writing note sequences, and notes can be written using the anglo-saxon or french system:

- `c d e f g a b`: anglo saxon.
- `do re mi fa sol la si`: french system.

A note is in fact a MIDI note number. `c` will yield `60`, the note C played at the 5th octave on a piano. Notes can receive the modifiers you expect them to receive:

- `#`: make a note sharp. You can stack sharps if you wish: `c####`.
- `b`: make a note flat. You can stack flats if you wish: `cbbbb`.
- `0-9`: octave modifier. You can write `c4`, `mi2` or any other note.
- `+`: make a note an octave higher. You can stack it as well: `c5++`.
- `-`: make a note an octave lower. You can stack it as well: `c5--`.
- `:`: feed a **chord qualifier**. They will turn a single note into an array of notes.

I dislike this latter feature quite a lot because it doesn't do anything to account for voice-leading, inversions or octave shifts! I keep it around because it can sometimes be useful. Here is a list of current chord qualifiers you can use:

```python
'dim'    : [0, 3, 6, 12],      'dim9'   : [0, 3, 6, 9, 14],
'hdim7'  : [0, 3, 6, 10],      'hdim9'  : [0, 3, 6, 10, 14],
'hdimb9'  : [0, 3, 6, 10, 13], 'dim7'   : [0, 3, 6, 9],
'7dim5'  : [0, 4, 6, 10],      'aug'    : [0, 4, 8, 12],
'augMaj7': [0, 4, 8, 11],      'aug7'   : [0, 4, 8, 10],
'aug9'   : [0, 4, 10, 14],     'maj'    : [0, 4, 7, 12],
'maj7'   : [0, 4, 7, 11],      'maj9'   : [0, 4, 11, 14],
'minmaj7': [0, 3, 7, 11],      '7'      : [0, 4, 7, 10],
'9'      : [0, 4, 10, 14],     'b9'     : [0, 4, 10, 13],
'mM9'    : [0, 3, 11, 14],     'min'    : [0, 3, 7, 12],
'min7'   : [0, 3, 7, 10],      'min9'   : [0, 3, 10, 14],
'sus4'   : [0, 5, 7, 12],      'sus2'   : [0, 2, 7, 12],
'b5'     : [0, 4, 6, 12],      'mb5'    : [0, 3, 6, 12],
```

#### Patterns in senders

Every parameter available with the `S()`, `M()` or `O()` object can be patterned. The **Sardine** grammar is automatically parsed and transformed to a list when used as a parameter of these objects. Take a look at the following example:

```python
@swim
def parametrized(d=1, i=0):
    S('cp:1_10', 
        cutoff= '1_10*100',
        speed='r*2 1 2',
        legato='0.5!2 0.2 0.8', 
        room='1_10/10', 
        dry=0.2).out(i)
    again(parametrized, d=0.5, i=i+1)
```

This is a very dynamic sequence thanks to the use of the **Sardine** grammar.  There are a few things you need to know before using the **Sardine** grammar in your own patterns. Patterns are parsed automatically as lists when they are declared as strings for each parameter. By default, **Sardine** will return the first element of the list because it doesn't know what value you would like to get out of the list!

The `.out()` method can receive an optional `iter` argument (the first for convenience). This argument will indicate what value you would like to get out of the list. You can't pick a value too far in your lists and you can use asymetrical lists of different lengths. **Sardine** will round your `iter` to pick the appropriate value just like the `itertools.cycle` would have done in regular Python. Try it by yourself to understand this mechanism better:

```python
# speed=1 because first in list and no iter
S('bd', speed='1 2 3 4').out()
# speed=3 because iter=2
S('bd', speed='1 2 3 4').out(2) # change me
```

In a **swimming function**, you might use an iterator to iterate on your list by recursion:

```python
@swim
def woohoo(d=0.5, i=0):
    S('bd', speed='1_10').out(i)
    again(woohoo, d=0.125, i=i+1) # change me
```

If you wish to iterate in the opposite direction, just count in the opposite direction:

```python
@swim
def woohoo(d=0.5, i=0):
    S('bd', speed='1_10').out(i)
    again(woohoo, d=0.125, i=i-1) # change me
```

If you wish to pick a random value out of your list, but only 25% of the time, do this:

```python
from random import randint
@swim
def woohoo(d=0.5, i=0):
    S('bd', speed='1_10').out(i)
    again(woohoo, d=0.125, 
            i=randint(1,20) if random() > 0.75 else i-1)
```

Combine the iteration system with the **Sardine** grammar for maximal fun.

#### Patterns everywhere

There are three objects that can be used to play with the patterning system everywhere, and not only with senders objects:

- `Pnum(pattern: str, i: int)`: for the number parser.
- `Pname(pattern: str, i: int)`: for the name parser.
- `Pnote(pattern: str, i: int)`: for the note parser.

These objects allow you to play with the patterning system everywhere in your **swimming functions**. It can be particularly interesting for generating rhythmic sequences:

```python
@swim
def bd(d=0.5, i=0):
    S('notes:1',
            room=0.5,
            speed='1 0.5 1 2',
            midinote='c5 e5 g5 a5').out(i)
    S('notes:1',
            speed='1 0.5 1 2',
            midinote='e5 g5 bb5 e6').out(i)
    a(bd, d=Pnum('1 0.5 0.5 2 0.5 0.5', i), i=i+1)
```

They are also great for investigating the different available grammars, and complement `parser()` and `parser_repl()` quite well!

## MIDI

**Sardine** supports all the basic messages one can send and receive through MIDI. It will support many more in the future, including SySex and custom messages. By default, **Sardine** is always associated to a default MIDI port. It can both send and receive information from that port. If the clock is `active`, you already know that clock messages will be forwarded to all your connected softwares/gear.

You can also open arbitrary MIDI ports as long as you know their name on your system. I will not enter into the topic of finding / creating / managing virtual MIDI ports. This subject is outside the scope of what **Sardine** offers and you are on your own to deal with this. A tutorial might come in the future. 

### MIDI Out

Here is an example of a **swimming function** sending a constant MIDI Note:

```python
@swim
def hop(d=0.5, i=0):
    M(delay=0.3, note=60, 
            velocity=127, channel=0).out()
    anew(hop, d=0.5, i=i+1)
```

The default MIDI output is accessible through the `M()` syntax (contrary to `S`, it is not an object!). As you will see, MIDI still need some improvements to support all messages throughout the use of the same object. Note that the channel count starts at `0`, covering a range from `0` to `15`. The duration for each every note should be written in milliseconds (`ms`) because MIDI is handling MIDI Notes as two separate messages (`note_on` and `note_off`). Following the MIDI standard, note and velocity values are expressed in the range `0-127`.

Let's go further and make an arpeggio using the pattern system:

```python
@swim
def hop(d=0.5, i=0):
    M(delay=0.3, note='60 46 50 67', 
            velocity=127, channel=0).out(i)
    anew(hop, d=0.5, i=i+1)
```

A similar function exists for sending MIDI CC messages. Let's combine it with our arpeggio:

```python
@swim
def hop(d=0.5, i=0):
    M(delay=0.3, note='60 46 50 67', 
            velocity=127, channel=0).out(i)
    cc(channel=0, control=20, value=randint(1,127))
    anew(hop, d=0.5, i=i+1)
```

### MIDI In

MIDI Input is supported through the use of a special object, the `MidiListener` object. This object will open a connexion listening to incoming MIDI messages. There are only a few types of messages you should be able to listen to:

* MIDI Notes through the `NoteTarget` object
* MIDI CC through the `ControlTarget` object

Additionally, you can listen to incoming Clock messages (`ClockListener`) but you must generally let Sardine handle this by itself. There are currently no good reasons to do this!

Every `MidiListener` is expecting a target. You must declare one and await on it using the following syntax:

```python
a = MidiListener(target=ControlTarget(20, 0))
@swim
def pluck(d=0.25):
    S('pluck', midinote=a.get()).out()
    anew(pluck, d=0.25)
```

In this example, we are listening on CC n°20 on the first midi channel (`0`), on the default MIDI port. Sardine cannot assert the value of a given MIDI Control before it receives a first message therefore the initial value will be assumed to be `0`.

You can fine tune your listening object by tweaking the parameters:

```python
# picking a different MIDI Port
a = MidiListener('other_midi_port', target=ControlTarget(40, 4))
```

## OSC

You can send OSC (**Open Sound Control**) messages by declaring your own OSC connexion and sending custom messages. It is very easy to do so. Two methods are available depending on what you are trying to achieve.

### Manual method


The following example details the simplest way to send an OSC message using Sardine:

```python
from random import randint, random, chance

# Open a new OSC connexion
my_osc = OSC(ip="127.0.0.1",
        port= 23000, name="Bibu",
        ahead_amount=0.25)

# Recursive function sending OSC
@swim
def custom_osc(d=1):
    my_osc.send(c, '/coucou', [randint(1,10), randint(1,100)])
    anew(custom_osc, d=1)

# Closing and getting rid of the connexion

cr(custom_osc)

del my_osc
```

Note that you **always** need to provide the clock as the first argument of the `send()` method. It is probably better to write a dedicated function to avoid having to specify the address everytime you want to send something at a specific address:

```python
def coucou(*args): my_osc.send(c, '/coucou', list(args))
```
### Using the OSCSender object

It is not recommended to send messages using the preceding method. You will not be able to patern parameters and addresses. Prefer the `OscSender` object, aliased to `O()`. The syntax is similar but you gain the ability to name your OSC parameters and you can use patterns to describe them (more on this later on) Compared to `S()` and `M()`, `O()` requires more parameters:
* `clock`: the clock, `c` by default.
* `OSC` object: the `OSC` connexion previously defined.

```python
# Open a new OSC connexion
my_osc = OSC(ip="127.0.0.1", port=23000, name="Bibu", ahead_amount=0.25)

# Simple address
O(my_osc, 'loulou', value='1 2 3 4').out()

# Composed address (_ equals /)
O(my_osc, 'loulou_yves', value='1 2 3 4').out()

@swim
def lezgo(d=1, i=0):
    O(my_osc, 'loulou_blabla', 
        value='1 2 3 4', 
        otherv='1 2|4 r*2').out(i)
    anew(lezgo, i=i+1)
```


## Crash

If you already know how to program, you know that 90% of your time is spent debugging code that doesn't work. Crashes will happen when you will play with **Sardine** but they are handled and taken care of so that the musical flow is never truly interrupted. If you write something wrong inside a **swimming function**, the following will happen: 

- if the function crashes and has never looped, it will not be recovered.
- if the function is already running and has already looped, the last valid function will be rescheduled and the current error message will be printed so that you can debug.

It means that once you start playing something, it will never stop until you want it to. You can train without the fear of crashes or weird interruptions.