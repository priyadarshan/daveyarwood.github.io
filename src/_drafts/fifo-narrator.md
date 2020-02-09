---
layout: post
title: "The FIFO-controlled, text-to-speech narrator"
tags:
  - unix
  - text to speech
  - sox
  - audio
  - i3
  - command line
published: true
---

{% include JB/setup %}

I'm very proud to be married to the amazing modern dance choreographer and
educator [Renay Aumiller][renay-aumiller]. From time to time, Renay and I
collaborate to produce works of modern dance accompanied by original music. Most
recently, we created a piece called _By Chance_ that involves feeding
suggestions from the audience into a web application that I wrote to produce a
randomly generated series of short dance vignettes.

This piece was originally called _Out of the Blue_ because we first performed it
in a room in a warehouse space with dark blue walls. In [a previous blog
post][out-of-the-blue], I talked about my process as I was creating the music,
and I shared some excerpts.

An interesting aspect of the piece that I haven't touched on yet is that it
features narration audio that I generated by using [text-to-speech][tts-wiki]
synthesis. Renay's idea was that she would converse with a robotic-sounding
narrator while she performed, which would both highlight the relationship
between humans and computers and add a little bit of humor to the performance.
To make the pace of the conversation sound more natural, I would cue each
portion of dialogue from my laptop.

# `say`

I had used the `say` command in the past to generate text-to-speech audio. The
`say` that comes with macOS is great. But I've been developing in an Ubuntu
environment for several years now, and I was curious to explore the
text-to-speech options available for Linux.

A while back, I installed some audio package (I forget which one) via `apt` and
unbeknownst to me, one of the package's dependencies was [GNUstep], a software
bundle that provides a `say` command, among other things.

I gave GNUstep's `say` a try, and the results were underwhelming compared to the
Apple version:

{% highlight bash %}
say "I am a terrible speech synthesizer"
{% endhighlight %}

<center>
  <p>
    <figure>
      <figcaption>
      Audio output:
      </figcaption>
      <audio controls src="{{ site.url }}/assets/2020-01-25-gnustep-say.mp3"></audio>
    </figure>
  </p>
</center>

It just didn't sound natural enough. I think it's wonderful that the Free
Software Foundation implemented an OSS replacement for the macOS `say` command,
but in an era where children grow up talking to natural-sounding TTS
personalities like Siri and Alexa, GNUstep's `say` can't help but sound robotic
and corny by comparison.

# Google Text-to-Speech

As I was looking around for other options, I saw that Google Cloud has a
[Text-to-Speech][google-tts] product. The speech quality is excellent, but it is
a paid product that requires a Google Cloud account if you want to use the
Text-to-Speech API directly.

It turns out, however, that [Google Translate][google-translate] uses the same
Text-to-Speech engine (albeit with less options), and you can use it out of the
box with a lot less ceremony (and for free, to boot).

There are multiple command-line tools available that implement this method of
using Google Text-to-Speech via Google Translate. I ended up using [this
one][desbma-google-speech]. Here's what it sounds like:

{% highlight bash %}
google_speech -l en-us "I'm a good speech synthesizer"
{% endhighlight %}

<center>
  <p>
    <figure>
      <figcaption>
      Audio output:
      </figcaption>
      <audio controls src="{{ site.url }}/assets/2020-01-25-google-speech.mp3"></audio>
    </figure>
  </p>
</center>


# Narration files

I collected the chunks of narrated text in a bunch of simple, plain text files
in a directory:

{% highlight bash %}
$ ls -1 narration/
02-introduction.txt
03-description.txt
04-collecting-input-01.txt
05-collecting-input-02.txt
07-ready-to-start.txt
08-starting.txt
09-the-end.txt
{% endhighlight %}

Each file contains anywhere from 1 to 119 lines of narration, depending on how
much the narrator has to say in that section. Each line of the file is either
empty or it contains an isolated sentence or phrase. I noticed that by inserting
empty lines, I could introduce pauses. This was useful because, as Renay and I
rehearsed her dialogue with the TTS narrator, we often found that we needed to
adjust the amount of pause time between the narrator's utterances, and doing so
was as easy as adding or removing empty lines.

I put together a quick Bash script to narrate every line of every text file, in
order, by piping each line through `google_speech`:

{% highlight bash %}
find narration/ | sort | tail -n+2 | while read filename; do
  cat "$filename" | while read line; do
    google_speech $gs_opts "$line"
  done
done
{% endhighlight %}

This worked great, but there was another requirement: we needed to be able to
pause for a variable amount of time in between narration files, and continue
only when I send a cue.

That's where the FIFO comes in.

# The FIFO

There is a neat feature of Unix called [named pipes][named-pipe], which are also
commonly known as FIFOs because of their _first in, first out_ behavior (not to
mention the fact that you can create them by using a command called `mkfifo`).

A named pipe is a special type of file that can be used like a
[pipe][unix-pipes]. You can put things into it and take things out of it in a
_first in, first out_ fashion.

The best way to understand how named pipes work is to try them yourself. You'll
need two terminals for this. In the first terminal, create a FIFO and use `cat`
to print the first data that comes out of the pipe:

{% highlight text %}
mkfifo the-fifo
cat the-fifo
{% endhighlight %}

You'll notice that the terminal "hangs" after the `cat` command. That's because
there is nothing available to print just yet.

Now, in another terminal, put some data into the pipe:

{% highlight text %}
echo "hello, fifo" > the-fifo
{% endhighlight %}

You'll see `hello, fifo` printed in the first terminal. Nice job, you just moved
data from one terminal to another!

When you're done, you can remove the FIFO with `rm`, just like any other file.

