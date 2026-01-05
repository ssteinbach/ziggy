# Local Ziggy Patches

This is a local copy of ziggy 0.1.0 (commit `4353b20`) with patches applied to work around a parser bug.

## Why Local Copy?

The ziggy 0.1.0 parser has a bug where it rejects integer tokens when parsing into `f64` fields. For example, `duration = 1` fails while `duration = 1.0` succeeds. This bug manifests when parsing union arrays containing nested unions combined with f64 fields.

This project is pinned to Zig 0.15.2 and cannot upgrade to ziggy versions that require Zig 0.16.0+, where this bug is fixed upstream.

## Bug Details

See `/Users/steinbach/Documents/workspace/wrinkles/ziggy_issue.md` for the full investigation, including:
- Systematic test cases isolating the bug
- Root cause analysis
- Version information

## Applied Patches

### 1. Parser.zig - parseFloat accepts integers

**Location:** `src/ziggy/Parser.zig:442-445`

**Change:** Modified `parseFloat()` to accept both `.float` and `.integer` tokens:
```zig
// Before:
try self.must(num, .float);

// After:
try self.mustAny(num, &.{ .float, .integer });
```

**Rationale:** Integers like `1` are valid float values and should be accepted when parsing into f64 fields.

### 2. Parser.zig - parseUnion accepts null as variant name

**Location:** `src/ziggy/Parser.zig:246-249`

**Change:** Modified `parseUnion()` to accept `.null` token as a union variant name:
```zig
// Before:
try self.must(first_tok, .identifier);

// After:
try self.mustAny(first_tok, &.{ .identifier, .null });
```

**Rationale:** The wrinkles schema uses `null` as a union variant name (e.g., `SerializableMediaDataReference.null`). The tokenizer classifies "null" as a keyword token rather than an identifier, so we need to explicitly accept it.

### 3. ResilientParser.zig - float type accepts integers

**Location:** `src/ziggy/ResilientParser.zig:487-490`

**Change:** Modified schema type-checking to accept `.integer` tokens for float types:
```zig
// Before:
.float => switch (tree.tag) {
    .float => {},

// After:
.float => switch (tree.tag) {
    .float, .integer => {},
```

### 4. Ast.zig - float type accepts integers

**Location:** `src/ziggy/Ast.zig:799-802`

**Change:** Same as ResilientParser.zig - accept `.integer` tokens for float types.

### 5. serializer.zig - force decimal output for floats

**Location:** `src/ziggy/serializer.zig:55-73`

**Change:** Modified float serialization to always include a decimal point for whole numbers:
```zig
// Before:
.float, .comptime_float => try writer.print("{}", .{value}),

// After:
.float, .comptime_float => {
    const v: f64 = @floatCast(value);
    if (@trunc(v) == v and !std.math.isInf(v) and !std.math.isNan(v)) {
        try writer.print("{d:.1}", .{v});
    } else {
        try writer.print("{d}", .{v});
    }
},
```

**Rationale:** Belt-and-suspenders approach - ensures output from Zig serialization always produces parseable format, even if the parser patches were reverted.

## Upstream Status

- **Bug version:** commit `4353b20` (Zig 0.15.x compatible)
- **Fixed version:** commit `6678447` (requires Zig 0.16.0+)
- **Upstream repo:** https://github.com/kristoff-it/ziggy

## Testing

Run the wrinkles test suite to verify patches work:
```bash
zig build test
```

The tests in `src/opentimelineio/serialization.zig` (Tests A-I) specifically exercise the nested union + f64 parsing scenarios.
