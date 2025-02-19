---
title: "Chromium Ozone/Wayland: The Last Mile Stretch"
author: 'Nick Yamane'
date: "2025-02-19T09:00:00-04:00"
images: ['chrome-wayland-flags-2025-02-18.png', 'chr-wl-per-surface-scale-fullhd-100_with-5627169.png']
toc: true
draft: false
comments:
  toot: 114031599876177866
tags: ['chromium', 'ozone', 'wayland', 'linux', 'mutter', 'gnome', 'igalia']
---

Hey there! I'm glad to finally start paying my blogging debt :) as this
is something I've been planning to do for quite some time now. To get the
ball rolling, I've shared some bits about me in my very first blog post
[Olá Mundo](/blog/ola-mundo/).

In this article, I'm going to walk through what we've been working on
since last year in the Chromium Ozone/Wayland project, on which I've
been involved (directly or indirectly) since I've joined Igalia back in
2018.

## Context

Lets start with some context, the project consists of implementing,
shipping and maintaining native [Wayland][wayland] support in the
Chromium project. Our team at [Igalia][igalia] has been leading the
effort since it was first merged upstream back in 2016. For more
historical context, there are a few [blog][msisov-post]
[posts][alex-post] and this [amazing talk][weh-talk], by my colleagues
Antonio Gomes and Max Ihlenfeldt, presented at last year's [Web Engines
Hackfest][weh].

Especially due to the [Lacros project][lacros], progresses on Linux
Desktop has been slower over the last few years. Fortunately, the
scenario changed since last year, when a new sponsor came up and made it
possible to address most of the outstanding missing features and issues
required to move Ozone Wayland to the finish line.

### Overall Status

