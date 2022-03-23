---
layout: post
title: Python Challenge - Talking Alarm
published: true
category: development
---

# Implementation
This program's purpose is to convert a textual 24-hour time format into a verbal statement of time, by both text and voice.

For example:
```
01:30 -> "It's one thirty am"
04:15 -> "It's four fifteen am"
```

This Python program:
* Accepts user input in the form of 24-hour time (i.e., "01", "30")
* Leverages libraries and custom logic to convert the time to a verbal statement in text (i.e., "It's one thirty am")
* Leverages text-to-speech libraries to give the alarm a voice, playing via the system MP3 codecs

{% gist cb8743c66c39ec25b3b9f7dc5aacf0db %}