+++
title = "What actually is io_uring?"
date = 2023-04-12
draft = true
+++

Ever since it became economically feasible to have multiple cores on a single computer, 
asynchronous programing has been a thing. The unix especification has attempted to tackle async I/O
from several syscalls, most notably `aio_read(3)` and `aio_write(3)`.

For several reasons, these system calls were inadequate for many use cases. 
[Long story short](https://kernel.dk/io_uring.pdf): async i/o is complex, and system calls expensive.