It's been a few months since Chromium Wayland backend has started to be
tested as the main browser backend by Google employees, through a finch
trial experiment, as well as internally at Igalia. Feedback collected
since then is quite positive in general. The exception is Nvidia setups,
which, depending on the driver version, may face major regressions (see
[Explicit Sync](#explicit-sync) section below for more details).

While official roll-out has been under discussion, it's still disabled
by default on Linux Desktop. Early adopters willing to test it are
encouraged to explicitly opt-in by flipping the `ozone-platform-hint`
chrome flag to `auto` or `wayland`. Issue reports are welcome at
[crbug.com/new](https://crbug.com/new?component=1456988&template=0).

There are also a few other Wayland-specific flags which might be
selectively enabled, if you feel brave enough :) such as, ui scaling,
text input v3, etc; all described in more details below.

{{< video src="chrome-wayland-flags-2025-02-18_mod"
    autoplay="true" controls="true" loop="true"
    caption="Chrome Wayland flags available in M135." >}}

## New developments

### Fractional Scaling

Initial fractional scaling support for Linux Desktop was originally
implemented by an external contributor [back in
2023](https://chromium-review.googlesource.com/c/chromium/src/+/4370091).
After some months of stabilization, reports of [blurriness and some
other subtle issues](https://issues.chromium.org/336007385) started to
pop up, which were listed as top-priority when this new project phase
kicked off back in 2024 June.

After some analysis, we could confirm that there were some fundamental
missing bits in the process. Which was the actual usage of the fractional
scale values provided by the Wayland compositor, via
the [fractional-scale-v1](https://wayland.app/protocols/fractional-scale-v1)
protocol extension. Rather than using it, fractional scales were being
*inferred* using [xdg-output](https://wayland.app/protocols/xdg-output-unstable-v1)
protocol instead, which is unsupported (for such usage) and prone to
precision issues.

{{< figure src="chr-wl-per-surface-scale-fullhd-100_with-5627169.png" >}}

The screenshot above shows a sharp Chrome window scaled by a 1.25
factor, running on Gnome Shell 47.

The work involved a considerable architectural refactoring to make it
possible to support the per-window scaling design of the Wayland
protocol, without breaking the standard per-display scaling implemented
in Chromium. The feature started shipping experimentally in Milestone
128, behind the `wayland-per-window-scale` chrome flag and disabled by
default until M135. I plan to cover it in more details soon in separate
blog post.

{{< video src="chrome-wayland-fractional-scaling-2025-02-18"
    autoplay="true" controls="true" loop="true" >}}

### Input Methods

IME has been yet another major pain point for some Wayland users. Back
in last June, a careful study was conducted by my colleague [Orko
Garai](https://garai.ca) in order to understand and consolidate the
possible approaches to tackle it, as well as pros, cons and potential
road-blockers for each of them. Besides the detailed outcomes of that
study, publicly available in the form of a [design
document](https://docs.google.com/document/d/1GkOphcAQBMdW4iPiMOd9eKd70tlXWQaR7M3GJXGUDpQ),
Orko has recently started a blog post series about the topic
[here](https://garai.ca/what_is_ime).

{{< figure src="japanese.gif" >}}

Experimental support for
[text-input-v3](https://wayland.app/protocols/text-input-unstable-v3)
protocol was implemented on the Chromium side, with ongoing work on the
Wayland community side to fullfill browser use cases from a protocol
perspective, which is expected to come soon as part of [version
3.2](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/282)
of the text-input protocol.

In the meantime, we keep working with the Chromium community on
improvements to the client-side implementation of the protocol, although
progress has been slow as we are still looking for some key browser
requirements to be solved on the protocol side to have the confidence
and buy-in for productization.

### Tab Dragging

Supporting the full original user experience for Chrome's tab dragging
under Wayland has proved to be complex, especially because the core
Wayland protocol does not cover all of its requirements. Back in 2021,
we designed a brand new protocol and implemented it in ChromeOS' Exo
Wayland compositor, in the context of Lacros project, to fullfill those
gaps.

Years later,
[xdg-toplevel-drag](https://wayland.app/protocols/xdg-toplevel-drag-v1)
has emerged as a community effort to standardize it upstream, thanks
David Redondo and Robert Mader for working on it. It consists of a
trimmed down version of the original `zcr-extended-drag` protocol with
some tweaks to make it more aligned with Wayland design principles. A
few months ago, initial support for it has landed in Chromium and we've
been stabilizing it since then. 

#### Full UX support in Mutter

Back in last November, xdg-toplevel-drag was
[supported](https://wayland.app/protocols/xdg-toplevel-drag-v1#compositor-support)
only in KWin and Jay compositors, when a demand to implement support for
it in Mutter was raised by our customer, and the task was assigned to
me.

Long story short, it was a pretty interesting and challenging experience
which I'm glad to have had. As a curiosity, last time I had coded in C
and glib had been **long ago**, maybe >13 years? 😱 Also, it was my
first time hacking on Mutter and Gnome code base. After all, the
[MR](https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/4107) did
landed and will start shipping as part of Gnome 48, next month.

{{< video src="xdg-toplevel-drag-mutter-gnome-shell-demo-2024-10-25"
    autoplay="true" controls="true" loop="true"
    caption="xdg-toplevel-drag-v1 demo on Gnome Shell 48." >}}

I'm preparing a blog post to share more technical details about the
protocol implementation from the perspective of a browser developer and
Gnome/Mutter newcomer.

Let me take the opportunity to say thanks to the Gnome developers who
helped me a lot in the process: Jonas Ådahl, Carlos Garnacho, Georges
Stavracas and Sebastian Wick. Really appreciate your help and patience,
guys!

#### Fallback UX

In parallel to the work on the regular tab drag experience through
xdg-toplevel-drag, a fallback implementation relying solely on core
Wayland drag-and-drop protocol has been led by my colleague Max
Ihlenfeldt. A few days ago, Max has published an [awesome in-depth blog
post](https://blogs.igalia.com/max/fallback-tab-dragging/) about his
work on it. Enjoy the read!

{{< video src="fallback-tab-dragging-smaller"
    autoplay="true" controls="true" loop="true"
    caption="Fallback tab dragging UX demo" >}}

The main difference to the regular UX is that, rather than instantly
creating a browser window when it gets dragged out of its tab strip, a
drag icon containing the tab thumbnail is used instead. Browser window
creation is then deferred to when the drop happens, as can be seen in
the video above. The feature was recently enabled by default, and
started shipping in Chrome 133.

### Text Scaling

Linux desktop environments usually support system-wide "text scaling"
settings, which are supposed to be handled by applications. On
Gnome-based environments, it can be triggered via several ways,
such as the "Large Text" accessibility feature.

Historically, it has been supported in Chromium X11 by resizing the
whole browser UI elements, instead of just text items. After an in-depth
analysis, it was decided to follow the very same approach for the
Wayland initial implementation. The main motivation was that there seems
to be gaps in both Chromium's internal UI framework as well as in
Chrome's UI/layout code, which would need to be fixed before supporting
such text-only live resizing/relayout.

{{< video src="chrome-wayland-text-scaling-demo-2024-08-12"
    autoplay="true" controls="true" loop="true"
    caption="Quick demo of text scale in action on Chromium Wayland on Gnome 47." >}}

Technically, the solution involved implementing an additional scaling
layer in Ozone/Wayland, so called "ui scale" which make the browser UI
to get fully resized/re-laid out instantly in reaction to system's "text
scaling factor" updates.

The feature has started shipping in Milestone 131, disabled by default
as usual. Users can enable it by using the `chrome://flags/#wayland-ui-scaling`
flag.

### Explicit Sync

To address some [display tearing reports](https://issues.chromium.org/issues/377438303),
support for the [linux-drm-syncobj-v1](https://wayland.app/protocols/linux-drm-syncobj-v1)
protocol has been implemented. The patch series has landed and started
shipping in version 132.0.6834.83, and can be enabled using the
`wayland-linux-drm-syncobj` chrome flag.

Feedback has been positive so far. If you're willing to give it a try,
please bear in mind that Linux kernel version >= 6.11 is required, and
don't hesitate to get back to us with your remarks.

## Ongoing and Future Work

Besides overall stabilization and maintenance, there is a large ongoing
effort led by my colleague Orko to get Chromium's interactive UI tests
infrastructure and code working with major Wayland compositors,
primary focus is Mutter/Gnome, though wlroots is also considered
for the future.

Another area we are currently investigating is "session management",
which will make it possible to restore browser window attributes, such
as, position, display and workspace across restarts.

Aside from that, there are a bunch of technical debt and follow-up
issues which spun off from some of the features and fixes listed above,
such as:

- [Video Picture-in-Picture does not stay on top of other windows](https://issues.chromium.org/40285423)
- [Support text-only UI resizing in response to system's font scale changes](https://issues.chromium.org/issues/365076678)
- [Best-effort to support Window Management web API](https://issues.chromium.org/356665088)
- [Per-window scale renderer-side refactors](https://issues.chromium.org/348590032)
- [No native theme support in non-browser chrome widgets](https://issues.chromium.org/396190940)
- [Wacom Stylus does not register when chromium is on wayland natively w/ ozone](https://issues.chromium.org/40282832)

New sponsors and partners are always welcome, so please don't hesitate
to mail us to discuss how we coud be of help.

## That's it!

Yay! Quite busy and exciting times!! I'd like to thank once more all of
our supporters, sponsors and, of course, [Igalia][igalia] for making all
this possible ❤️ Looking forward to the challenges ahead! Stay tuned for
more updates and don't hesitate to reach out if you have questions or
other remarks. 👋👋

[igalia]: <https://igalia.com>
[wayland]: <https://wayland.freedesktop.org>
[msisov-post]: <https://blogs.igalia.com/msisov/chrome-on-wayland-waylandification-project/>
[alex-post]: <https://blogs.igalia.com/adunaev/2021/12/17/ozone-our-way-to-the-big-change/>
[weh]: <https://webengineshackfest.org/>
[weh-talk]: <https://www.youtube.com/watch?v=KSxaoXOSxhs>
[lacros]: <https://chromium.googlesource.com/chromium/src/+/refs/tags/88.0.4324.84/docs/lacros.md>

