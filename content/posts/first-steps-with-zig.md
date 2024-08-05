---
title: "First Steps With Zig"
date: 2024-08-05T18:24:53+02:00
summary: Notes while learning & playing with Zig
tags:
- zig
---

So I've been look to code a bit more this summer. I spent the last year doing security challenges, then working a couple of months in some company that didn't fit my needs, now I'm looking forward to learn some new tech, and I guess [Zig](https://ziglang.org/) would be my next personal challenge.

# A few resources before going in

* [Official docs](https://ziglang.org/documentation/0.13.0/)
* [Learn section](https://ziglang.org/learn/)
* [ziglings](https://codeberg.org/ziglings/exercises/)
* [Zig by example](https://zig-by-example.com/)
* [Zig cookbook](https://cookbook.ziglang.cc/)

# Getting there

So, I've done a first pass on ziglings (quite too quickly, as I've done all exercices during one single day). Main difficulty to me was those "strings" management (hint: there is no string management - yet- in Zig. Strings are `[]const u8` I guess, and you'll have to do pointer conversion to work with those eventually). Anyway, I've started to build a mini `ls` program. You know, this thing to list file.

But let's start with basics. I've installed `Zig` with `home-manager`, along with `zls` so I've got some help in code editor, and that's it.

A basic annotated hello-world program will look like:

```zig
// the standard library is imported. "std" will be used for 
const std = @import("std");

// main must be public, it returns a void, with possibility to return an error (thanks to "!")
pub fn main() !void {
    // getting a stdout fd to write into; In a const, as the value won't change.
    const stdout = std.io.getStdOut().writer();
    // we try to print something. try means an error can be returned.
    try stdout.print("Hello world!", .{});
}
```

Without the annotations:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello world!", .{});
}
```

Building is simple as calling `zig build-exe`:

```shell
$ zig build-exe hello-world.zig
$ eza --bytes --git --group --long -snew --group-directories-first
.rw-r--r--       144 mycroft mycroft  5 Aug 18:43 hello-world.zig
.rw-r--r-- 2,026,000 mycroft mycroft  5 Aug 18:44 hello-world.o
.rwxr-xr-x 2,239,816 mycroft mycroft  5 Aug 18:44 hello-world
$ ./hello-world
Hello world!
```

# Writing something to list files

Ok, we've got a running Zig compiler. Now is time to build `ls`.

A first try, my first try, was looking like:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const wd = std.fs.cwd();
    var dir: std.fs.Dir = wd.openDir(".", .{ .iterate = true }) catch |err| {
        try stdout.print("could not open directory: {any}", .{err});
        return;
    };

    // wd & dir are both "Dir" structures. However, "wd" returned by std.fs.cwd is not iterable.
    var it = dir.iterate();

    while (try it.next()) |entry| {
        try stdout.print("{s}\n", .{entry.name});
    }
}
```

I got some real issue with having 2 `Dir` structures pointing to the same directory. This works. But well...

Let's try to open a directory but its absolute path. This requires using `realpathAlloc`, meaning we'll have to create a memory allocator and free memory after use.

```zig
const std = @import("std");

fn get_cwd(allocator: std.mem.Allocator) ![]u8 {
    return try std.fs.cwd().realpathAlloc(allocator, ".");
}

pub fn main() !void {
    // getting an allocator that'll be used in realpathAlloc
    const allocator = std.heap.page_allocator;

    const stdout = std.io.getStdOut().writer();
    const stderr = std.io.getStdErr().writer();

    const wd = try get_cwd(allocator);
    defer allocator.free(wd);

    try stderr.print("current dir: {s}\n", .{wd});

    // our call to openDir with the iterate flag
    var iterable_dir = try std.fs.openDirAbsolute(wd, .{ .iterate = true });

    var it = iterable_dir.iterate();

    // actually listing files!
    while (try it.next()) |entry| {
        try stdout.print("{s}\n", .{entry.name});
    }
}
```

This will output:

```shell
$ zig build-exe ls.zig
$ ./ls
current dir: /home/mycroft/dev/zig-bla
hello-world.o
hello-world
hello-world.zig
ls.zig
ls.o
ls
```

Now would be a good time to sort the thing. We'll have to store all entries, then sort them, thne print them. Sorting strings in Zig is not as easy as it should be. It is required to use `std.mem.sort` and build a compare function:

```zig

// ...

fn compareStrings(_: void, lhs: []const u8, rhs: []const u8) bool {
    return std.mem.order(u8, lhs, rhs).compare(std.math.CompareOperator.lt);
}

pub fn main() !void {
    // ...
    // the following is coming at the end of main
    var entries = std.ArrayList([]const u8).init(allocator);
    defer entries.deinit();

    while (try it.next()) |entry| {
        // entry is duplicated as the buffer is reused by next()
        try entries.append(try std.heap.page_allocator.dupe(u8, entry.name));
    }

    std.mem.sort([]const u8, entries.items, {}, compareStrings);

    for (entries.items) |entry| {
        try stdout.print("{s}\n", .{entry});
    }
}
```

This will show sorted files. Next step will be to print a little more metadatas of listed files:

```zig
pub fn main() !void {
    // ...
    for (entries.items) |entry| {
        const stat = iterable_dir.statFile(entry) catch {
            continue;
        };

        const file_type: u8 = switch (stat.kind) {
            std.fs.File.Kind.directory => 'd',
            std.fs.File.Kind.file => '.',
            std.fs.File.Kind.sym_link => 's',
            else => '?',
        };

        try stdout.print("{c} {d:8} {o} {s}\n", .{ file_type, stat.size, stat.mode % 0o1000, entry });
    }
    // ...
}
```

As a result:

```shell
$ ./ls
.  2239816 755 hello-world
.  2026000 644 hello-world.o
.      144 644 hello-world.zig
.  2367776 755 ls
.  2201576 644 ls.o
.     1546 644 ls.zig
```

Now that we have more details, we might want to have an argument flag to list only non hidden files, only filenames, etc. As a bonus, last unparsed argument will be used as directory to open.

```zig
pub fn main() !void {
    // ...
    var opt_long = false;
    var opt_hidden = false;
    var wd = try get_cwd(allocator);
    defer allocator.free(wd);

    for (std.os.argv[1..]) |arg| {
        const argf = std.mem.span(arg);

        if (std.mem.eql(u8, argf, "-l")) {
            opt_long = true;
        } else if (std.mem.eql(u8, argf, "-a")) {
            opt_hidden = true;
        } else {
            wd = try std.heap.page_allocator.dupe(u8, argf);
        }
    }

    // ...

    for (entries.items) |entry| {
        // ...

        if (!opt_hidden and entry[0] == '.') {
            continue;
        }

        // ...

        if (opt_long) {
            try stdout.print("{c} {d:8} {o} {s}\n", .{ file_type, stat.size, stat.mode % 0o1000, entry });
        } else {
            try stdout.print("{s}\n", .{entry});
        }
    }
```

It semes this does the job.

Last, I'm not really comfortable with the allocator. I'm afraid that something leaks. To check if nothing is wrong, I'll use `GeneralPurposeAllocator` to check possible leak or errors. Make sure to replace all `std.heap.page_allocator` with `allocator`:

```zig
pub fn main() !void {
    // getting an allocator that'll be used in realpathAlloc
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer std.debug.assert(gpa.deinit() == .ok);

    const allocator = gpa.allocator();

    for (std.os.argv[1..]) |arg| {
        const argf = std.mem.span(arg);

        // ...
        } else {
            allocator.free(wd);
            wd = try allocator.dupe(u8, argf);
        }
    }
    defer allocator.free(wd);

    // ...
    while (try it.next()) |entry| {
        try entries.append(try allocator.dupe(u8, entry.name));
    }

    // ...
    for (entries.items) |entry| {
        defer allocator.free(entry);

        // ...
    }
}
```

If there is a double free, or leak, `gpa.deinit()` won't be `.ok` and there will be some message print on output:

```sh
$ ./ls
...
error(gpa): memory address 0x753bfe1a2000 leaked:
/nix/store/5yk32f31879lfsnyv0yhl0af0v2dz9dz-zig-0.13.0/lib/zig/std/mem/Allocator.zig:319:40: 0x103b307 in dupe__anon_3893 (ls)
    const new_buf = try allocator.alloc(T, m.len);

```

If everything is all right, no error will be reported.

# Wrap-up

In this introductory article, a small `ls` implementation was quickly written in Zig. We sorted strings, played with command line arguments and memory using a basic heap allocator.
