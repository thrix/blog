---
title: "Fixing Emoji Fonts in Chrome on Fedora Atomic Desktops"
date: 2026-02-17
draft: false
tags: ["linux", "fedora-atomic", "bluebuild", "fonts", "fontconfig", "fedora"]
summary: "How I debugged and fixed missing emoji rendering in Google Chrome on a BlueBuild/Fedora Atomic Desktops system — and why fontconfig wasn't the answer."
---

I recently noticed that emoji characters stopped rendering in Google Chrome on
my Fedora Atomic Desktops workstation. Instead of colorful emoji, I was getting
blank boxes or monochrome glyphs. Here's how I debugged it, went down a
fontconfig rabbit hole, and eventually found the real fix.

## My setup

I run a custom [BlueBuild](https://blue-build.org/) image based on
`sway-atomic` (Fedora Atomic Desktops with Sway). The image installs
`fontawesome-fonts-all` (used by Waybar for status bar icons) and Google
Chrome, which pulls in `google-noto-color-emoji-fonts` as a dependency. The
relevant part of my BlueBuild `recipe.yml`:

```yaml
modules:
  - type: rpm-ostree
    install:
      - fontawesome-fonts-all
      - https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
      # ... other packages
```

## Debugging

### Step 1: Verify fontconfig sees the emoji fonts

```console
$ fc-list | grep -i emoji
/usr/share/fonts/google-noto-color-emoji-fonts/Noto-COLRv1.ttf: Noto Color Emoji:style=Regular
/usr/share/fonts/google-noto-emoji-fonts/NotoEmoji-Regular.ttf: Noto Emoji:style=Regular
```

The fonts are installed and fontconfig can see them. Good.

### Step 2: Check the emoji generic family

```console
$ fc-match emoji
Noto-COLRv1.ttf: "Noto Color Emoji" "Regular"
```

This looks correct -- the `emoji` generic family maps to Noto Color Emoji.

### Step 3: Check charset-based matching

This is where things looked suspicious. When fontconfig resolves which font to
use for a specific emoji codepoint (U+1F600, the grinning face), it returns
fonts sorted by priority:

```console
$ fc-match -s ':charset=1F600' | head -5
Font Awesome 6 Free-Regular-400.otf: "Font Awesome 6 Free" "Regular"
Noto-COLRv1.ttf: "Noto Color Emoji" "Regular"
NotoEmoji-Regular.ttf: "Noto Emoji" "Regular"
Symbola.ttf: "Symbola" "Regular"
Font Awesome 6 Free-Solid-900.otf: "Font Awesome 6 Free" "Solid"
```

Font Awesome was winning the charset-based match. This looked like the smoking
gun -- Font Awesome covers some emoji-range codepoints and was ranked higher
than Noto Color Emoji.

### The fontconfig red herring

My first attempt was a fontconfig override that used `<alias>` rules with
`<prefer>` and `binding="strong"` to force Noto Color Emoji ahead of Font
Awesome in the sans-serif, serif, and emoji families. After applying the
override, `fc-match` confirmed the correct ordering:

```console
$ fc-match -s 'sans-serif:charset=1F600' | head -3
Noto-COLRv1.ttf: "Noto Color Emoji" "Regular"
NotoColorEmoji.ttf: "Noto Color Emoji" "Regular"
Font Awesome 6 Free-Regular-400.otf: "Font Awesome 6 Free" "Regular"
```

Fontconfig was now correctly prioritizing Noto Color Emoji. But **Chrome still
showed no emoji**. The fontconfig ordering was never the actual problem.

### Step 4: Test system rendering

I used Pango to render emoji directly and verify the system could handle the
COLRv1 font:

```python
from gi.repository import PangoCairo, Pango
import cairo

surface = cairo.ImageSurface(cairo.FORMAT_ARGB32, 400, 100)
ctx = cairo.Context(surface)
layout = PangoCairo.create_layout(ctx)
layout.set_text("\U0001F600\U0001F525\U0001F680", -1)
fd = Pango.FontDescription.from_string("Noto Color Emoji 48")
layout.set_font_description(fd)
PangoCairo.show_layout(ctx, layout)
surface.write_to_png("/tmp/emoji-pango.png")
```

This produced correct color emoji. The system CAN render COLRv1 -- Pango and
GTK applications work fine. The issue is Chrome-specific.

### Step 5: Test with CBDT format

I downloaded the older CBDT (bitmap) format of Noto Color Emoji from the
[noto-emoji repository](https://github.com/googlefonts/noto-emoji):

```bash
curl -sL -o ~/.local/share/fonts/NotoColorEmoji-CBDT.ttf \
  "https://github.com/googlefonts/noto-emoji/raw/main/fonts/NotoColorEmoji.ttf"
fc-cache -f
```

After restarting Chrome -- emoji worked.

## Root cause

The real issue has nothing to do with Font Awesome or fontconfig priority.

Fedora 43
[switched](https://fedoraproject.org/wiki/Changes/Use_COLR_for_Noto_Color_Emoji)
Noto Color Emoji from **CBDT** (bitmap) format to **COLRv1** (vector) format.
The font file is now `Noto-COLRv1.ttf`. While GTK/Pango applications render
COLRv1 correctly, **Chrome's Skia rendering engine cannot use this font on
Fedora**. When Chrome encounters an emoji codepoint, it finds the COLRv1 font
through fontconfig but fails to render it, resulting in blank output.

This also breaks terminal emulators that use the `fcft` library (like
[foot](https://codeberg.org/dnkl/foot)), which logs:

```text
err: fcft.c:695: /usr/share/fonts/google-noto-color-emoji-fonts/Noto-COLRv1.ttf: COLRv1 not supported
```

## The fix

Install the CBDT format of Noto Color Emoji alongside the COLRv1 version.
Chrome will use the CBDT format it can render, while GTK applications continue
using the COLRv1 version.

### Manual fix

```bash
sudo curl -sL -o /usr/share/fonts/google-noto-color-emoji-fonts/NotoColorEmoji-CBDT.ttf \
  "https://github.com/googlefonts/noto-emoji/raw/main/fonts/NotoColorEmoji.ttf"
fc-cache -f
# Restart Chrome completely
```

### BlueBuild integration

Add a script snippet to your `recipe.yml` to download the CBDT font during the
image build:

```yaml
- type: script@v1
  snippets:
    # Install CBDT format Noto Color Emoji — Chrome cannot render the COLRv1
    # format shipped by Fedora 43's google-noto-color-emoji-fonts package
    - curl -sL -o /usr/share/fonts/google-noto-color-emoji-fonts/NotoColorEmoji-CBDT.ttf
        "https://github.com/googlefonts/noto-emoji/raw/main/fonts/NotoColorEmoji.ttf"
```

No fontconfig overrides are needed. Both font formats have the same family name
(`Noto Color Emoji`), so applications pick whichever format they can render.

## Why not just remove Font Awesome?

On a Sway-based desktop, Font Awesome is typically used by
[Waybar](https://github.com/Alexays/Waybar) for status bar icons (battery,
network, volume, workspace indicators, etc.). Removing it would break the
status bar. But as it turns out, Font Awesome was never the problem -- it was a
red herring that led me down the fontconfig path.

## Key takeaway

When emoji rendering breaks after a distro upgrade, don't assume fontconfig
priority is the issue. The
[Fedora 43 COLRv1 migration](https://discussion.fedoraproject.org/t/f43-change-proposal-use-colr-for-noto-color-emoji-system-wide/148883)
introduced a font format that not all renderers support yet. The fix is to
provide the older CBDT format alongside COLRv1 so that applications like Chrome
have a format they can use.

The fix is available in my
[workstation](https://github.com/thrix/workstation) BlueBuild repository --
see the
[recipe.yml](https://github.com/thrix/workstation/blob/main/recipes/recipe.yml)
for reference.
