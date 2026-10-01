# wine-xshape-alpha-fix

Experimental Wine X11 patch for shaped ARGB windows when running through
xwayland-satellite.

## Problem

Some Wine applications create 32-bit ARGB X11 windows whose pixels outside
the visible window shape are still fully opaque.

For example, FL Studio's animated fruit splash/about window contains opaque
black pixels outside the fruit and relies on the X11 Shape extension
(`ShapeBounding`) to clip them.

A normal X11 compositor honors the shape and displays the window correctly.

With xwayland-satellite, the rectangular ARGB buffer is forwarded to Wayland,
but the X11 visual bounding shape is not reproduced. The result is an opaque
black rectangle around the shaped window.

## Fix

This patch changes Wine's X11 window-surface path when the environment
variable

    WINE_X11_BAKE_SHAPE_ALPHA=1

is set.

Instead of relying on X11 Shape for visual clipping, Wine:

1. keeps the X11 window's visual bounding shape rectangular;
2. uses Wine's existing 1-bit shape bitmap;
3. sets pixels outside that shape to transparent ARGB;
4. uploads that ARGB image to the X window.

This makes the visual shape survive xwayland-satellite because the shape is
encoded directly in the alpha channel.

The normal Wine behavior is unchanged unless
`WINE_X11_BAKE_SHAPE_ALPHA=1` is set.

## Tested

Tested with:

- Wine Staging 11.18
- Gentoo Linux
- traditional multilib Wine (`-wow64`, `ABI_X86="32 64"`)
- xwayland-satellite
- niri
- FL Studio 20

Before the patch:
- black rectangle around the FL Studio fruit window

After the patch:
- correct transparency
- no flashing/flickering
- normal FL Studio startup speed
- no UI lag while the fruit animation is visible

## Usage

Apply:

    patches/wine-11.18-xshape-alpha.patch

to Wine 11.18, rebuild Wine, then launch the affected program with:

    WINE_X11_BAKE_SHAPE_ALPHA=1

Do not export this globally unless you specifically want the modified behavior
for all 32-bit-depth X11 Wine surfaces.

## Gentoo

For Wine Staging 11.18:

    sudo mkdir -p /etc/portage/patches/app-emulation/wine-staging-11.18

    sudo cp patches/wine-11.18-xshape-alpha.patch \
        /etc/portage/patches/app-emulation/wine-staging-11.18/

    sudo emerge -1av =app-emulation/wine-staging-11.18

Then add:

    WINE_X11_BAKE_SHAPE_ALPHA=1

to the environment of the application that needs the workaround.

## Status

Experimental workaround / proof of concept.

The patch currently enables the behavior through an environment variable and
has only been tested on Wine Staging 11.18 with FL Studio.

It is not intended as an upstream-ready Wine patch in its current form.
