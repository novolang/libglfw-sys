# Changelog

All notable changes to libglfw-sys are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

### Corrected against the GLFW 3.3.10 header and sources

- `GLFW_CLIENT_API` is 0x00022001. 0x00021001 is `GLFW_RED_BITS`.
- `GLFW_FORMAT_UNAVAILABLE` is 0x00010009. 0x00010008 is
  `GLFW_PLATFORM_ERROR`.
- `GLFW_X11_CLASS_NAME` and `GLFW_X11_INSTANCE_NAME` are 0x00024001
  and 0x00024002. 0x00023001 and 0x00023002 are two of the Cocoa
  hints. GLFW 3.3 has no `GLFW_WAYLAND_APP_ID` at all.
- 0x00051001 is `GLFW_COCOA_CHDIR_RESOURCES` and 0x00051002 is
  `GLFW_COCOA_MENUBAR`.
- `glfwGetKey` refuses a key code outside `GLFW_KEY_SPACE` to
  `GLFW_KEY_LAST` with `GLFW_INVALID_ENUM` and answers `GLFW_RELEASE`.
  It never answers -1.
- `glfwGetKeyName` uses the scancode only when the key is
  `GLFW_KEY_UNKNOWN`, which is -1. A key code is otherwise enough on
  its own.
- Sixteen calls may be made from a thread other than the main one, not
  `glfwPostEmptyEvent` alone.

Two of those reach the test suite. The constant named
`GLFW_X11_CLASS_NAME` now holds 147457, the value the name stands for.
The assertion that an unknown key code answers -1 now asserts
`GLFW_RELEASE` and the `GLFW_INVALID_ENUM` that comes with it.

## 0.1.0 — 2026-09-16

The first release: sixty-nine entry points of the GLFW C API, one
`@ffi` declaration each, and no logic.

### Added

- `libglfw` — the whole surface, in eight groups.
  - The library: `glfwGetVersion`, `glfwGetVersionString`,
    `glfwInitHint`, `glfwInit`, `glfwTerminate` and `glfwGetError`.
  - The monitors: `glfwGetMonitors`, `glfwGetPrimaryMonitor`, the four
    geometry queries, `glfwGetMonitorName`, `glfwGetVideoMode` and
    `glfwGetVideoModes`.
  - The window's creation: `glfwDefaultWindowHints`, `glfwWindowHint`,
    `glfwWindowHintString`, `glfwCreateWindow`, `glfwDestroyWindow`
    and the two close-flag calls.
  - The window's state: the title, the position, the size, the
    framebuffer size, the content scale, the six visibility calls, the
    two attribute calls and the two monitor calls.
  - Events: `glfwPollEvents`, `glfwWaitEvents`,
    `glfwWaitEventsTimeout` and `glfwPostEmptyEvent`.
  - Input by polling: the two input-mode calls,
    `glfwRawMouseMotionSupported`, `glfwGetKey`, `glfwGetKeyName`,
    `glfwGetMouseButton`, the two cursor-position calls, the three
    cursor-shape calls and the two clipboard calls.
  - The clock: `glfwGetTime`, `glfwSetTime`, `glfwGetTimerValue` and
    `glfwGetTimerFrequency`.
  - The context and Vulkan: `glfwMakeContextCurrent`,
    `glfwGetCurrentContext`, `glfwSwapBuffers`, `glfwSwapInterval`,
    `glfwGetProcAddress`, `glfwExtensionSupported`,
    `glfwVulkanSupported`, `glfwGetRequiredInstanceExtensions` and
    `glfwCreateWindowSurface`.
- `tests/libglfw_tests.nv` — twelve tests over the entry points. Each
  asks `glfwInit` first and stops when it answers 0, so the suite is
  written to pass on a machine with no display as well as on one with
  a display. The windows it creates are created hidden.

### Unverified

