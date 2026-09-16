# libglfw-sys

GLFW is a library for creating a window with an OpenGL or Vulkan
drawing surface, and for reading the keyboard, the mouse and the
monitors attached to the machine. It is documented in the
[GLFW documentation](https://www.glfw.org/docs/latest/), and it is what
most small graphics programs use instead of talking to X11, Wayland,
Win32 or Cocoa themselves. This package declares sixty-nine of that
library's entry points to novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in GLFW. The package contains no logic of its
own, and it does nothing without the C library installed.

**Unverified.** GLFW was not installed on the machine this package was
written on, so the test suite has never been linked and no declaration
has ever been called. The declarations were checked against the GLFW
3.3 reference manual. Treat the whole package as unmeasured until
someone runs it on a machine with GLFW and a display.

The sixty-nine entry points are the polling half of GLFW. The section
"What is not included" says what the other half was, and what a program
loses with it.

## What it is

A **window** is a region the window manager gives the program, together
with a drawing surface. GLFW creates it, and everything else in the
library is about that window or about the machine it is on.

A **monitor** is a display attached to the machine. A **video mode** is
a resolution, a colour depth and a refresh rate that monitor supports.

**Screen coordinates** are what the window manager measures in.
**Pixels** are what the drawing surface is made of. On a display whose
**content scale** is not one — a high-resolution laptop screen, say —
a window 400 screen coordinates wide has a framebuffer 800 pixels wide,
and a program that confuses the two draws at a quarter of the screen or
at four times it.

An **event** is something the user or the window manager did. GLFW
collects events as they arrive and does nothing with them until the
program asks. `glfwPollEvents` is that ask.

A **hint** is a setting that applies to the next window created rather
than to a window that exists. Hints are library state: they survive
window creation, and a program that sets one sets it for every later
window until it resets them.

**Polling** is asking, after the events have been taken, what the state
now is: whether the window has been asked to close, whether a key is
down, where the cursor is. It is the alternative to being told, and
this package has only the first.

## Install

```
novo pkg add libglfw-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package `libglfw3-dev`:

```
sudo apt install libglfw3-dev
```

On macOS the Homebrew formula is `glfw`. On other systems the library
builds from the GLFW source with CMake.

## Example

A window that stays open until it is asked to close or Escape is
pressed:

```novo ignore
use libglfw

fn main() [io, ffi]
    if libglfw.glfw_init() == 0
        println("GLFW did not start")
        return

    // No OpenGL context: this window is for Vulkan, or for nothing.
    libglfw.glfw_window_hint(139265, 0)
    let window = libglfw.glfw_create_window(640, 480, "novo", 0, 0)
    if window == 0
        libglfw.glfw_terminate()
        return

    // The loop: take the events, then ask what they did.
    while libglfw.glfw_window_should_close(window) == 0
        libglfw.glfw_poll_events()
        // 256 is Escape, and 1 is GLFW_PRESS.
        if libglfw.glfw_get_key(window, 256) as i32 == 1
            libglfw.glfw_set_window_should_close(window, 1)

    libglfw.glfw_destroy_window(window)
    libglfw.glfw_terminate()
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in `tests/libglfw_tests.nv`,
where the windows are created hidden.

## What the package contains

| Module | Contents |
| --- | --- |
| `libglfw` | Every entry point, in eight groups: the library, the monitors, the window's creation, the window's state, the events, the polled input, the clock, and the OpenGL context with the Vulkan surface calls. |

The eight groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The library | 6 | Reports the version, starts and stops the library, and reads the last error. |
| The monitors | 9 | Lists the displays and reports each one's position, size, name and video modes. |
| The window's creation | 7 | Sets the hints, makes the window, and carries the close flag. |
| The window's state | 17 | The title, the geometry, the visibility and the attributes. |
| Events | 4 | Takes what has arrived, waits for something to arrive, or wakes the waiter. |
| Polled input | 13 | The keyboard, the mouse, the cursor shape and the clipboard. |
| The clock | 4 | Seconds as a floating point number, and the raw timer behind it. |
| Context and Vulkan | 9 | Makes a context current, swaps the buffers, and answers what a Vulkan surface needs. |

## How to choose an entry point

`glfwPollEvents` is for a program that draws every frame: it takes
whatever has arrived and returns at once. `glfwWaitEvents` is for a
program that draws only when something changed: it uses no processor
while it waits. `glfwWaitEventsTimeout` is the same wait with an upper
bound, and it is the one a test can call.

`glfwGetWindowSize` answers screen coordinates and
`glfwGetFramebufferSize` answers pixels. Give the framebuffer size to
`glViewport` or to a Vulkan swapchain, and the window size to anything
the user positions.

`glfwGetKey` answers whether a key is down now. It is not how a program
reads typed text, and there is no entry point in this package that is;
see "What is not included".

`glfwGetKeyName` answers what a key prints on the user's layout. Use it
to show a key binding, never to read input.

## The rules a user needs

1. **`glfwInit` comes first and `glfwTerminate` comes last.** Only
   `glfwGetVersion`, `glfwGetVersionString` and `glfwInitHint` work
   outside the pair. `glfwTerminate` destroys every window and cursor
   the program still holds.
2. **A pointer is an `Int`, and zero is null.** A window, a monitor and
   a cursor are addresses the library returned, and every creation call
   answers 0 on failure.
3. **An out-parameter is the address of a caller-owned slot.** GLFW
   answers a pair of numbers that way rather than by returning a
   structure. A slot holding a C `int` is written with
   `ptr.write_i32` and read with `ptr.read_word(a) as i32`.
4. **An answer can be negative, so read it with `as i32`.**
   `glfwGetKey` answers -1 for a key code it does not know, and
   `glfwCreateWindowSurface` answers a Vulkan result code, whose
   failures are negative.
5. **An error is taken off a queue, not returned.** `glfwGetError`
   answers the last error and clears it, and answers 0 when there was
   none. Pass 0 for the description, or the address of a slot that
   receives the address of a string valid only until the next error.

   | Code | Name | What it means |
   | --- | --- | --- |
   | 0x00000000 | `GLFW_NO_ERROR` | nothing went wrong |
   | 0x00010001 | `GLFW_NOT_INITIALIZED` | a call was made outside `glfwInit` and `glfwTerminate` |
   | 0x00010002 | `GLFW_NO_CURRENT_CONTEXT` | a context call was made with no context current |
   | 0x00010003 | `GLFW_INVALID_ENUM` | a hint, mode or attribute is not one GLFW has |
   | 0x00010004 | `GLFW_INVALID_VALUE` | the value is out of range |
   | 0x00010005 | `GLFW_OUT_OF_MEMORY` | an allocation failed |
   | 0x00010006 | `GLFW_API_UNAVAILABLE` | the machine has no driver for the client interface asked for |
   | 0x00010008 | `GLFW_FORMAT_UNAVAILABLE` | the clipboard holds something that is not text |

6. **Hints are library state.** They apply to the next window and
   survive its creation. `glfwDefaultWindowHints` puts them all back.

   | Hint | Number | What it sets |
   | --- | --- | --- |
   | `GLFW_RESIZABLE` | 0x00020003 | whether the user may resize the window |
   | `GLFW_VISIBLE` | 0x00020004 | whether the window appears when it is created |
   | `GLFW_DECORATED` | 0x00020005 | whether the window manager draws a frame |
   | `GLFW_CLIENT_API` | 0x00021001 | 0x00030001 for OpenGL, 0 for none |
   | `GLFW_CONTEXT_VERSION_MAJOR` | 0x00022002 | the OpenGL version asked for |
   | `GLFW_CONTEXT_VERSION_MINOR` | 0x00022003 | the same |

   A Vulkan program sets `GLFW_CLIENT_API` to 0, or GLFW creates an
   OpenGL context the program never uses.

7. **Screen coordinates are not pixels.** `glfwGetWindowSize` answers
   the first and `glfwGetFramebufferSize` the second, and they differ
   by the content scale.
8. **A video mode is a 24-byte record.** Six C `int`s in this order:
   the width, the height, the red bits, the green bits, the blue bits
   and the refresh rate. `glfwGetVideoModes` answers an array of them
   and `glfwGetVideoMode` answers one.
9. **A content scale is two 32-bit floats behind pointers, and the
   standard library cannot read one back.** `ptr.write_f32` writes a
   32-bit float and there is no call that reads one, so
   `glfwGetMonitorContentScale` and `glfwGetWindowContentScale` give a
   caller the raw bits and nothing more. A program that needs the
   number can divide the framebuffer size by the window size instead.
10. **A time is a C `double` and is expressible.** `glfwGetTime`,
    `glfwSetTime`, `glfwWaitEventsTimeout`, `glfwGetCursorPos` and
    `glfwSetCursorPos` all carry doubles, which the foreign function
    interface passes. A C `float` is a different type and is not; see
    "What is not included".
11. **The key codes are the printable character where there is one.**
    32 is space, 65 to 90 are A to Z and 48 to 57 are 0 to 9; 256 is
    Escape, 257 Enter, 258 Tab, 259 Backspace, and 262 to 265 are
    right, left, down and up. A key answers 1 while it is down and 0
    while it is not.
12. **The mouse buttons are 0, 1 and 2**, for left, right and middle.
13. **Sticky input keeps a press until it is read.** Input mode
    0x00033002 for keys and 0x00033003 for buttons makes a press that
    happened and ended between two polls still answer 1 once. It is how
    a program that polls slowly stops missing a tap.
14. **A grabbed cursor is how a camera is driven.** Input mode
    0x00033001 set to 0x00034003 hides the cursor and stops it leaving
    the window, and `glfwGetCursorPos` then answers unbounded values.
    0x00034001 restores it.
15. **A string the library answers belongs to the library.** The
    version string, a monitor's name, a key's name and the clipboard's
    text are all addresses GLFW owns. Copy each with `ptr.read_str`
    before the next call.
16. **`glfwGetProcAddress` answers an address this package cannot
    call.** Use it to ask whether the current context has an OpenGL
    function, and for nothing else.
17. **Every window call must be made on the thread that called
    `glfwInit`.** GLFW's event handling is single-threaded, and
    `glfwPostEmptyEvent` is the only call another thread may make.

## What is not included

- **Every `glfwSet*Callback`.** Twenty entry points, each taking a C
  function pointer, and the novo-lang foreign function interface passes
  integers, floats and strings. The polled calls in this package cover
  most of what they reported. These four have no polled form:

  | What | Its only callback | What a program loses |
  | --- | --- | --- |
  | typed text | `glfwSetCharCallback` | text entry of any kind |
  | the scroll wheel | `glfwSetScrollCallback` | scrolling and pinch zoom |
  | dropped files | `glfwSetDropCallback` | drag and drop |
  | the moment of a resize | `glfwSetFramebufferSizeCallback` | redrawing during a drag rather than after it |

  A program that needs any of the four needs a C shim of its own that
  holds the callback and records what it was told, and a polled call of
  its own to read that record.

- **`glfwSetWindowOpacity`, `glfwGetWindowOpacity` and
  `glfwSetGamma`.** Their argument or return type is a C `float`, and
  an `@ffi` declaration's one floating point type is a C `double`.
- **The joystick and gamepad interface.** `glfwGetJoystickAxes` and
  `glfwGetGamepadState` answer arrays of 32-bit floats, and the
  standard library has no call that reads one out of memory.
- **`glfwSetWindowIcon` and `glfwCreateCursor`.** Both take an array of
  `GLFWimage` records. They are left out of the first release.
- **`glfwGetGammaRamp` and `glfwSetGammaRamp`.** A gamma ramp is three
  arrays of 16-bit values behind pointers. Left out of the first
  release.
- **`glfwSetWindowUserPointer` and `glfwGetWindowUserPointer`.** They
  exist to carry a program's own pointer into a callback.
- **`glfwGetPlatform` and `glfwPlatformSupported`.** They are GLFW 3.4.
  Every declaration in this package is one a GLFW 3.3 library exports
  as well, so a 3.3 installation links.
- **The native access interface.** `glfwGetX11Display`,
  `glfwGetWaylandWindow` and their neighbours exist only on their own
  platform, and a binding that declared them would fail to link on
  every other one.

## Related packages

There is no GLFW port in novo-lang and there will not be one. A window
is not a format or an algorithm; it is what the operating system gives
a program, and the only implementation is the platform's own.

`libvulkan-sys` binds the Vulkan loader. This package's
`glfwGetRequiredInstanceExtensions` answers the extension names that
package's `vkCreateInstance` must enable, and
`glfwCreateWindowSurface` turns a window into the `VkSurfaceKHR` a
swapchain presents to.

## Tests

`tests/libglfw_tests.nv` holds twelve tests written against the
signatures. They call the C library, so `novo test` needs GLFW
installed and linkable:

```
novo test tests/libglfw_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

GLFW was not installed on the machine this package was written on, so
**the suite has never been linked and has never been run**. Without the
library the link fails, naming `-lglfw`. Every assertion below is
therefore a claim about what GLFW's documentation says, and not an
observation.

The suite needs a display as well as a library. Every test asks
`glfwInit` first and stops when it answers 0, so it is written to pass
on a machine with no display too. The windows it creates are created
hidden, so nothing appears on screen, and it reads and writes no file
and needs no privileges.

`glfw_wait_events` is the one entry point no test calls. It blocks
until an event arrives, and on an idle machine with no user that is
forever; `glfw_wait_events_timeout` is the form a test can call, and it
is called with a tenth of a second.

## Implementation status

| Group | State |
| --- | --- |
| The library | Complete. Unverified. |
| The monitors | Complete except for the gamma ramp. Unverified. |
| The window's creation | Complete except for the window icon. Unverified. |
| The window's state | Complete except for the opacity, which is a C `float`. Unverified. |
| Events | Complete for the polling model. Unverified. |
| Polled input | Complete except for the custom cursor image. Unverified. |
| The clock | Complete. Unverified. |
| Context and Vulkan | Complete. Unverified. |
| The callbacks | Absent. Every one is a C function pointer. |
| Typed text, scrolling and dropped files | Absent. They have no polled form. |
| Joysticks and gamepads | Absent. They answer arrays of 32-bit floats. |
| The native access interface | Absent. Each call exists on one platform only. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

GLFW itself is distributed under the zlib/libpng licence, and
installing it is the reader's own step.
