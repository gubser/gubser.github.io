---
title:  "Live rendering of piano keys to sheet music in the browser"
comments: true
---

# Live rendering of piano keys to sheet music in the browser
![Random Rachmaninov Chord](/assets/2019-09-19/chord.png)

Ha! There is a Working Draft for interfacing with [MIDI devices in the browser](https://www.w3.org/TR/webmidi/). That is so cool.
Unfortunately, only Chrome (and Chromium) has support for it yet.

Web MIDI API gives you low-level access to the midi messages stream for any MIDI controller connected to your computer. Today, most E-Piano or Keyboards come with an USB port.

I've written a simple website with vanilla JavaScript that uses the marvellous [vexflow](http://www.vexflow.com/) library to render the notes you press live on sheet music.

Try it out here: [https://eliogubser.com/chord/](https://eliogubser.com/chord/) 

Here's a video of it in action:
<div style="padding:56.25% 0 0 0;position:relative;"><iframe src="https://player.vimeo.com/video/361015939?color=ffffff&title=0&byline=0&portrait=0" style="position:absolute;top:0;left:0;width:100%;height:100%;" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe></div><script src="https://player.vimeo.com/api/player.js"></script>

Source Code:
[https://github.com/gubser/chord/blob/master/chord.js](https://github.com/gubser/chord/blob/master/chord.js)
