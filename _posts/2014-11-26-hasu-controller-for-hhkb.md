---
layout: post
title: The hasu controller for the Happy Hacking Keyboard
---

So there's nerd status, and then there's _nerd_ status. Today, I can proudly
say that I've finally hit _nerd_ status. About six months ago, I purchased a
[Happy Hacking Keyboard](http://imgur.com/a/TnlGf) (an expense your significant
other will never forgive you for), and I've never looked back.

Today, I received
[Hasu's alternative controller](https://geekhack.org/index.php?topic=12047.0)
for the HHKB. It gives you some pretty impressive functionality, like vi mode
in any application, as well as being able to control the mouse with hjkl.

The installation is a breeze. There is one ribbon cable to remove, and one
screw holding the controller in place. Once you have replaced the controller,
everything should work great, but in order to customize the key mappings to
be what you would like, you need to follow a few steps.

1. Install [Crosspack](http://www.obdev.at/products/crosspack/index.html) -
this is a tool that allows you to compile the code that will live on the new
microcontroller.

2. Install [dfu-programmer](http://dfu-programmer.github.io/) - similarly, this
is a tool that will allow you to actually communicate with the microcontroller
and flash the firmware.

3. Clone the tmk\_keyboard repository and `cd` into the hhkb directory:

    ```
    git clone git@github.com:tmk/tmk_keyboard
    cd tmk_keyboard/keyboard/hhkb/
    ```

4. This is where Crosspack comes in. You may need to restart your terminal in
order for the `avr-gcc` binary to be available to you. Compile the
tmk\_keyboard application:

    ```
    make -f Makefile
    ```

5. Hit the reset button-looking switch on the back of your new controller.

6. Compile the dfu code:

    ```
    make -f Makefile dfu
    ```

7. Compile the keymap you would like to use (denoted by the text after
"keymap\_" in the filename):

    ```
    make KEYMAP=hhkb
    ```

8. Flash the device with the new keymap:

    ```
    make dfu
    ```

Voila! You should have the new keymap enabled, and be able to switch to it by
holding both shifts and then "1" (or "0" to go back to normal). I should note,
it took me quite a while to realize that when I was switching layers
(essentially different keymaps), that I couldn't get back to the original.
That's because if you modify the key sequence to change layers, you won't be
able to get back, without unplugging and plugging your keyboard back in. So be
sure that you don't move your shift keys and number keys (or if you do, just
make note of them).

Here are my keymaps:

```c
/* Layer 0: Default Layer
 * ,-----------------------------------------------------------.
 * |Esc|  1|  2|  3|  4|  5|  6|  7|  8|  9|  0|  -|  =|  \|  `|
 * |-----------------------------------------------------------|
 * |Tab  |  Q|  W|  E|  R|  T|  Y|  U|  I|  O|  P|  [|  ]|Backs|
 * |-----------------------------------------------------------|
 * |Contro|  A|  S|  D|  F|  G|  H|  J|  K|  L|  ;|  '|Enter   |
 * |-----------------------------------------------------------|
 * |Shift   |  Z|  X|  C|  V|  B|  N|  M|  ,|  .|  /|Shift |Fn0|
 * `-----------------------------------------------------------'
 *       |Alt|Gui  |         Space         |Gui  |Alt|
 *       `-------------------------------------------'
 */
KEYMAP(ESC, 1,   2,   3,   4,   5,   6,   7,   8,   9,   0,   MINS,EQL, BSLS,GRV,   \
       TAB, Q,   W,   E,   R,   T,   Y,   U,   I,   O,   P,   LBRC,RBRC,BSPC,       \
       LCTL,A,   S,   D,   F,   G,   H,   J,   K,   L,   SCLN,QUOT,ENT,             \
       LSFT,Z,   X,   C,   V,   B,   N,   M,   COMM,DOT, SLSH,RSFT,FN0,             \
            LALT,LGUI,          SPC,                RGUI,RALT),

/* Layer 1: Cursor(HHKB mode)
 * ,-----------------------------------------------------------.
 * |Pwr| F1| F2| F3| F4| F5| F6| F7| F8| F9|F10|F11|F12|Ins|Del|
 * |-----------------------------------------------------------|
 * |Caps |   |   |   |   |   |   |   |Psc|Slk|Pus|Up |   |Backs|
 * |-----------------------------------------------------------|
 * |Contro|VoD|VoU|Mut|   |   |  *|  /|Hom|PgU|Lef|Rig|Enter   |
 * |-----------------------------------------------------------|
 * |Shift   |   |   |   |   |   |  +|  -|End|PgD|Dow|Shift |Fn0|
 * `-----------------------------------------------------------'
 *      |Alt |Gui  |Space                  |Alt  |Gui|
 *      `--------------------------------------------'
 */
KEYMAP(PWR, F1,  F2,  F3,  F4,  F5,  F6,  F7,  F8,  F9,  F10, F11, F12, INS, DEL, \
       CAPS,NO,  NO,  NO,  NO,  NO,  NO,  NO,  PSCR,SLCK,PAUS,UP,  NO,  BSPC,     \
       LCTL,VOLD,VOLU,MUTE,NO,  NO,  PAST,PSLS,HOME,PGUP,LEFT,RGHT,ENT,           \
       LSFT,NO,  NO,  NO,  NO,  NO,  PPLS,PMNS,END, PGDN,DOWN,RSFT,FN0,           \
            LALT,LGUI,          SPC,                RGUI,RALT),

/* Layer 2: Mouse Mode for HJKL
 * ,-----------------------------------------------------------.
 * |Esc|  1|  2|  3|  4|  5|  6|  7|  8|  9|  0|  -|  =|  \|  `|
 * |-----------------------------------------------------------|
 * |Tab  |  Q|  W|  E|  R|  T|  Y|  U|  I|  O|  P|  [|  ]|Backs|
 * |-----------------------------------------------------------|
 * |Contro|  A|  S|  D|  F|  G|McL|McD|McU|McR|  ;|  '|Enter   |
 * |-----------------------------------------------------------|
 * |Shift   |  Z|  X|  C|  V|  B|  N|  M|  ,|  .|  /|Shift |Fn0|
 * `-----------------------------------------------------------'
 *       |Alt|Gui  |         Space         |Gui  |Alt|
 *       `-------------------------------------------'
 */
KEYMAP(ESC, 1,   2,   3,   4,   5,   6,   7,   8,   9,   0,   MINS,EQL, BSLS,GRV,   \
       TAB, Q,   W,   E,   R,   T,   Y,   U,   I,   O,   P,   LBRC,RBRC,BSPC,       \
       LCTL,A,   S,   D,   F,   G,   MS_L,MS_D,MS_U,MS_R,SCLN,QUOT,ENT,             \
       LSFT,Z,   X,   C,   V,   B,   N,   M,   COMM,DOT, SLSH,RSFT,FN0,             \
            LALT,LGUI,          BTN1,                RGUI,RALT),
```
