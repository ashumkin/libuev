libuEv | Simple event loop for Linux
====================================

> "Why an event loop library, why not use threads?"

With the advent of light-weight processes (threads) programmers these
days have a [golden hammer](http://c2.com/cgi/wiki?GoldenHammer) they
often swing without consideration.  Event loops and non-blocking I/O is
often a far easier approach, as well as less error prone.

The purpose of many applications is, with a little logic sprikled on
top, to act on: network packets entering an interface, timeouts
expiring, mouse clicks, or other types of events.  Such applications are
often very well suited to use an event loop.

Applications that need to churn massively parallel algorithms are more
suitable for running multiple (independent) threads on several CPU
cores.  However, threaded applications need to deal with all the issues
concurrency can cause, like: race conditions, deadlocks, live locks,
etc. Writing error free threaded applications is hard, debugging them
can be even harder.

Sometimes the combination of multiple threads *and* an event loop per
thread can be the best approach, but each application of course needs to
be broken down individually to find the most optimal approach.  Do keep
in mind, however, that not all systems your application will run on have
multiple CPU cores -- some small embedded systems still use a single CPU
core, even though they run Linux.

LibuEv is a simple event loop in the style of the more established
[libevent](http://libevent.org/),
[libev](http://software.schmorp.de/pkg/libev.html) and the venerable
[Xt(3)](http://unix.com/man-page/All/3x/XtDispatchEvent) event loop.
The *u* (micro) in the name refers to both the small feature set and the
small size overhead impact of the library.  The primary target of libuEv
is modern Linux systems.

Experienced developers may appreciate that libuEv is built on top of
modern Linux APIs: epoll, timerfd and signalfd.


API
---

Here is the interface to libuEv.  It handles three different types of
events: I/O (files, sockets, message queues, etc.), timers, and
signals.

    /* Event loop functions */
    int uev_init        (uev_ctx_t *ctx);
    int uev_exit        (uev_ctx_t *ctx);
    int uev_run         (uev_ctx_t *ctx, int flags);  /* UEV_ONCE, UEV_NONBLOCK, or zero(0) */

    /* Event callback definition, arg is the same arg as passed to below *_init() functions */
    typedef void (uev_cb_t)(uev_ctx_t *ctx, uev_t *w, void *arg, int events);
    
    /* I/O watcher */
    int uev_io_init     (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, int fd, int events);
    int uev_io_set      (uev_t *w, int fd, int events);
    int uev_io_stop     (uev_t *w);

    /* Timer watcher, timeout and period in milliseconds */
    int uev_timer_init  (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, int timeout, int period);
    int uev_timer_set   (uev_t *w, int timeout, int period);
    int uev_timer_stop  (uev_t *w);

    /* Signal watcher */
    int uev_signal_init (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, int signo);
    int uev_signal_set  (uev_t *w, int signo);
    int uev_signal_stop (uev_t *w);

To be able to setup callbacks to events the developer first need to
create an *event context*, achieved by calling `uev_init()` with a
pointer to a local `uev_ctx_t` variable.

Events are monitored by watchers and each watcher needs to be
registered, with a callback, with the event context.  This is done by
passing the `uev_ctx_t` variable, along with an `uev_t` variable to each
event's `_init()` function.

   1. Prepare an event context with `uev_init()`
   2. Register callback with `uev_io_init()`, `uev_signal_init()` or
      `uev_timer_init()`
   3. Enter the event loop with `uev_run()`
   4. Exit the event loop with `uev_exit()`, possibly from a callback

**Note:** Make sure to use non-blocking stream I/O!  Most hard to find
  bugs in event driven applications is due to file descriptors and
  sockets being opened in blocking mode.  Be careful out there!


Example
-------

Here follows a very brief example to illustrate how one can use libuEv
to act on joystick input.

```C
#include <err.h>
#include <errno.h>
#include <stdio.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>

#include "uev.h"

struct js_event {
	uint32_t time;		/* event timestamp in milliseconds */
	int16_t  value;		/* value */
	uint8_t  type;		/* event type */
	uint8_t  number;	/* axis/button number */
} e;

static void joystick_cb(uev_ctx_t *ctx, uev_t *w, void *arg, int events)
{
	read (w->fd, &e, sizeof(e));

	switch (e.type) {
	case 1:
		if (e.value) printf("Button %d pressed\n", e.number);
		else 	     printf("Button %d released\n", e.number);
		break;

	case 2:
		printf("Joystick axis %d moved, value %d!\n", e.number, e.value);
		break;
	}
}

int main(void)
{
	int       fd = open("/dev/input/js1", O_RDONLY, O_NONBLOCK);
	uev_t     js1_watcher;
	uev_ctx_t ctx;

	if (fd < 0)
		errx(errno, "Cannot find a joystick attached.");

	uev_init(&ctx);
	uev_io_init(&ctx, &js1_watcher, joystick_cb, NULL, fd, UEV_READ);

	puts("Starting, press Ctrl-C to exit.");

	return uev_run(&ctx, 0);
}
```

To compile the program, save the code as `joystick.c` and call GCC with
`gcc -o joystick joystick.c io.c timer.c signal.c main.c` from this
directory, skips using a Makefile altogether.

For a more complete, and perhaps more relevant example, see the code for
the TFTP/FTP server [uftpd](https://github.com/troglobit/uftpd).  It
uses libuEv as a GIT submodule.

Also, see the `bench.c` program (`make bench` from within the library)
for [reference benchmarks](http://libev.schmorp.de/bench.html) against
libevent and libev.


Build & Install
---------------

The library is built and developed for GNU/Linux systems, as such it may
use GNU GCC and GNU Make specific features.  Patches to support *BSD
kqueue are most welcome.

   * `make all`: The library
   * `make test`: Test and showcase
   * `make install`: Honors `$prefix` and `$DESTDIR` environment variables

Size of libuEv:

    $ make strip
    CC      main.o
    CC      io.o
    CC      timer.o
    CC      signal.o
    ARCHIVE libuev.a
    LINK    libuev.so.1
    STRIP   libuev.a
    STRIP   libuev.so.1
    text	   data	    bss	    dec	    hex	filename
    1177	      0	      0	   1177	    499	main.o     (ex libuev.a)
     308	      0	      0	    308	    134	io.o       (ex libuev.a)
     682	      0	      0	    682	    2aa	timer.o    (ex libuev.a)
     563	      0	      0	    563	    233	signal.o   (ex libuev.a)
    6306	    768	      8	   7082	   1baa	libuev.so.1


Origin & References
--------------------

LibuEv was originally based on
[LibUEvent](http://code.google.com/p/libuevent/) by
[Flemming Madsen](http://www.madsensoft.dk/) but has been completely
rewritten and is now more similar to the famous
[libev](http://software.schmorp.de/pkg/libev.html) by Mark Lehmann.
Another small event library used for inspiration is the very small
[Picoev](https://github.com/kazuho/picoev) by
[Oku Kazuho](https://github.com/kazuho).

   * http://code.google.com/p/libuevent/
   * http://software.schmorp.de/pkg/libev.html
   * http://libev.schmorp.de/bench.html
   * http://libevent.org/
   * http://developer.cybozu.co.jp/archives/kazuho/2009/08/picoev-a-tiny-e.html
   * http://coderepos.org/share/browser/lang/c/picoev/

LibuEv is maintained by [Joachim Nilsson](mailto:troglobit@gmail.com) at
[GitHub](https://github.com/troglobit/libuev)

