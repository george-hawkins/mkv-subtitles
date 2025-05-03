MKV subtitles and vobsub in particular
======================================

I was watching a Japanese movie in `.mkv` format and the only included English subtitles had included content for the deaf ("crowd gasps" and the like) and this quickly gets tiring.

So, this is how I extracted the subtitles. And then as the subtitles were in vobsub format, which is image based rather than text, I needed to convert them to a text `.srt` file that I could then edit and remove the content for the deaf.

To work with `.mkv` you need to install `mkvtoolnix`:

```
$ brew install mkvtoolnix
```

Then you can look at your `.mkv` file:

```
$ $ mkvmerge -i 'Paprika (2006).mkv' 
File 'Paprika (2006).mkv': container: Matroska
Track ID 0: video (HEVC/H.265/MPEG-H)
Track ID 1: audio (AAC)
Track ID 2: audio (AAC)
Track ID 3: subtitles (VobSub)
Track ID 4: subtitles (VobSub)
Track ID 5: subtitles (VobSub)
Track ID 6: subtitles (VobSub)
Track ID 7: subtitles (VobSub)
Track ID 8: subtitles (VobSub)
...
```

Now you know that traks 3 and on are subtitles but you have to use the more verbose `mkvinfo` and search for `Language.*en`, i.e. the English one:

```
$ mkvinfo 'Paprika (2006).mkv' | less
    | + Track
    |  + Track number: 4 (track ID for mkvmerge & mkvextract: 3)
                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |  + Track UID: 824028482569575953
--> |  + Track type: subtitles
    |  + "Lacing" flag: 0
--> |  + Codec ID: S_VOBSUB
    |  + Codec's private data: size 348
--> |  + Language (IETF BCP 47): en
    |  + Content encodings
    |   + Content encoding
    |    + Content compression
```

Note: I first just read `Track number: 4` and used `4` but as you see, it then goes on to show the track ID to use with `mkvextract` etc., i.e. `3`.

So, you want the `subtitles` track with language `en`, if it shows as `S_VOBSUB` (as it does here), it means its DVD subtitles encoded as an image stream.

Now, extract the subtitles - specify the track (in this case `3`), a `:` and a base name for the subtitle files (in this case `paprika_en`):

```
$ mkvextract tracks 'Paprika (2006).mkv' 3:paprika_en
```

This is relatively slow but at least you get a progress percentage as it goes.

As we're extracting vobsub files, you'll end up with:

```
$ ls paprika_en.*
paprika_en.idx    paprika_en.sub
```

There are many tools to convert these into `.srt` but note that they depend on OCR and some do a better job than others.

I tried [`vobsubocr`](https://github.com/elizagamedev/vobsubocr). It's not paricularly popular but looking at the code, it seems like the task is relatively simple and no rocket science is needed (the hard work is done, i.e. the OCRing, is done by Tesseract)

To install Rust:

```
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

It add a line to your `.bashrc` to set up its environment so either open a new terminal or just run this setup manually in your current shell:

```
$ source ~/.cargo/env
```

Then I just installed `vobsubocr`:

```
$ cargo install --git https://github.com/elizagamedev/vobsubocr
```

Then I ran the tool, specifying `paprika_en.idx` (and it assumes there's a corresponding `.sub` file):

```
$ vobsubocr -l eng -c tessedit_char_blacklist='|\/`_~' -o paprika_en.srt paprika_en.idx
```

Note: the `tessedit_char_blacklist`, this is a list of blacklisted characters that are unlikely to validly occur in the subtitles and without it, Tesseract sometimes e.g. interpreted `I` as `|`.

Even with this blacklist, it often mixed up `!` and `I`. The `!` appears in lots of places validly but I would have hoped given that Tesseract has to be told what language is being converted that it would e.g "know" that `!` generally only appears in very particular situations and definitely not at the start of sentences. I manually, cleaned up all the `!` issues (oddly, it never seemed to mistake an `!` for an `I` - it was always only the other way around). And `1` and `I` are also frequently confused.

This is surprisingly quick - clearly OCRing isn't a taxing job for modern computers.

I then edited the file to remove bits for the deaf (the whole point of this exercise).

There were three main patterns to look for and remove:

* `[in English] It's the greatest show time!`
* `-[crowd gasps, cheers]`
* `-[Atsuko] You mean you opened it.`
* `[crowd gasping]`
* `[Paprika] There's someone who keeps`

Note: in all case, I never deleted a line, I just deleted character, sometimes leaving completely blank lines - whether this is the correct approach, I don't know - but I didn't want to change the structure of the file.

Things like `[in English] ...` tell you that the character is actually speaking English (even though in this case, it's a Japanese movie), so these lines can be removed completely (I just did a case-insensitive search for "english" and removed these few lines - well not removed entirely - as noted, I just removed the characters on the relevant lines).

The `-` seems to be used to indicate that the noise is coming from a new source (whether the dialog is switching between people or e.g. a car makes a noise and then an animal does).

I didn't want to remove the `-` in the case where somebody is saying something, e.g. `-[Atsuko] You mean...` but did where it's nothing other than a sound, e.g. `-[crowd gasps, cheers]`

So, to remove things that were nothing more than sounds, I did:

```
:%s/-\[.*\]\s*$/
```

And then for everything else, I did:

```
:%s/\[.*\]\s*/
```

That's it.
