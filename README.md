Simplified [`winetricks`](https://github.com/Winetricks/winetricks) that installs fonts, fast ~~and dangerous~~.

# Why
Got annoyed with `winetricks corefonts` taking nearly 2 minutes to install.
```
winetricks corefonts  13.13s user 8.16s system 18% cpu 1:54.12 total
```

# How
`winetricks` roughly does the following when installing the fonts:
- copy the font files into the prefix
- wait for `wineserver` to shutdown
- register fonts, by calling `regedit` twice for each font face
- if needed, register font replacements, by calling `regedit` for each font face

※ each `regedit` invocation is called twice, with `wine` and `wine64`

`corefonts` does 120 `regedit` calls and 11 `wineserver` waits.

Instead, this script is staging registry changes into temporary files, then merges them to import all changes in bulk.
This allows to cut down amount of `regedit` calls down to 2.

Waiting for `wineserver` to shutdown after copying font files was also removed.
Digging into history leads to [winetricks#896](https://github.com/Winetricks/winetricks/issues/896).

Comparing registries after installing `corefonts` with this and `winetricks` look seemingly the same.
(Apart from some fonts randomly disappearing and appearing in `HKCU\Software\Wine\Fonts\External Fonts`, but that doesn't seem to be `winetricks` related)

Not sure how to test `HKCU\Software\Wine\Fonts\Cache` population, but you should `wineserver -w` after.

This may or may not to come back to bite me later.

The end result can be seen in the following unscientific benchmarks done in empty tmpfs wine prefix initialized with `wineboot && wineserver -w`:
```shell
% time winetricks corefonts
winetricks corefonts  13.13s user 8.16s system 18% cpu 1:54.12 total
% time winetricksfast corefonts && time wineserver -w
winetricksfast corefonts  0.17s user 0.05s system 7% cpu 2.719 total
wineserver -w  0.00s user 0.00s system 0% cpu 4.053 total
```
And the worst case scenario with `allfonts`:
```shell
% time winetricks allfonts
winetricks allfonts  61.96s user 39.67s system 22% cpu 7:38.00 total
% time winetricksfast allfonts && time wineserver -w
winetricksfast allfonts  2.48s user 0.39s system 54% cpu 5.219 total
wineserver -w  0.00s user 0.00s system 0% cpu 4.057 total
```
