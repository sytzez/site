---
title: "Responsive sheet music in Hugo using MusicXML"
date: 2022-10-08
draft: true
tags: [ "hugo", "music", "blogging" ]
---

### Introduction

When writing blogs about music, you might want to render snippets of music notation in between paragraphs of text.
I will show you how to easily embed [MusicXML](https://www.musicxml.com/) content within your [Hugo](https://gohugo.io/) posts.

MusicXML is the most common format of exchanging musical notation information between different programs.
Whatever musical notation software you use, chances are very high they have an "export to MusicXML" functionality.
Examples include [Sibelius](https://www.avid.com/sibelius), [Finale](https://www.finalemusic.com/), [noteflight](https://www.noteflight.com/) and many others.
If you want to use a free, fully functional program, I recommend downloading [MuseScore](https://musescore.org/).

To render your exported MusicXML in Hugo,
we will be using a javascript library called [OpenSheetMusicDisplay](https://opensheetmusicdisplay.org/). 
It reads MusicXML and renders it as an SVG using the widely used [VexFlow](https://www.vexflow.com/) library.

### 1. Add scripts to the header

To start, add the following scripts to your header:

{{< highlight html >}}
<script id="osmd-script" async src="https://cdn.jsdelivr.net/npm/opensheetmusicdisplay@1.5.7/build/opensheetmusicdisplay.min.js"></script>
<script>
  // Wait for the DOM to completely load.
  document.addEventListener('DOMContentLoaded', function () {
    // Wait for the async osmd script to load.
    document.getElementById('osmd-script').addEventListener('load', function () {
      // Iterate through all elements with class 'osmd-container'.
      document.querySelectorAll('.osmd-container').forEach(function (container) {
        // Create an OpenSheetMusicDisplay object for the container.
        var osmd = new opensheetmusicdisplay.OpenSheetMusicDisplay(container, {
          // Use minimal spacing and hide the title and instrument names.
          drawingParameters: 'compacttight'
        })

        // Load the MusicXML file.
        osmd.load(container.dataset.musicXmlSrc)
          .then(function () {
            // Render the notation inside the container.
            osmd.render()
          })
      })
    })
  })
</script>
{{< /highlight >}}

It loads the OpenSheetMusicDisplay script asynchronously.
After that script has been loaded and the DOM has finished rendering, it goes through all elements with the `osmd-container` class.
For each of those containers, it creates a new OpenSheetMusicDisplay object and renders the MusicXML content inside it.

### 2. Create the music-xml shortcode

To populate a page with `osmd-container` classes we can create a Hugo shortcode by adding a file called
`layouts/shortcodes/music-xml.html` with the following contents:

{{< highlight html >}}
<div class="osmd-container" data-music-xml-src="{{ .Get 0 }}"></div>
{{< /highlight >}}

### 3. Use the shortcode inside your content

The shortcode can be used from within any Hugo content or layout file:

{{< highlight html >}}
{{</* music-xml "/music-xml/bla.mxl" */>}}
{{< /highlight >}}

Simply provide the local or global path to a MusicXML file.

### The result

Here's a live example.
Note how the division of the bars adjusts when you shrink your screen width, it's truly responsive!

I like to add an `<audio>` tag with an mp3 file under the notation to provide playback.

{{< music-xml "https://downloads2.makemusic.com/musicxml/MozaVeilSample.xml" >}}

<audio controls>
    <source src="/music-xml/bla.mp3" type="audio/mpeg">
</audio>