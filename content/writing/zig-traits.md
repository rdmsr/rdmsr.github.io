+++
title = "Poor man's traits in Zig"
date = 2026-09-09

[extra]
type = "Post"
+++

Zig is a pretty nice programming language, but it unfortunately does not support compiler-enforced interfaces or traits. Instead, the language relies on duck typing to operate on generic types.


Luckily, Zig supports static reflection and compile-time code evaluation, so it is quite easy to implement "Poor man's traits":

```zig
/// Asserts that a given type `T` matches the schema declared by `I`.
/// This includes public methods and fields.
pub fn assert_interface(T: type, I: type) void {
    const tinfo = @typeInfo(I);

    inline for (tinfo.@"struct".fields) |f| {
        if (!@hasField(T, f.name)) {
            @compileError(std.fmt.comptimePrint("Expected field '{s}' of type '{s}' in type '{s}' required by '{s}'", .{ f.name, @typeName(f.type), @typeName(T), @typeName(I) }));
        } else {
            if (@FieldType(T, f.name) != f.type) {
                @compileError(std.fmt.comptimePrint("Expected field '{s}' of type '{s}' in type '{s}' required by '{s}', got '{s}'", .{ f.name, @typeName(f.type), @typeName(T), @typeName(I), @typeName(@FieldType(T, f.name)) }));
            }
        }
    }

    inline for (tinfo.@"struct".decls) |decl| {
        const member = @field(I, decl.name);

        if (@typeInfo(@TypeOf(member)) == .@"fn") {
            if (!@hasDecl(T, decl.name)) {
                @compileError(std.fmt.comptimePrint("Expected method '{s}' in type '{s}' required by '{s}'", .{ decl.name, @typeName(T), @typeName(I) }));
            }

            const impl_member = @field(T, decl.name);
            const IfaceFnType = @TypeOf(member);
            const ImplFnType = @TypeOf(impl_member);

            if (IfaceFnType != ImplFnType) {
                const iface_fn = @typeInfo(IfaceFnType).@"fn";
                const impl_fn = @typeInfo(ImplFnType).@"fn";

                if (iface_fn.params.len != impl_fn.params.len) {
                    @compileError(std.fmt.comptimePrint("Parameter count mismatch in method '{s}' in type '{s}' required by '{s}'", .{ decl.name, @typeName(T), @typeName(I) }));
                }
                if (iface_fn.return_type != impl_fn.return_type) {
                    @compileError(std.fmt.comptimePrint("Return type mismatch in method '{s}' in type '{s}' required by '{s}'", .{ decl.name, @typeName(T), @typeName(I) }));
                }

                for (0..iface_fn.params.len) |i| {
                    if (iface_fn.params[i].type != impl_fn.params[i].type) {
                        @compileError(std.fmt.comptimePrint("Parameter type mismatch in method '{s}' in type '{s}' required by '{s}'", .{ decl.name, @typeName(T), @typeName(I) }));
                    }
                }
            }
        } else if (@TypeOf(member) == type) {
            if (!@hasDecl(T, decl.name)) {
                @compileError(std.fmt.comptimePrint("Expected type declaration '{s}' in type '{s}' required by '{s}'", .{ decl.name, @typeName(T), @typeName(I) }));
            }

            const impl_member = @field(T, decl.name);
            if (@TypeOf(impl_member) != type) {
                @compileError(std.fmt.comptimePrint("Declaration '{s}' in type '{s}' is required to be a type by '{s}', but got '{s}'", .{ decl.name, @typeName(T), @typeName(I), @typeName(@TypeOf(impl_member)) }));
            }
        } else {
            if (!@hasDecl(T, decl.name)) {
                @compileError(std.fmt.comptimePrint("Expected variable declaration '{s}' in type '{s}' required by '{s}'", .{ decl.name, @typeName(T), @typeName(I) }));
            }

            const impl_member = @field(T, decl.name);
            if (@TypeOf(impl_member) != @TypeOf(member)) {
                @compileError(std.fmt.comptimePrint("Declaration '{s}' in type '{s}' is required to be a variable of type '{s}' by '{s}', but got '{s}'", .{ decl.name, @typeName(T), @typeName(@TypeOf(member)), @typeName(I), @typeName(@TypeOf(impl_member)) }));
            }
        }
    }
}
```

This would then be used by declaring a "schema" and asserting against it:

```zig
const AnimalSchema = struct {
    pub const sound: []const u8 = "";
    pub fn pet() void {}
};

const Dog = struct {
    pub const sound: []const u8 = "bark";

    pub fn pet() void {
        // Do something
    }
};

const Cat = struct {
    pub const sound: []const u8 = "meow";

    pub fn pet() void {
        // Do something
    }
};

fn pet_animal(comptime T: type) void {
    assert_interface(AnimalSchema, T);
    T.pet();
}

pet_animal(Dog);
pet_animal(Cat);
```
