summary: How to Write a Codelab
id: how-to-write-a-codelab
categories: Sample
tags:
status: Published
authors: Victor Akinyemi
feedback link: https://dscfuta.com

# How to Write a Codelab

<!-- ---------------------- -->

## Overview

Duration: 1

### What you'll learn

- how to set the amount of time each slide will take to finish
- how to include code snippets
- how to hyperlink items
- how to include images
- other stuff

<!-- ---------------------- -->

## Setting Duration

Duration: 2

To indicate how long each slide will take to go through, set the `Duration` under each Heading 2 (i.e. `##`) to an integer.
The integers refer to minutes. If you set `Duration: 4` the a particular slide will take 4 minutes to complete.

The total time will be automatically calculated for you and will be displayed on the codelab once you create it.

<!-- ---------------------- -->

## Code Snippets

Duration: 3

To include code snippets you can do a few things.

- Inline highlighting can be done using the tiny tick mark on your keyboard: "`"
- Embedded code

### JavaScript

```javascript
function scream({ words }) {
  alert(words.toUpperCase());
}

scream({ words: "hey there, how are you" });
```

<!-- ---------------------- -->

## Hyperlinking and Embedded Images

### Hyperlinking

[Super awesome coding tutorials](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

### Images

![Wide random image](assets/landscape.jpg)

<!-- ---------------------- -->

## Other Stuff

Duration: 1

Once you're done writing your codelab, export is as HTML using `claat export FILENAME.md`, then run `grunt serve` to preview it. You may also leave `grunt serve` running and re-export your codelab using `claat` after making changes, then refresh your browser.

Checkout the official documentation here: [Codelab Formatting Guide](https://github.com/googlecodelabs/tools/blob/master/FORMAT-GUIDE.md)

The markdown formatting guide can be found here: [Markown Parser](https://github.com/googlecodelabs/tools/blob/claat/parser/md/README.md)
