---
title: "Zig: Using fieldParentPtr"
date: 2024-09-22T10:33:01+02:00
summary: Using fieldParentPtr to generically handle kind of object values
tags:
- zig
---

`fieldParentPtr` can be useful in case it is required to create multiple kind of objects all based on a common type that can be passed to generic functions and then coerced to their original type. A usecase for this is a `Value` object that then will contain `string`, `number`, `bool`, `function`, `instance` when building a new language.

```sh
$ mkdir zig-objects && cd zig-objects
$ zig init
info: created build.zig
info: created build.zig.zon
info: created src/main.zig
info: created src/root.zig
info: see `zig build --help` for a menu of options
```

Defining the object in `object.zig`:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Self is our Object struct/module that will be the root object for each our types
const Self = @This();

// Defining object types
pub const ObjType = enum {
    String,
};

// the root module is storing allocator used for creating objects, and object's type
// other things can be stored here as well, as long as they are common to all types
allocator: Allocator,
kind: ObjType,

// generic object creator, that will create parent object and will fill root object.
// this function will only be called from typed object new() function, so it will be
// private to the module
fn new(comptime T: type, allocator: Allocator, kind: ObjType) *T {
    const typed_obj = allocator.create(T) catch unreachable;

    // fill the root object
    typed_obj.obj = Self{
        .allocator = allocator,
        .kind = kind,
    };

    return typed_obj;
}

// a parent String object, with final value and root object
pub const String = struct {
    obj: Self,
    chars: []const u8,

    // String object creation
    pub fn new(allocator: Allocator, chars: []const u8) *String {
        const str_obj = Self.new(String, allocator, ObjType.String);

        str_obj.chars = allocator.dupe(u8, chars) catch unreachable;

        return str_obj;
    }

    pub fn destroy(self: *String) void {
        self.obj.allocator.free(self.chars);
        self.obj.allocator.destroy(self);
    }
};

// helpers for the String object
pub fn isString(self: *Self) bool {
    return self.kind == ObjType.String;
}

// asString takes a generic Object, and if it is a ObjType.String, returns the parent String
pub fn asString(self: *Self) *String {
    std.debug.assert(self.isString());
    return @fieldParentPtr("obj", self);
}
```

Adding a new kind of Object is as simple as adding a new ObjType, creating the struct, and adding helpers:

```zig
pub const ObjType = enum {
    String,
    Number,
};

pub const Number = struct {
    obj: Self,
    num: u32,

    pub fn new(allocator: Allocator, num: u32) *Number {
        const num_obj = Self.new(Number, allocator, ObjType.Number);

        num_obj.num = num;

        return num_obj;
    }

    pub fn destroy(self: *Number) void {
        self.obj.allocator.destroy(self);
    }
};

pub fn isNumber(self: *Self) bool {
    return self.kind == ObjType.Number;
}

// asNumber will retrieve from a generic root object its parent typed object
pub fn asNumber(self: *Self) *Number {
    std.debug.assert(self.isNumber());
    return @fieldParentPtr("obj", self);
}
```

Writing generic helpers is fairly easy as it is just a matter of adding switch cases:

```zig
const std = @import("std");

const Object = @import("object.zig");
const String = Object.String;
const Number = Object.Number;

pub fn dump(obj: *Object) void {
    switch (obj.kind) {
        Object.ObjType.String => std.debug.print("String: {s}\n", .{obj.asString().chars}),
        Object.ObjType.Number => std.debug.print("Number: {d}\n", .{obj.asNumber().num}),
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{ .safety = true }){};
    defer _ = std.debug.assert(gpa.deinit() == .ok);
    const allocator = gpa.allocator();

    const str_obj = String.new(allocator, "testaroo");
    const num_obj = Number.new(allocator, 42);

    // std.debug.print("{}: {s}\n", .{ str_obj.obj.kind, str_obj.chars });
    // std.debug.print("{}: {d}\n", .{ num_obj.obj.kind, num_obj.num });
    dump(&str_obj.obj);
    dump(&num_obj.obj);

    str_obj.destroy();
    num_obj.destroy();
}
```

Building and running:

```zig
$ zig build run
String: testaroo
Number: 42
```

## Resources

- [Zig's @fieldParentPtr for dumbos like me](https://www.ryanliptak.com/blog/zig-fieldparentptr-for-dumbos/)
- [@fieldParentPtr](https://ziglang.org/documentation/master/#fieldParentPtr)