GLFW is not installed on the machine this package was written on, so
the suite has never been linked and no declaration has ever been called.
The declarations were checked against the GLFW 3.3 reference manual.
Treat the whole package as unmeasured until someone runs it on a
machine with GLFW and a display.

### The callbacks, and the polling half that is left

GLFW reports every event through a C function pointer. The novo-lang
foreign function interface passes integers, floats and strings, so not
one of the twenty `glfwSet*Callback` entry points is here.

What is left is the polling half of the same C API, and it is a working
half. `glfwPollEvents` drains the queue, and `glfwWindowShouldClose`,
`glfwGetKey`, `glfwGetMouseButton`, `glfwGetCursorPos`,
`glfwGetWindowSize` and `glfwGetFramebufferSize` are asked afterwards
what it did. A game loop written that way — poll, read the state, draw,
repeat — needs nothing this package does not have.

Four things have no polled form and are therefore out of reach rather
than merely out of this release:

| What | The only way to it | What a program loses |
| --- | --- | --- |
| typed text | `glfwSetCharCallback` | text entry of any kind |
| the scroll wheel | `glfwSetScrollCallback` | scrolling and pinch zoom |
| dropped files | `glfwSetDropCallback` | drag and drop |
| the moment of a resize | `glfwSetFramebufferSizeCallback` | redrawing during a drag rather than after it |

A program that needs typed text needs a C shim of its own, and that
shim is the same shape as the one `libtree-sitter-sys` needs for the
by-value half of its own C API.

### The C `float` types that cannot be declared

An `@ffi` declaration has one floating point type, `Float`, and it is a
C `double`. `glfwSetWindowOpacity` and `glfwSetGamma` take a C `float`
and `glfwGetWindowOpacity` returns one, so none of the three can be
declared. A `double` is fine, which is why `glfwGetTime`,
`glfwSetTime`, `glfwWaitEventsTimeout`, `glfwGetCursorPos` and
`glfwSetCursorPos` are here.

A `float` behind a pointer is half reachable.
`glfwGetMonitorContentScale` and `glfwGetWindowContentScale` write two
32-bit floats into slots the caller owns, and they are declared,
because the standard library's `ptr.write_f32` can put one there. There
is no `ptr.read_f32`, so a caller gets the scale only as its raw bits.
The joystick and gamepad calls answer arrays of floats for the same
reason and are left out entirely.

### Named as missing

**Every `glfwSet*Callback`.** Twenty entry points: the window position,
size, close, refresh, focus, iconify, maximise, framebuffer size and
content scale callbacks; the key, character, character-with-modifiers,
mouse button, cursor position, cursor enter, scroll and drop callbacks;
and the joystick, monitor and error callbacks. Each takes a C function
pointer.

**`glfwSetWindowOpacity`, `glfwGetWindowOpacity` and `glfwSetGamma`.**
Their argument or return type is a C `float`.

**The joystick and gamepad calls.** `glfwGetJoystickAxes` and
`glfwGetGamepadState` answer arrays of 32-bit floats, and the standard
library has no call that reads one back out of memory.

**`glfwSetWindowIcon` and `glfwCreateCursor`.** Both take an array of
`GLFWimage` records, each a width, a height and the address of pixel
data the caller lays out by hand.

**`glfwGetGammaRamp` and `glfwSetGammaRamp`.** A `GLFWgammaramp` is
three arrays of 16-bit values behind pointers, which a caller lays out
by hand in the same way.

**`glfwSetWindowUserPointer` and `glfwGetWindowUserPointer`.** They
exist to carry a program's own pointer into a callback, and there are
no callbacks.

**`glfwGetPlatform` and `glfwPlatformSupported`.** They are GLFW 3.4,
and this package's declarations are the ones a 3.3 library also
exports, so that a 3.3 installation links.

**The native access header `glfw3native.h`.** `glfwGetX11Display`,
`glfwGetWaylandWindow` and their neighbours are guarded by platform
macros, and each one exists only on its own platform. A binding that
declared them would fail to link everywhere else.
