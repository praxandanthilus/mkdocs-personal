---
draft: false 
date: 2026-02-11 
categories:
  - Docs as Code
  - Doc Hacks
  - Content Reuse
  - Python
  - MkDocs
---

# Docs as Code: Who Said You Couldn't Reuse Content?

One of the major downsides I often hear with regards to the Docs as Code methodology is that it doesn't have a native or easy way to reuse content. After all, having a single source of truth, especially for developer reference, must be a requirement for any technical documentation solution. If reuse isn't a feature, then a CMS or other traditional doc environment must be better.

Hogwash. The very thing that makes docs as code great for version control & developer documentation also makes this "problem" easy to solve. In about 20 minutes I created and tested a script in Python that takes advantage of a few existing libraries and MkDocs' native integration with Jinja2 to define a simple directive to embed content from any where in your repo (not just docs!) to one or more destination documents.

Now, `pymdownx-snippets` and symlinks do exist, but the former isn't recommended for user-facing docs, and the latter can get messy very quick (and many devs don't like using them).

## MkDocs macros-plugin to the rescue

I love MkDocs for a lot of reasons, but mainly it's because the plugins/extensions that have been developed for it, especially for [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/), meet my needs whenever I go looking for a feature I'm interested in or a problem I need to solve. The [MkDocs Macros plugin](https://mkdocs-macros-plugin.readthedocs.io/en/latest/), according to its creators, is actually more like a "mini-framework," meaning that it opens MkDocs up to a lot more functionality by the integrations that it leverages. What I ended up discovering was a way to take advantage of its Jinja2 integrations, specifically with respect to templates.

The default behavior allows you to add a directive to your Markdown to include the contents of an external file in pages rendered with MkDocs. While this is useful as a jumping off point, it wasn't exactly what I wanted. I had a specific use case where I needed to extract just a portion of content from the source file as an excerpt, rather than the entire contents of the file itself (ironically because I was using placeholders and front matter for a different macros plugin usage that I didn't want to include here).

Thankfully, since MkDocs and most of its plugins are Python-based, its easy to implement a customization as ... a macro! The main thing I needed was the ability to include an excerpt from a source `.md` file in one or more destination files, and I needed to be able to control that excerpt and its look-and-feel in the rendered site in the following ways:

- Identify a specific range of lines of code/markdown to encompass the excerpt (for example, I need lines 9-35, but nothing before or after).
- Apply a specific level to any headings appearing in the excerpt, so that the ToC in the generated site had them placed at the proper levels. For example, the excerpt might be taken from a doc with H1/H2, but needs to be placed under an H2 heading in the destination document, so the minimum level of headings in the excerpt would need to be rendered as an H3.
- In my case, the source markdown file was in a separate directory from the destination file, so if there were any images or other attachments in the source, they would become broken in the destination. So I also needed a way to fix the URL pointing to the image that would allow it to render in the source and destination documents regardless of where they lived in the repo.

Sounds like a lot, right? But really, most of that functionality existed already, hidden inside Python libraries and Jinja2 functionality. So here's how I put it all together.

### Define a Range

I decided to use the `itertools` `iSlice` helper function first, to allow me to declare a range of lines to mark the boundaries of the excerpt. I wanted to just be able to use whole integers corresponding to line numbers to determine that range.

```

from mkdocs_macros import iSlice

def define_env(env):
    @env.macro
    def excerpt(path, start, end):
        """
        Extract lines from `start` to `end` (1‑based, inclusive).
        Usage:
            {{ excerpt("path/to/file.md", 10, 25) }}
        """
        slicer = iSlice(path)
        return slicer(start=start, end=end)

```

### Normalize Headings

Next, I needed a way to normalize headings in the excerpted content. This involved having the script look in the Markdown file for headings, then shift/adjust those headings based on a value specified in the directive (for example, `heading=3`).

