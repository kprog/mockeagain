Description
===========

This tool emulates an ideally slow network by mocking the
polling and read/write syscalls exposed by glibc via the
LD_PRELOAD technique.

By preloading this dynamic library to your network server
process, it will intercept the "poll", "close", "send", and "writev"
syscalls, only allow the writing syscalls to actually write a single byte
at a time (without flushing), and returns EAGAIN until another
"poll" called on the current socket fd.

The socket fd must be called first by a "poll" call to mark
itself to this tool as an "active fd" and trigger subsequence
the writing syscalls to behave differently.

With this tool, one can emulate extreme network conditions
even locally (with the loopback device).

Build
=====

Just issue the following command to build the file mockeagain.so

    make

The Gnu make and gcc compiler are required.

On *BSD, it's often required to run the command "gmake".

Usage
=====

For now, only "poll" syscall and slow writing mocking are supported.

Here we take Nginx as an example:

1. Ensure that you have passed the --with-poll_module option while
   invoking nginx's configure script to build nginx.

2. Ensure that you've enabled the "poll" event model in your nginx.conf:

    events {
        use poll;
        worker_connections 1024;
    }

3. Ensure that you've disabled nginx's own write buffer:

    postpone_output 1; # only postpone a single byte, default 1460 bytes

and also have the following line in your nginx.conf:

    env LD_PRELOAD;

4. Run your Nginx this way:

    LD_PRELOAD=/path/to/mockeagain.so /path/to/nginx ...

Environments
============

The following system environment variables can be used
to control the behavior of this tool. You should
use Nginx's standard "env" directive to enable the
environments that you use in your nginx.conf file, as in

    env MOCKEAGAIN_VERBOSE;
    env MOCKEAGAIN;
    env MOCKEAGAIN_WRITE_TIMEOUT_PATTERN;

If you're using the Test::Nginx test scaffold, then these directives are
automatically configured for you.

MOCKEAGAIN
----------

This environment controls whether to mocking reads or writes or both.

When the letter "r" or "R" appear in the value string, then reads will be mocked.

When the letter "w" or "W" appear in the value string, then writes will be mocked.

When this environment is either not set or set to an empty string, then writes will be mocked by default.

MOCKEAGAIN_VERBOSE
------------------

You can set the environment MOCKEAGAIN_VERBOSE to 1 if you want more diagnostical outputs of this tool in stderr. But you'll also have to add the following line to your nginx.conf:

    env MOCKEAGAIN_VERBOSE;

When setting MOCKEAGAIN_VERBOSE to 1, you'll get messages like the following in your nginx's error.log file:

    mockeagain: mocking "writev" on fd 3 to signal EAGAIN.
    mockeagain: mocking "writev" on fd 3 to emit 1 of 188 bytes.
    mockeagain: mocking "writev" on fd 3 to emit 1 of 187 bytes.
    mockeagain: mocking "recv" on fd 5 to read 1 byte of data only

MOCKEAGAIN_WRITE_TIMEOUT_PATTERN
--------------------------------

When this environment is set to value "foo bar", then the when the string "foo bar" appears in the output stream (not necessarily in a single write call), then it will trigger an indefinite write timeout.

This feature is very useful in mocking a write timeout in a particular position in the output stream.

Note that this environment also requires that the MOCKEAGAIN variable value contains "w" or "W".

For now, this feature only supports the "writev" call.

Glibc API Mocked
----------------

Event API
* poll

Writing API
* writev
* send

Reading API
* read
* recv
* recvfrom

TODO
====

* add support for write, sendto, and more writing syscalls.
* add support for other event interfaces like epoll, select, kqueue, and more.
* add support for slow read mocking, like the recvfrom syscall.

Success Stories
===============

ngx_echo
--------

The echo_after_body directive used to have a bug that the output filter
failed to take into account the NGX_AGAIN constant returned from
the downstream output filter chain. Enabling mockeagain's writing mode
can trivially capture this issue even on localhost.

To reproduce, update ngx_http_echo_filter.c:165

    #if 0

to

    #if 1

and run ngx_echo's test file t/echo-after-body.t with mockeagain's writing mode enabled.

ngx_redis
---------

ngx_redis 0.3.5 and earlier had a subtle bug in its redis reply parser
as explained here:

    https://groups.google.com/group/openresty/browse_thread/thread/b5ddf1b24d3c9677

Even though this bug was first captured by the [etcproxy](https://github.com/chaoslawful/etcproxy) tool, but mockeagain's reading mode
can also capture it reliably even on localhost.

To reproduce, just compile ngx_redis 0.3.5 with ngx_srcache, and
then run the t/redis.t test file in ngx_srcache's test suite, and with mockeagain reading mode enabled.

libdrizzle
----------

libdrizzle 0.8 and earlier had a subtle bug in its MySQL packet parser
and mockeagain's reading mode can capture it reliably even on localhost.

To reproduce, just compile libdrizzle 0.8 with ngx_drizzle, and run the t/keepalive.t test file in ngx_drizzle's test suite
with mockeagain's reading mode enabled.

Author
======

Zhang "agentzh" Yichun (章亦春)

Copyright & License
===================

This module is licenced under the BSD license.

Copyright (C) 2011 by Zhang "agentzh" Yichun (章亦春) <agentzh@gmail.com>.

All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

    * Redistributions of source code must retain the above copyright
    notice, this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above copyright
    notice, this list of conditions and the following disclaimer in the
    documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED
TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