<center>
<img src="{{ site.url }}/assets/2020-01-29-fifo-demo.gif"
     title="Basic usage of a FIFO"
     width="75%">
</center>

In addition to shoveling data from one process to another, you can also use a
FIFO to send signals. One process receives continuously from the FIFO, and it
does work of some kind whenever it receives a signal.

This gif illustrates the concept:

<center>
<img src="{{ site.url }}/assets/2020-01-31-fifo-signaling.gif"
     title="Using a FIFO to send signals"
     width="75%">
</center>

"Bang" is a term that I took from the multimedia signal processing language
[Pure Data][pure-data]. A "bang" is a signal with no content that is used to
control timing.

# The FIFO narrator

It occurred to me that I could use this type of signaling to cue the TTS
narration blurbs in _By Chance_. I wrote a little `fifo-narrator` script that
receives "bangs" from a FIFO in a loop, and each time it receives a "bang," it
narrates the next file in the `narration` directory:

{% highlight bash %}
fifo="narration.fifo"
rm -f $fifo
mkfifo $fifo

i=1

while true; do
  filename="$(find narration/ | tail -n+2 | sort | sed -n "${i}p")"
  if [[ -z "$filename" ]]; then break; fi

  echo -------------------------
  echo "Waiting for a \"bang\" on: $fifo"
  echo -------------------------
  cat $fifo >/dev/null

  cat "$filename" | while read line; do
    google_speech $gs_opts "$line"
  done

  ((++i))
done
{% endhighlight %}

To send the signals, I set up an [i3] keybinding that puts the word `BANG`
into the FIFO whenever I press the `mod` key and `;` at the same time:

{% highlight bash %}
bindsym $mod+semicolon \
  exec --no-startup-id \
  "[ -p ~/code/by-chance/narration.fifo ] \
     && echo BANG > ~/code/by-chance/narration.fifo"
{% endhighlight %}

And with that, I was able to control the length of the pauses, allowing the
narration to proceed by pressing `mod + ;`.

# Success?

Our first performance of _By Chance_ (at the time called _Out of the Blue_) in
January 2019 was a smash success! We performed in a warehouse space where we set
up the lighting and sound system ourselves. I ran the sound from my laptop out
of the headphone jack and into a small bass amplifier, which worked flawlessly.
We were the only performers in that particular part of the warehouse, which
meant that we were able to set everything up for the tech rehearsals, leave it
in place for the three performances that weekend, and double-check right before
each performance that everything was still working. All three performances went
off without a hitch! Renay danced wonderfully, the crowds were engaged with the
idea and they really seemed to enjoy it.

We performed the piece again in November, but unfortunately, it was marred by
technical difficulties. Partway through the introductory text-to-speech
narration, the narrator stopped unexpectedly. In a panic, I flipped over to the
terminal window where my `fifo-narrator` script was running, and there I was
presented with a long Python stacktrace. Someone in the crowd went "hey, that's
Python!" Being already in the middle of a performance, Renay had to improvise
and stumble onward without having a narrator to converse with. Afterward, she
was furious with me.

* I hadn't thought about the fact that you need to have access to the Internet
  in order to use the Google Text-to-Speech.
* Incidentally, it turns out that the google_speech CLI tool works offline by
  caching the audio synthesized from text input that you've handed to it
  before.
  * Performance #1 had (luckily) gone off without a hitch despite my laptop
    not being connected to the internet at the time. The text-to-speech audio
    was being played back from the cache from when we had rehearsed at home.
  * Somehow, during performance #2, I think the cache failed. (Maybe it was
    a TTL cache and it just happened to expire at the worst possible time?
    Who knows?)
* It became clear that our performance should not depend on the reliability of
  a network connection or the availability of the Google Text-to-Speech
  service.
* A better approach would be to pre-record the text-to-speech narration and
  simply play it back during the performance. I could use the same FIFO setup
  to cue moving from each audio file to the next.

* Rough calculation of the pauses between utterances in each script file, based
  on the number of consecutive blank lines.

* SoX
  * man page: "Sound eXchange, the Swiss Army knife of audio manipulation"
  * reads and writes audio files
  * actually does a lot more than that, and I've barely scratched the surface of
    what it can do
    * maybe use some of the examples in the man page, which look interesting
  * I'm using it for two things:
    * generate wav files for specific durations of silence
      * `sox -n -r 24k -c 1 "$part_filename" trim 0.0 "$duration"`
    * stitch together TTS mp3s padded by the silent wav files to add pauses
      * `sox $(ls $parts_dir/*) "$mp3_filename"`


* Performance #3 (January 2020) went more smoothly.

# Comments?

Reply to [this tweet][tweet] with any comments, questions, etc.!

[tweet]: https://twitter.com/dave_yarwood/status/FIXME

[renay-aumiller]: https://www.radances.com
[out-of-the-blue]: {% post_url 2019-07-15-out-of-the-blue %}
[tts-wiki]: https://en.wikipedia.org/wiki/Speech_synthesis
[GNUstep]: http://wiki.gnustep.org/index.php/User_FAQ#What_is_GNUstep.3F
[google-tts]: https://cloud.google.com/text-to-speech/
[google-translate]: https://translate.google.com/
[desbma-google-speech]: https://github.com/desbma/GoogleSpeech
[named-pipe]: https://en.wikipedia.org/wiki/Named_pipe
[unix-pipes]: https://en.wikipedia.org/wiki/Pipeline_(Unix)
[pure-data]: https://puredata.info/
[i3]: https://i3wm.org/
[black-box-theater]: https://en.wikipedia.org/wiki/Black_box_theater