```

from mkdocs_macros import iSlice
import re

def define_env(env):

    @env.macro
    def excerpt(path, start, end, heading=None):
        """
        Extract lines from `start` to `end` and optionally normalize
        heading levels so the smallest heading becomes `heading`.

        Usage:
            {{ excerpt("file.md", 10, 25) }}
            {{ excerpt("file.md", 10, 25, heading=3) }}
        """
        slicer = iSlice(path)
        text = slicer(start=start, end=end)

        if heading is None:
            return text

        return _adjust_headings(text, heading)


def _adjust_headings(text, base_level):
    """
    Shift all Markdown headings so that the smallest heading level
    becomes `base_level`.
    """
    lines = text.splitlines()

    # Find the minimum heading level in the excerpt
    heading_pattern = re.compile(r'^(#+)\s+')
    levels = []

    for line in lines:
        m = heading_pattern.match(line)
        if m:
            levels.append(len(m.group(1)))

    if not levels:
        return text  # no headings to adjust

    min_level = min(levels)
    shift = base_level - min_level

    adjusted = []
    for line in lines:
        m = heading_pattern.match(line)
        if m:
            hashes = '#' * (len(m.group(1)) + shift)
            line = heading_pattern.sub(hashes + ' ', line)
        adjusted.append(line)

    return "\n".join(adjusted)

```
    
### Fix Relative URLs

Lastly, to ensure that images and other relative URLs to assets a rendered in the site, regardless of destination, I used the `macros-plugin` native method `fixURL()` to render that image in both locations.

---

Note: Modifying URLs in this way does unfortunately cause them to cease rendering in Github's Markdown preview, but obviously this functionality assumes that people are reading the docs on the rendered site.

---

```

from mkdocs_macros import iSlice
import re
import os

def define_env(env):

    @env.macro
    def excerpt(path, start, end, heading=None):
        """
        Extract lines from `start` to `end`, optionally normalize heading
        levels, and fix relative URLs so images/links still work.
        """
        slicer = iSlice(path)
        text = slicer(start=start, end=end)

        # Normalize headings if requested
        if heading is not None:
            text = _adjust_headings(text, heading)

        # Fix relative URLs (images, links)
        src_path = path
        dest_path = env.page.file.src_path
        text = env.fix_url(text, src_path, dest_path)

        return text


def _adjust_headings(text, base_level):
    """
    Shift all Markdown headings so that the smallest heading level
    becomes `base_level`.
    """
    lines = text.splitlines()
    heading_pattern = re.compile(r'^(#+)\s+')
    levels = []

    # Find minimum heading level
    for line in lines:
        m = heading_pattern.match(line)
        if m:
            levels.append(len(m.group(1)))

    if not levels:
        return text

    min_level = min(levels)
    shift = base_level - min_level

    adjusted = []
    for line in lines:
        m = heading_pattern.match(line)
        if m:
            hashes = '#' * (len(m.group(1)) + shift)
            line = heading_pattern.sub(hashes + ' ', line)
        adjusted.append(line)

    return "\n".join(adjusted)

```
### Don't forget to modify mkdocs.yml

Make sure to add the following to your `mkdocs.yml` config to call the script when the site is served or built.

```

plugins:
  - macros:
      modules:
        - macros/main.py

```

The common file layout for the above would be:

```

mkdocs.yml
docs/
macros/
    main.py

```

Obviously your script can be named anything and the folder path can be wherever you want it relative to the main docs folder. Just make sure the YAML is correctly defined. 

### The completed directive

Now you can include a directive in any Markdown file to include an excerpt you specify.

In the below example, your excerpt comes from `file.md`, specifically the content from lines **10 - 25**, and any headings in the excerpt begin at **level 3**. You do not need to add a `fixURL()` directive, URLs will be fixed automatically.

## In Conclusion

Yes, there's some manual work involved in reusing content in Docs as Code, but hopefully this illustrates the power of having docs close to and usable with code, and a little more knowledge (and some vibe coding) can go a long way to having your docs as code and eating it, too!

