---
{"dg-publish":true,"permalink":"/resources/how-to-make-the-mal-minimal-dashboard-theme-panel-height-dynamic/","created":"2026-06-25","updated":"2026-06-25","dg-note-properties":{"type":["Article"],"topics":["[[Web Development]]","[[CSS]]","[[MyAnimeList]]"],"tags":null,"created":"2026-06-25","modified":"2026-06-25","author":["Domagoj Zubac"]}}
---


**Topics:** ==[[Web Development\|Web Development]],[[CSS\|CSS]],[[MyAnimeList\|MyAnimeList]]==

The [Minimal Dashboard](https://valeriolyndon.github.io/Theme-Customiser/theme?t=https://valeriolyndon.github.io/Theme-Customiser-Json/5cm/Dashboard.json) theme for MyAnimeList by 5cm is a clean, fixed-panel list design. By default the panel height is hardcoded to `541px`, which means it looks different depending on screen size. This guide shows how to replace all hardcoded values with a single CSS variable so the panel adapts to any viewport.

## Prerequisites

- A MyAnimeList account with a list set up
- CSS for the Minimal Dashboard theme that you can find via the [Theme Customiser](https://valeriolyndon.github.io/Theme-Customiser/)

## How the layout math works

The theme uses three related values that all need to stay in sync:

| Variable | Default value | Role |
|---|---|---|
| Header height | `68px` | Fixed top bar |
| Panel height | `541px` | The visible dashboard area |
| Panel bottom edge | `609px` | Where the footer starts (`68 + 541`) |

Changing one without the others causes the footer to overlap the list or leave a visible gap.

## Applying the fix

1. Go to your MyAnimeList profile.
2. Click **Profile Settings -> List Style Design**.
3. Choose **Modern Template** and click on **Default Theme** to edit it.
4. In the **Add Custom CSS** editor, locate or add a `:root` block at the top of your custom CSS and add the two new variables:

```css
:root {
  --panel-bottom: 60vh; /* ← this is the only value you need to change */
  --header-h: 68px;
}
```

4. Find and replace these three existing rules in the CSS with the versions below:

```css
body::before,
body::after {
  height: calc(var(--panel-bottom) - var(--header-h));
}

footer {
  top: var(--panel-bottom);
  height: calc(100% - var(--panel-bottom));
}

.list-unit {
  padding: 10px 30px calc(100vh - var(--panel-bottom));
}
```

5. Save and refresh your list page to see the result.

## Choosing a value for `--panel-bottom`

| Value | Result |
|---|---|
| `50vh` | Panel ends at half the screen height |
| `60vh` | Panel takes up 60% — good default for most screens |
| `70vh` | More room for longer lists on tall monitors |
| `calc(100vh - 100px)` | Near-fullscreen with a small gap at the bottom |

`60vh` is a good starting point. Going below `50vh` makes the list area cramped on 1080p displays since the `68px` header already eats into the available space.

## Why this works

CSS viewport units (`vh`) are relative to the browser window height and recalculate automatically on resize. By assigning `--panel-bottom` once in `:root`, all three layout rules that previously held separate hardcoded copies of `609px` now read from the same source, so they always stay in sync.