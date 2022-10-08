---
title: "Responsive sheet music in Hugo using MusicXML"
date: 2022-10-08
draft: true
tags: [ "hugo", "music", "blogging" ]
---

When writing blogs about music, you might want to render bits of music notation in between paragraphs of text.
I will show you how to easily embed [MusicXML](https://www.musicxml.com/) content within your [Hugo](https://gohugo.io/) posts.

MusicXML is the most common format of exchanging musical notation information between different programs.
Whatever musical notation software you use, chances are very high they have an "export to MusicXML" functionality.
Examples include [Sibelius](https://www.avid.com/sibelius), [Finale](https://www.finalemusic.com/), [noteflight](https://www.noteflight.com/) and many others.
If you want to use a free, fully functional program, I recommend downloading [MuseScore](https://musescore.org/).

To render your exported MusicXML in Hugo,
we will be using a javascript library called [OpenSheetMusicDisplay](https://opensheetmusicdisplay.org/). 
It reads MusicXML and renders it as an SVG using the widely used [VexFlow](https://www.vexflow.com/) library.

To start, add the following scripts to your header:

```html
<script id="osmd-script" async src="https://cdn.jsdelivr.net/npm/opensheetmusicdisplay@1.5.7/build/opensheetmusicdisplay.min.js"></script>
<script>
  // Wait for the async osmd script to load.
  document.getElementById('osmd-script').addEventListener('load', function () {
    // Wait for the DOM to completely load.
    document.addEventListener('DOMContentLoaded', function () {
      // Iterate through all elements with class 'osmd-container'.
      document.querySelectorAll('.osmd-container').forEach(function (container) {
        // Create an OpenSheetMusicDisplay object for the container.
        var osmd = new opensheetmusicdisplay.OpenSheetMusicDisplay(container)

        // Parse the MusicXML from the container.
        osmd.load(container.innerHTML)
          .then(function () {
            // Render the notation inside the container.
            osmd.render()
          })
      })
    })
  })
</script>
```

It loads the OpenSheetMusicDisplay script asynchronously.
After that script has been loaded and the DOM has finished rendering, it goes through all elements with the `osmd-container` class.
For each of those containers, it creates a new OpenSheetMusicDisplay object and renders the MusicXML content inside it.

To populate a page with `osmd-container` classes we can create a Markdown codeblock by creating a file called
`layouts/_default/_markup/render-codeblock-music-xml.html` with the following contents:

```html
<div class="osmd-container">
    {{- .Inner | safeHTML }}
</div>
```

We can use the codeblock like this inside any Markdown file in your Hugo project:

````markdown
```music-xml
    bla
```
````

The result will look like this:

```music-xml
test
```
