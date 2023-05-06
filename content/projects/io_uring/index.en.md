+++
title = "Making a cp implementation with liburing and rust"
date = 2023-04-12
draft = false 
+++

Async I/O in linux has made a lot of progress, namely in the form of
[liburing](https://kernel.dk/io_uring.pdf), which allows for "submitting" requests for system calls
without having to call them from userspace directly. Being a kernel API, it is, of course, exposed to consumers through C.

However, these last few years I have taken interest in Rust, mostly because of its bold assertions regarding memory safety and
overall polished nature of the language (in my opinion anyway), and I thought it would be interesting to leverage
its memory guarantees for my use case.

## Naive cp implementation at a glance

A "naive" cp implementation in rust might look something like this:
```rust
fn naive_cp(in_path: &PathBuf, out_path: &PathBuf) -> Result<()> {
    let mut in_file = File::open(in_path)?;
    let mut out_file = File::create(out_path)?;
    let mut buffer = [0u8; 1024 * entries];
    loop {
        let bytes_read = in_file.read(&mut buffer)?;
        if bytes_read == 0 {
            return Ok(());
        }
        out_file.write_all(&buffer[0..bytes_read])?;
    }
}
```
This repeatedly calls into [`read(2)`](https://man7.org/linux/man-pages/man2/read.2.html) and [`write(2)`](https://man7.org/linux/man-pages/man2/write.2.html), making it inefficent
for large files.

## First approach

To truly make use of the bulk I/O transfer capabilities allowed to use,
I'm going to make several [`preadv2`](https://man7.org/linux/man-pages/man2/readv.2.html) calls chained, followed by their 
corresponding [`pwritev2`](https://man7.org/linux/man-pages/man2/readv.2.html) into the output file.

Vectored reads "scatter" data from a file descriptor, into a collection of user-provided buffer,
more specifically, an `struct iovec` which has the following structure:

```c
struct iovec {
   void  *iov_base;    /* Starting address */
   size_t iov_len;     /* Number of bytes to transfer */
};
```

effectively reading into an array of buffers of len `IOVCNT`, per the man pages.
However, if we want to make several vectored reads (followed by several vectored writes), our buffer needs to be mapped into an 
`ENTRIES x IOVCNT` matrix of `iovec`s
```
                                                    

                                  IOVCNT                                            
               /----------------------------------------\                           
              /                                          \                          
      +--------------|--------------|-------------|--------------+                  
      | IOVEC_BUFLEN |     iovec    |    iovec    |    iovec     |    -             
      ------------------------------------------------------------     \            
      |              |              |             |              |     |            
      ------------------------------------------------------------     |   ENTRIES  
      |              |              |             |              |     |            
      ------------------------------------------------------------     |            
      |              |              |             |              |     /            
      +--------------|--------------|-------------|--------------+    -             
                                                    
                                                    
```

We make our first attempt at reading in bulk as such:

```rust
const ENTRIES: usize = 32;
const IOVCNT: usize = 16;
const IOVCNT_BUFLEN: usize = 32;
const BUFSIZE: usize = ENTRIES * IOVCNT * IOVCNT_BUFLEN;
let mut buffer = [0u8; BUFSIZE];
let mut io_uring = IoUring::new(ENTRIES as u32)?;
let in_file = File::open(in_path)?;
// We want to iterate over the "rows" of the matrix, and array_chunks_mut gives
// us mutable references to it
for slice in buffer.array_chunks_mut::<{ IOVCNT * IOVCNT_BUFLEN }>() {
    // Slices need to be mutable, as the kernel is going to write into them!
    let mut read_chunks: Vec<IoSliceMut> = slice
        // Divide the row into equal sized buffers
        .chunks_exact_mut(IOVCNT_BUFLEN)
        // Create an IoSliceMut from each individual buffer
        .map(IoSliceMut::new)
        // Collect into a vector
        .collect();

    assert_eq!(IOVCNT, read_chunks.len());

    let readv_e = opcode::Readv::new(
        Fd(in_file.as_raw_fd()),
        // Pass the starting address of that vector to the call
        read_chunks.as_mut_ptr().cast(),
        IOVCNT as _,
    )
    .build()
    // Make completions appear in the same order as they were requested
    .flags(squeue::Flags::IO_LINK);
    unsafe {
        // Submit read events
        io_uring.submission().push(&readv_e)?;
    }
}

// Block on completions
io_uring.submit_and_wait(ENTRIES)?;
let completion_results: Vec<i32> = io_uring.completion().map(|cqe| cqe.result()).collect();
assert!(completion_results.iter().all(|read_res| *read_res > 0));
```

But alas! this doesn't work. `preadv2()` either returns a positive integer or 0 if the call was successful and a negative
integer indicating an error. In this case, the error alternated between `EINVAL` and `EFAULT`. Something funky was happening 
with the pointers we are sending to `preadv2()`.

On further inspection, it was noted to me that the references to `read_chunks` where being dropped on each iteration, meaning
that they weren't valid pointers being sent to the kernel. Just another case of subtle memory errors being caught by the compiler.

We need the references we send to `push()` to outlive the reading operation, so on further refinement, our bulk read implementation with
its corresponding bulk write looks something like this:

```rust
const ENTRIES: usize = 32;
const IOVCNT: usize = 16;
const IOVEC_BUFLEN: usize = 32;
const BUFSIZE: usize = ENTRIES * IOVCNT * IOVEC_BUFLEN;
let mut buffer = [0u8; BUFSIZE];
let mut io_uring = IoUring::new(ENTRIES as u32)?;
let in_file = File::open(in_path)?;
let file_size = in_file.metadata()?.size();
let out_file = File::create(out_path)?;

let needed_rows = (file_size as usize).div_ceil(IOVCNT * IOVEC_BUFLEN);
// Slices need to be mutable, as the kernel is going to write into them!
let mut read_slices: Vec<_> = buffer
    // Divide the row into equal sized buffers
    .array_chunks_mut::<IOVEC_BUFLEN>()
    // Array chunks returns "little arrays", but IoSliceMut::new()
    // Takes slices as input
    .map(AsMut::as_mut)
    // Create an IoSliceMut from each individual buffer, which 
    // rust coerces into an iovec
    .map(IoSliceMut::new)
    // Collect into a vector. We might want to optimize this so 
    // no allocations occur.
    .collect();

assert_eq!(read_slices.len(), IOVCNT * ENTRIES);
assert!(needed_rows <= ENTRIES);

let readv_es: Vec<_> = read_slices
    // Take up to IOVCNT IoSliceMut's at a time
    .chunks_exact_mut(IOVCNT)
    // We need to take into account the offset argument we are passing to 
    // preadv2
    .zip((0..BUFSIZE).step_by(IOVEC_BUFLEN * IOVCNT))
    .map(|(slice, offset)| {
        opcode::Readv::new(
            Fd(in_file.as_raw_fd()),
            slice.as_mut_ptr().cast(),
            IOVCNT as _,
        )
        .offset(offset as u64)
        .build()
        // Make reads sequential (Might not need this)
        .flags(squeue::Flags::IO_LINK)
        // Unchanged by the kernel on completion, but helps us with debugging
        .user_data(0x01)
    })
    // We might not need the whole buffer, so we only take the "rows" we need
    .take(needed_rows)
    .collect();

assert_eq!(needed_rows, readv_es.len());
for e in readv_es {
    unsafe {
        io_uring.submission().push(&e)?;
    }
}

io_uring.submit_and_wait(needed_rows)?;
let completion_results: Vec<i32> = io_uring.completion().map(|cqe| cqe.result()).collect();
let read_bytes: i32 = completion_results.iter().sum();
let original_contents = fs::read_to_string(in_path)?;
let read_contents = String::from_utf8(buffer[0..(read_bytes as usize)].into())?;
// Ensure we read correctly, will be removed afterwards
assert_eq!(original_contents, read_contents);

// Collect preadv2() call results, might need to check we got no OS errors
let actual_bytes_read: i32 = completion_results.iter().sum();

// We might have read less bytes than we expected, so we need to account
// for that by 'shortening' our buffer, dividing it up into IOVEC_BUFLEN
// sizes and chaining the remaining, possibly smaller buffer (if any)
let bytes_read_buffer = &buffer[..actual_bytes_read as usize];
let write_buf_chunks = bytes_read_buffer.array_chunks::<IOVEC_BUFLEN>();
let rem = write_buf_chunks.remainder();

let write_slices: Vec<_> = write_buf_chunks
    .map(AsRef::as_ref)
    .chain(std::iter::once(rem))
    .map(IoSlice::new)
    .collect();

let write_sqes: Vec<_> = write_slices
    .chunks(IOVCNT)
    .zip((0..actual_bytes_read).step_by(IOVEC_BUFLEN * IOVCNT))
    .map(|(slice, offset)| {
        // The slice we got here might be smaller than the row size,
        // and the actual iovecs we need might be less than IOVECNT
        let underlying_memory_size = slice.iter().fold(0, |acc, s| acc + s.len());
        let iovecnt = underlying_memory_size.div_ceil(IOVEC_BUFLEN);
        opcode::Writev::new(
            Fd(out_file.as_raw_fd()),
            slice.as_ptr().cast(),
            iovecnt as _,
        )
        .offset(offset as u64)
        .build()
        .flags(squeue::Flags::IO_LINK)
        .user_data(0x02)
    })
    .collect();

let write_len = write_sqes.len();
assert_eq!(write_len, needed_rows);
for e in write_sqes {
    unsafe {
        io_uring.submission().push(&e)?;
    }
}
io_uring.submit_and_wait(needed_rows)?;
```

and... it works! Not without a little tweaking first though, but it does work.
But currently, it only works because our buffer is bigger than our input file, so
we need to put this inside a loop. Additionally, it would be nice to do this without any
allocations whatsoever...


## The final product

Brace yourselves, as our buffer arithmetics need to be on point for this one:

```rust
fn liburing_cp(in_path: &PathBuf, out_path: &PathBuf) -> Result<()> {
    const ENTRIES: usize = 8;
    const IOVCNT: usize = 8;
    const IOVEC_BUFLEN: usize = 32;
    const BUFSIZE: usize = ENTRIES * IOVCNT * IOVEC_BUFLEN;
    let mut buffer = [0u8; BUFSIZE];
    let mut io_uring = IoUring::new(ENTRIES as u32)?;
    let in_file = File::open(in_path)?;
    let file_size = in_file.metadata()?.size();
    let out_file = File::create(out_path)?;

    // We need to know how much of the file we are going to be reading off every iteration
    let whole_chunks = file_size as usize / BUFSIZE;
    let partial_chunk = file_size as usize % BUFSIZE;
    let mut file_chunk_sizes = (0..whole_chunks).map(|_| BUFSIZE).collect::<Vec<_>>();
    file_chunk_sizes.push(partial_chunk);
    let offsets = file_chunk_sizes
        .into_iter()
        .enumerate()
        .map(|(i, chunk_size)| (chunk_size, i * BUFSIZE, i * BUFSIZE + chunk_size));

    let mut preadv2_sqes = Vec::with_capacity(ENTRIES);
    let mut pwritev2_sqes = Vec::with_capacity(ENTRIES);

    for (file_chunk_sz, start, end) in offsets {
        assert!(file_chunk_sz <= BUFSIZE);
        // At most, this will always be ENTRIES
        let needed_rows = file_chunk_sz.div_ceil(IOVCNT * IOVEC_BUFLEN);
        assert!(needed_rows <= ENTRIES);

        // Slices need to be mutable, as the kernel is going to write into them!
        let mut read_slices: Vec<_> = buffer
            // Divide the row into equal sized buffers
            .array_chunks_mut::<IOVEC_BUFLEN>()
            // Array chunks returns "little arrays", but IoSliceMut::new()
            // Takes slices as input
            .map(AsMut::as_mut)
            // Array chunks returns "little arrays", but IoSliceMut::new()
            // Takes slices as input
            .map(IoSliceMut::new)
            // Collect into a vector. We might want to optimize this so
            // no allocations occur.
            .collect();

        // Drop previously inserted sqes
        preadv2_sqes.truncate(0);
        read_slices
            // Take up to IOVCNT IoSliceMut's at a time
            .chunks_exact_mut(IOVCNT)
            // We need to take into account the offset argument we are passing to
            // preadv2
            .zip((start..end).step_by(IOVEC_BUFLEN * IOVCNT))
            .map(|(slice, offset)| {
                opcode::Readv::new(
                    Fd(in_file.as_raw_fd()),
                    slice.as_mut_ptr().cast(),
                    IOVCNT as _,
                )
                .offset(offset as u64)
                .build()
                // Unchanged by the kernel on completion, but helps us with debugging
                .user_data(0x01)
            })
            // We might not need the whole buffer, so we only take the "rows" we need
            .take(needed_rows)
            .collect_into(&mut preadv2_sqes);

        assert_eq!(preadv2_sqes.len(), needed_rows);
        for e in &preadv2_sqes {
            unsafe {
                io_uring.submission().push(e)?;
            }
        }

        io_uring.submit_and_wait(needed_rows)?;
        let completion_results: Vec<i32> = io_uring.completion().map(|cqe| cqe.result()).collect();

        let actual_bytes_read: i32 = completion_results.iter().sum();

        // We might have read less bytes than we expected, so we need to account
        // for that by 'shortening' our buffer, dividing it up into IOVEC_BUFLEN
        // sizes and chaining the remaining, possibly smaller buffer (if any)
        let bytes_read_buffer = &buffer[..actual_bytes_read as usize];
        let write_buf_chunks = bytes_read_buffer.array_chunks::<IOVEC_BUFLEN>();
        let rem = write_buf_chunks.remainder();

        let write_slices: Vec<_> = write_buf_chunks
            .map(AsRef::as_ref)
            .chain(std::iter::once(rem))
            .map(IoSlice::new)
            .collect();

        // Drop previously inserted sqes
        pwritev2_sqes.truncate(0);
        write_slices
            .chunks(IOVCNT)
            .zip((start..start + actual_bytes_read as usize).step_by(IOVEC_BUFLEN * IOVCNT))
            .map(|(slice, offset)| {
                // The slice we got here might be smaller than the row size,
                // and the actual iovecs we need might be less than IOVECNT
                let underlying_memory_size = slice.iter().fold(0, |acc, s| acc + s.len());
                let iovecnt = underlying_memory_size.div_ceil(IOVEC_BUFLEN);
                opcode::Writev::new(
                    Fd(out_file.as_raw_fd()),
                    slice.as_ptr().cast(),
                    iovecnt as _,
                )
                .offset(offset as u64)
                .build()
                .flags(squeue::Flags::IO_LINK)
                .user_data(0x02)
            })
            .take(needed_rows)
            .collect_into(&mut pwritev2_sqes);

        assert_eq!(pwritev2_sqes.len(), needed_rows);
        for e in &pwritev2_sqes {
            unsafe {
                io_uring.submission().push(e)?;
            }
        }
        io_uring.submit_and_wait(needed_rows)?;
        // We need to consume the iterator, otherwise remaining sqes will remain on our final
        // iteration!
        let _: Vec<i32> = io_uring.completion().map(|cqe| cqe.result()).collect();
    }

    Ok(())
}

```

Now, it _is_ kind of hard to follow, specially when it comes to all of the offsets involved, but I found that rust's functional
programming style allowed me to focus on the actual buffer arithmetic and system call parameters, rather than focusing on tricky loop
variables.

Now we just call it from our `main()` as such:
```rust
fn main() {
    let args: Vec<String> = std::env::args().collect();
    let (in_path, out_path) = cmd::parse_args(&args).expect("args error");
    liburing_cp(&in_path, &out_path).expect("uring cp");
    let original_contents = fs::read_to_string(&in_path).expect("reading original file");
    let actual_contents = fs::read_to_string(&out_path).expect("reading output file");
    assert_eq!(original_contents, actual_contents);
}
```
And TADA!, we are done.

## Wrapping up 

I have to confess that this probably wasn't the most _rustic_ approach. [rio](https://docs.rs/rio/0.9.4/rio/) exposes
an API that doesn't need to reach into `unsafe` code, but where is the fun in that? Overall, even this basic
wrapper around the API seems to show great promise for speeding I/O bound programs!

P.S, the inspiration for this post came from [here](https://unixism.net/loti/tutorial/cp_liburing.html)

