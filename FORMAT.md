# `hxb` format

- [`hxb` file format](#hxb-file-format)
- [Primitives](#primitives)
- [Haxe](#haxe)

---

## `hxb` file format

A `hxb` file has the following structure:

- : `"hxb1"` (magic bytes and version)
- [Header](#header) : header chunk
- [Chunk](#chunk)... : data chunks:
  - [StringPool](#stringpool)
  - [DocPool](#docpool)
  - [TypeList](#typelist)
  - [FieldList](#fieldlist)
  - [TypeDeclarations](#typedeclarations)
  - [ModuleExtra](#moduleextra)
- [End](#end) : end chunk

A `hxb` file corresponds to exactly one Haxe module (`module_def` in `type.ml`).

### `Chunk`

The general structure of a chunk is based on PNG chunks. See [PNG Chunks](http://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html) and [PNG Structure - Chunk naming conventions](http://www.libpng.org/pub/png/spec/1.2/PNG-Structure.html#Chunk-naming-conventions).

A "critical" chunk is necessary to use the `hxb` file for AST serialisation and deserialisation purposes.

A "public" chunk is specified in this document. Applications may define their own chunks which will be "private".

- [u32](#u32) : size of data (not including the header or the checksum)
- ... : 4-byte string identifier consisting of uppercase and lowercase letters only; the case of each letter indicates chunk properties:
  - bool : case of first byte - uppercase = critical chunk, lowercase = ancillary chunk
  - bool : case of second byte - uppercase = public chunk, lowercase = private chunk
  - bool : case of third byte - uppercase (reserved for future use)
  - bool : case of fourth byte - uppercase (reserved for future use)
- (bytes) : data; specific to each chunk
- [u32](#u32) : CRC-32 checksum

### `Header`

[Chunk](#chunk) with identifier `"HHDR"` and data:

- [bool](#bool) : config - store positions
- [Path](#path) : `m_path`

### `End`

[Chunk](#chunk) with identifier `"HEND"` and no data.

### `StringPool`

[Chunk](#chunk) with identifier `"STRI"` and data:

- [arr](#arr) [str](#str) : strings in pool

The type [pstr](#pstr) is a (0-based) index in the array in the last-read StringPool chunk.

### `DocPool`

[Chunk](#chunk) with identifier `"dOCS"` and data:

- [arr](#arr) [str](#str) : docstrings in pool

Docstrings are stored separately from other strings in this chunk.

### `TypeList`

[Chunk](#chunk) with identifier `"TYPL"` and data:

- [arr](#arr) [Path](#path) : external classes
- [arr](#arr) [Path](#path) : external enums
- [arr](#arr) [Path](#path) : external abstracts
- [arr](#arr) [Path](#path) : external typedefs
- [arr](#arr) [ForwardType](#forwardtype) : classes declared in this module
- [arr](#arr) [ForwardType](#forwardtype) : enums declared in this module
- [arr](#arr) [ForwardType](#forwardtype) : abstracts declared in this module
- [arr](#arr) [ForwardType](#forwardtype) : typedefs declared in this module

A class is referenced with a single index, which can refer to both external and internal classes. An index that is within range of the external classes array represents a reference to an external class. Otherwise, the length of the external classes array is subtracted from the index and the result is the index in the internal classes array. Enums, abstracts, and typedefs are indexed in the same way using their respective arrays.

### `FieldList`

[Chunk](#chunk) with identifier `"FLDL"` and data:

- [arr](#arr) [str](#str) ... : an array of field names for each class, internal and external, in the last [TypeList](#typelist) chunk, in the same order
- [arr](#arr) [str](#str) ... : an array of field names for each enum, internal and external, in the last [TypeList](#typelist) chunk, in the same order

### `TypeDeclarations`

[Chunk](#chunk) with identifier `"TYPE"` and data:

- [ClassType](#classtype) ... : class declarations
- [EnumType](#enumtype) ... : enum declarations
- [AbstractType](#abstracttype) ... : abstract declarations
- [DefType](#deftype) ... : typedef declarations

All declarations are in the same order as the corresponding "* declared in this module" field of the last [TypeList](#typelist) chunk.

### `ModuleExtra`

(`module_def_extra` in `type.ml`)

TODO: `m_id` ? ULEB128 for `m_added`, `m_mark`, or `m_processed` ?

[Chunk](#chunk) with identifier `"xTRA"` and data:

- [str](#str) : `m_file`
- (16 bytes) : `m_sign`
- [arr](#arr) : `m_display.m_inline_calls`
  - [pos](#pos)
  - [pos](#pos)
- [arr](#arr) : `m_display.m_type_hints`
  - [pos](#pos)
  - [Type](#type)
- [arr](#arr) [enum](#enum) : `m_check_policy`
  - 0 `NoCheckFileTimeModification`
  - 1 `CheckFileContentModification`
  - 2 `NoCheckDependencies`
  - 3 `NoCheckShadowing`
- [f64](#f64) : `m_time`
- [nullable](#nullable) [Path](#path) : `m_dirty.m_path`
- [i32](#i32) : `m_added`
- [i32](#i32) : `m_mark`
- [arr](#arr) [Path](#path) : `m_deps.map(m -> m.m_path)`
- [i32](#i32) : `m_processed`
- [enum](#enum) : `m_kind`
  - 0 `MCode`
  - 1 `MMacro`
  - 2 `MFake`
  - 3 `MExtern`
  - 4 `MImport`
- [arr](#arr) : `m_binded_res`
  - [str](#str) : resource name
  - [str](#str) : resource data
- [arr](#arr) : `m_if_feature`
  - [str](#str) : feature
  - [ClassRef](#classref)
  - [ClassFieldRef](#classfieldref)
  - [bools](#bools)(1)
    - enabled
- [arr](#arr) : `m_features`
  - [str](#str) : feature
  - [bools](#bools)(1)
    - enabled

---

## Primitives

### `u8`

Unsigned 8-bit integer.

### `u32`

Unsigned 32-bit integer, little-endian.

### `i32`

Signed 32-bit integer, little-endian.

### `f64`

Double-precision IEEE floating-point number.

### `leb128`

Signed [LEB128](https://en.wikipedia.org/wiki/LEB128).

### `uleb128`

Unsigned [LEB128](https://en.wikipedia.org/wiki/LEB128).

### `str`

- [uleb128](#uleb128) : string length
- (bytes) : string data

A `str` may be used to represent arbitrary binary data.

### `arr`

- [uleb128](#uleb128) : array length
- T... : 0 or more entries

### `bool`

- [u8](#u8) : `0` = `false`, `1` = `true`

### `bools`

`floor((n + 7) / 8)` bytes. `n` boolean flags stored in successive bytes. LSB first, little-endian.

### `enum`

- [u8](#u8) : for enum case
- ... : associated data for case, if any

### `nullable`

- [u8](#u8) : present
- T : (if present) data

### `delta`

- [leb128](#leb128) : - signed offset from last decoded delta

The last decoded value is reset to `0` at the beginning of every chunk.

## Haxe

### `pos`

(`pos` in `globals.ml`)

No data when positions not encoded, as specified in the [Header](#header) chunk. Otherwise:

- [delta](#delta) : `pmin`
- [leb128](#leb128) :
  - least significant bit: bool : file present?
  - remaining bits: [delta](#delta) : `pmax`
- (if file present) [pstr](#pstr) : `pfile`

If file is not present, it is the same as the last decoded `pos`. The first `pos` in a chunk must always have a file present.

### `Path`

(`path` in `globals.ml`)

- [arr](#arr) [pstr](#pstr) : package
- [pstr](#pstr) : name

### `pstr`

An index in the array in the last [StringPool](#stringpool) chunk is stored. The first element is referenced as `0`, the second as `1`, etc.

- [uleb128](#uleb128) : string index

### `ClassRef`
### `EnumRef`
### `AbstractRef`
### `DefRef`

- [uleb128](#uleb128) : index in the corresponding [TypeList](#typelist) array

See [TypeList](#typelist) for indexing method.

### `ClassFieldRef`
### `EnumFieldRef`

- [uleb128](#uleb128) : index in the corresponding [FieldList](#fieldlist) array

### `ForwardType`

- [str](#str) : type name
- [pos](#pos) : position
- [pos](#pos) : name position
- [bool](#bool) : private

### `Binop`

(`binop` in `ast.ml`)

Compound assignment binary operators cannot be nested, hence this enum has separate cases for compound assignment operators:

- [enum](#enum)
  - 0 `OpAdd`
  - 1 `OpMult`
  - 2 `OpDiv`
  - 3 `OpSub`
  - 4 `OpAssign`
  - 5 `OpEq`
  - 6 `OpNotEq`
  - 7 `OpGt`
  - 8 `OpGte`
  - 9 `OpLt`
  - 10 `OpLte`
  - 11 `OpAnd`
  - 12 `OpOr`
  - 13 `OpXor`
  - 14 `OpBoolAnd`
  - 15 `OpBoolOr`
  - 16 `OpShl`
  - 17 `OpShr`
  - 18 `OpUShr`
  - 19 `OpMod`
  - 10 `OpInterval`
  - 21 `OpArrow`
  - 22 `OpIn`
  - 40-62 `OpAssignOp(...)`

### `Unop`

(`unop` and `unop_flag` in `ast.ml`)

Unary operators are always referenced with a postfix boolean, hence this enum has separate cases for prefix and postfix operators:

- [enum](#enum)
  - 0 prefix `OpIncrement`
  - 1 prefix `OpDecrement`
  - 2 prefix `OpNot`
  - 3 prefix `OpNeg`
  - 4 prefix `OpNegBits`
  - 40-44 postfix ...

### `Constant`

(`constant` and `string_literal_kind` in `ast.ml`)

- [enum](#enum)
  - 0 `Int(v)`
    - [pstr](#pstr) : `v`
  - 1 `Float(v)`
    - [pstr](#pstr) : `v`
  - 2 `Ident(v)`
    - [pstr](#pstr) : `v`
  - 3 `Regexp(v, opt)`
    - [pstr](#pstr) : `v`
    - [pstr](#pstr) : `opt`
  - 4 `String(v, SDoubleQuotes)`
    - [pstr](#pstr) : `v`
  - 5 `String(v, SSingleQuotes)`
    - [pstr](#pstr) : `v`

### `TypePath`

(`type_path` in `ast.ml`)

- [arr](#arr) [pstr](#pstr) : `tpackage`
- [pstr](#pstr) : `tname`
- [arr](#arr) [TypeParam](#TypeParam) : `tparams`
- [nullable](#nullable) [pstr](#pstr) : `tsub`

### `PlacedTypePath`

(`placed_type_path` in `ast.ml`)

- [TypePath](#typepath) : type path
- [pos](#pos) : position

### `TypeParam`

(`type_param_or_const` in `ast.ml`)

- [enum](#enum)
  - 0 `TPType(t)`
    - [TypeHint](#typehint) : `t`
  - 1 `TPExpr(e)`
    - [Expr](#expr) : `e`

### `ComplexType`

(`complex_type` in `ast.ml`)

- [enum](#enum)
  - 0 `CTPath(p)`
    - [TypePath](#typepath) `p`
  - 1 `CTFunction(args, ret)`
    - [arr](#arr) [TypeHint](#typehint) `args`
    - [TypeHint](#typehint) `ret`
  - 2 `CTAnonymous(fields)`
    - [arr](#arr) [Field](#field) `fields`
  - 3 `CTParent(t)`
    - [TypeHint](#typehint) `t`
  - 4 `CTExtend(p, fields)`
    - [arr](#arr) [PlacedTypePath](#placedtypepath) `p`
    - [arr](#arr) [Field](#field) `fields`
  - 5 `CTOptional(t)`
    - [TypeHint](#typehint) `t`
  - 6 `CTNamed(n, t)`
    - [PlacedName](#placedname) `n`
    - [TypeHint](#typehint) `n`
  - 7 `CTIntersection(tl)`
    - [arr](#arr) [TypeHint](#typehint) `tl`

### `TypeHint`

(`type_hint` in `ast.ml`)

- [ComplexType](#complextype) : complex type
- [pos](#pos) : position

### `Function`

(`func` in `ast.ml`)

- [arr](#arr) [TypeParamDecl](#typeparamdecl) : `f_params`
- [arr](#arr) [FunctionArg](#functionarg) : `f_args`
- [nullable](#nullable) [TypeHint](#typehint) : `f_type`
- [Expr](#expr) : `f_expr`

### `FunctionArg`

(tuple in `func` in `ast.ml`)

- [PlacedName](#placedname) : argument name
- [bool](#bool) : optional
- [arr](#arr) [MetadataEntry](#metadataentry) : metadata
- [nullable](#nullable) [TypeHint](#typehint) : argument type
- [nullable](#nullable) [Expr](#expr) : default value

### `PlacedName`

(`placed_name` in `ast.ml`)

- [pstr](#pstr) : name
- [pos](#pos) : position

### `ExprDef`

(`expr_def`, `while_flag`, `function_kind`, and `display_kind` in `ast.ml`)

- [enum](#enum)
  - 0 `EConst(c)`
    - [Constant](#constant) : `c`
  - 1 `EArray(e1, e2)`
    - [Expr](#expr) : `e1`
    - [Expr](#expr) : `e2`
  - 2 `EBinop(op, e1, e2)`
    - [Binop](#binop) : `op`
    - [Expr](#expr) : `e1`
    - [Expr](#expr) : `e2`
  - 3 `EField(e, field)`
    - [Expr](#expr) : `e`
    - [pstr](#pstr) : `field`
  - 4 `EParenthesis(e)`
    - [Expr](#expr) : `e`
  - 5 `EObjectDecl(fields)`
    - [arr](#arr) [ObjectField](#objectfield) : `fields`
  - 6 `EArrayDecl(values)`
    - [arr](#arr) [Expr](#expr) : `values`
  - 7 `ECall(e, params)`
    - [Expr](#expr) : `e`
    - [arr](#arr) [Expr](#expr) : `params`
  - 8 `ENew(t, params)`
    - [PlacedTypePath](#placedtypepath) : `t`
    - [arr](#arr) [Expr](#expr) : `params`
  - 9 `EUnop(op, postfix, e)`
    - [Unop](#unop) : `op`, `postfix`
    - [Expr](#expr) : `e`
  - 10 `EVars(vars)`
    - [arr](#arr) [Var](#var) : `vars`
  - 11 `EFunction(FKAnonymous, f)`,
  - 12 `EFunction(FKNamed(name, inlined), f)`,
  - 13 `EFunction(FKArrow, f)`
    - [Function](#function) : `f`
    - (if 12) [PlacedName](#PlacedName) : `name`
    - (if 12) [bool](#bool) : `inlined`
  - 14 `EBlock(exprs)`
    - [arr](#arr) [Expr](#expr) : `exprs`
  - 15 `EFor(it, expr)`
    - [Expr](#expr) : `it`
    - [Expr](#expr) : `expr`
  - 16 `EIf(econd, eif, null)` (no else),
  - 17 `EIf(econd, eif, eelse)` (with else)
    - [Expr](#expr) : `econd`
    - [Expr](#expr) : `eif`
    - (if 17) [Expr](#expr) : `eelse`
  - 18 `EWhile(econd, e, NormalWhile)`,
  - 19 `EWhile(econd, e, DoWhile)`
    - [Expr](#expr) : `econd`
    - [Expr](#expr) : `e`
  - 20 `ESwitch(e, cases, null)` (no default),
  - 21 `ESwitch(e, cases, pos)` (empty default with position),
  - 22 `ESwitch(e, cases, edef)` (default case)
    - [Expr](#expr) : `e`
    - [arr](#arr) [Case](#case) : `cases`
    - (if 21) [pos](#pos) : `pos`
    - (if 22) [Expr](#expr) : `edef`
  - 23 `ETry(e, [catch])` (single catch)
    - [Expr](#expr) : `e`
    - [Catch](#catch) : `catch`
  - 24 `ETry(e, catches)` (zero or multiple catches)
    - [Expr](#expr) : `e`
    - [arr](#arr) [Catch](#catch) : `catches`
  - 25 `EReturn(null)` (void return)
  - 26 `EReturn(e)` (expression return)
    - [Expr](#expr) : `e`
  - 27 `EBreak`
  - 28 `EContinue`
  - 29 `EUntyped(e)`
    - [Expr](#expr) : `e`
  - 30 `EThrow(e)`
    - [Expr](#expr) : `e`
  - 31 `ECast(e, null)` (unsafe cast),
  - 32 `ECast(e, t)` (safe cast)
    - [Expr](#expr) : `e`
    - (if 32) [TypeHint](#typehint) : `t`
  - 33 `EDisplay(e, DKCall)`,
  - 34 `EDisplay(e, DKDot)`,
  - 35 `EDisplay(e, DKStructure)`,
  - 36 `EDisplay(e, DKMarked)`,
  - 37 `EDisplay(e, DKPatern(false))`,
  - 38 `EDisplay(e, DKPatern(true))`
    - [Expr](#expr) : `e`
  - 39 `EDisplayNew(t)`
    - [PlacedTypePath](#placedtypepath) : `t`
  - 40 `ETernary(econd, eif, eelse)`
    - [Expr](#expr) : `econd`
    - [Expr](#expr) : `eif`
    - [Expr](#expr) : `eelse`
  - 41 `ECheckType(e, t)`
    - [Expr](#expr) : `e`
    - [TypeHint](#typehint) : `t`
  - 42 `EMeta(s, e)`
    - [MetadataEntry](#metadataentry) : `s`
    - [Expr](#expr) : `e`

### `ObjectField`

(tuple in `expr_def.EObjectDecl` in `ast.ml`)

- [pstr](#pstr) : field name
- [pos](#pos) : position of name
- [bool](#bool) : quoted
- [Expr](#expr) : expression

### `Var`

(tuple in `expr_def.EVars` in `ast.ml`)

- [PlacedName](#placedname) : variable name
- [bool](#bool) : final
- [nullable](#nullable) [TypeHint](#typehint) : variable type
- [nullable](#nullable) [Expr](#expr) : expression

### `Case`

(tuple in `expr_def.ESwitch` in `ast.ml`)

- [arr](#arr) [Expr](#expr) : matched values
- [nullable](#nullable) [Expr](#expr) : guard
- [nullable](#nullable) [Expr](#expr) : expression
- [pos](#pos) : position

### `Catch`

(tuple in `expr_def.ETry` in `ast.ml`)

- [PlacedName](#placedname) : exception variable name
- [TypeHint](#typehint) : exception type
- [Expr](#expr) : catch expression
- [pos](#pos) : position of variable

### `Expr`

(`expr` in `ast.ml`)

- [ExprDef](#ExprDef) : expression kind
- [pos](#pos) : position

### `TypeParamDecl`

(`type_param` in `ast.ml`)

TODO: recursion?

- [PlacedName](#placedname) : `tp_name`
- [arr](#arr) [TypeParamDecl](#typeparamdecl) : `tp_params`
- [nullable](#nullable) [TypeHint](#typehint) : `tp_constraints`
- [arr](#arr) [MetadataEntry](#metadataentry) : `tp_meta`

### `doc`

(`documentation` in `ast.ml`)

Any occurence of `doc` may be `null` (when no docstring is present) - absence is indicated with a value of `0`. Otherwise, an index in the array in the last-read [DocPool](#docpool) chunk is stored, offset by one, i.e. the first element is referenced as `1`, the second as `2`, etc.

- [uleb128](#uleb128) : offset doc index or `0`

If a DocPool chunk is not present, documentation is omitted in the `hxb` file. Decoded docstrings should become `null`.

### `MetadataEntry`

(`metadata_entry` in `ast.ml`)

TODO: index well-known metas in an enum?

- [pstr](#pstr) : name
- [arr](#arr) [Expr](#expr) : arguments
- [pos](#pos) : position

### `PlacedAccess`

(`placed_access` and `access` in `ast.ml`)

- [enum](#enum) : access modifier
  - 0 `APublic`
  - 1 `APrivate`
  - 2 `AStatic`
  - 3 `AOverride`
  - 4 `ADynamic`
  - 5 `AInline`
  - 6 `AMacro`
  - 7 `AFinal`
  - 8 `AExtern`
- [pos](#pos) : position

### `Field`

(`class_field` and `class_field_kind` in `ast.ml`)

- [PlacedName](#placedname) : `cff_name`
- [doc](#doc) : `cff_doc`
- [pos](#pos) : `cff_pos`
- [arr](#arr) [MetadataEntry](#metadataentry) : `cff_meta`
- [arr](#arr) [PlacedAccess](#placedaccess) : `cff_access`
- [enum](#enum) : `cff_kind`
  - 0 `FVar(t, e)`
    - [nullable](#nullable) [TypeHint](#typehint) : `t`
    - [nullable](#nullable) [Expr](#expr) : `e`
  - 1 `FFun(f)`
    - [Function](#function) : `f`
  - 2 `FProp(get, get, t, e)`,
  - 3 `FProp(get, set, t, e)`,
  - 4 `FProp(get, null, t, e)`,
  - 5 `FProp(get, default, t, e)`,
  - 6 `FProp(get, never, t, e)`,
  - 7 `FProp(set, get, t, e)`,
  - 8 `FProp(set, set, t, e)`,
  - 9 `FProp(set, null, t, e)`,
  - 10 `FProp(set, default, t, e)`,
  - 11 `FProp(set, never, t, e)`,
  - 12 `FProp(null, get, t, e)`,
  - 13 `FProp(null, set, t, e)`,
  - 14 `FProp(null, null, t, e)`,
  - 15 `FProp(null, default, t, e)`,
  - 16 `FProp(null, never, t, e)`,
  - 17 `FProp(default, get, t, e)`,
  - 18 `FProp(default, set, t, e)`,
  - 19 `FProp(default, null, t, e)`,
  - 20 `FProp(default, default, t, e)`,
  - 21 `FProp(default, never, t, e)`,
  - 22 `FProp(never, get, t, e)`,
  - 23 `FProp(never, set, t, e)`,
  - 24 `FProp(never, null, t, e)`,
  - 25 `FProp(never, default, t, e)`,
  - 26 `FProp(never, never, t, e)`
    - [pos](#pos) : position of read access specifier
    - [pos](#pos) : position of write access specifier
    - [nullable](#nullable) [TypeHint](#typehint) : `t`
    - [nullable](#nullable) [Expr](#expr) : `e`

### `Type`

(`t` and `tsignature` in `type.ml`)

In case of `TLazy`, resolve, then encode result.

- [enum](#enum)
  - 0 `TMono(null)` (unbound monomorph)
  - 1 `TMono(t)` (bound monomorph)
    - [Type](#type) : `t`
  - 2 `TEnum(t, [])` (no params),
  - 3 `TEnum(t, params)` (one or more params)
    - [EnumRef](#enumref) : `t`
    - (if 3) [arr](#arr) [Type](#type) : `params`
  - 4 `TInst(t, [])` (no params),
  - 5 `TInst(t, params)` (one or more params)
    - [ClassRef](#classref) : `t`
    - (if 5) [arr](#arr) [Type](#type) : `params`
  - 6 `TType(t, [])` (no params),
  - 7 `TType(t, params)` (one or more params)
    - [DefRef](#defref) : `t`
    - (if 7) [arr](#arr) [Type](#type) : `params`
  - 8 `TFun(args, ret)`
    - [arr](#arr) [TFunArg](#tfunarg) : `args`
    - [Type](#type) : `ret`
  - 9 `TAnon(anon)`
    - [AnonType](#anontype) : `anon`
  - 10 `TDynamic(...)` (recursive `TDynamic`)
  - 11 `TDynamic(t)`
    - [Type](#type) : `type`
  - 12 `TAbstract(t, [])` (no params),
  - 13 `TAbstract(t, params)` (one or more params)
    - [AbstractRef](#abstractref) : `t`
    - (if 13) [arr](#arr) [Type](#type) : `params`

### `TFunArg`

(tuple in `tsignature` in `type.ml`)

- [pstr](#pstr) : argument name
- [bool](#bool) : optional
- [Type](#type) : type

### `TypeParams`

(`type_params` in `type.ml`)

- [arr](#arr)
  - [pstr](#pstr)
  - [Type](#type)

### `TConstant`

(`tconstant` in `type.ml`)

- [enum](#enum)
  - 0 `TInt(i)`
    - [leb128](#leb128) : `i`
  - 1 `TFloat(s)`
    - [pstr](#pstr) : `s`
  - 2 `TString(s)`
    - [pstr](#pstr) : `s`
  - 3 `TBool(false)`
  - 4 `TBool(true)`
  - 5 `TNull`
  - 6 `TThis`
  - 7 `TSuper`

### `TVarExtra`

(`tvar_extra` in `type.ml`)

- [TypeParams](#typeparams)
- [nullable](#nullable) [TypedExpr](#typedexpr)

### `TVar`

(`tvar`, `tvar_origin`, and `tvar_kind` in `type.ml`)

- [leb128](#leb128) : `v_id`
- [pstr](#pstr) : `v_name`
- [Type](#type) : `v_type`
- [enum](#enum) : `v_kind`
  - 0 `VGenerated`
  - 1 `VInlined`
  - 2 `VInlinedConstructorVariable`
  - 3 `VExtractorVariable`
  - 4 `VUser(TVOLocalVariable)`
  - 5 `VUser(TVOArgument)`
  - 6 `VUser(TVOForVariable)`
  - 7 `VUser(TVOPatternVariable)`
  - 8 `VUser(TVOCatchVariable)`
  - 9 `VUser(TVOLocalFunction)`
- [bools](#bools)(2)
  - `v_capture`
  - `v_final`
- [nullable](#nullable) [TVarExtra](#tvarextra) : `v_extra`
- [arr](#arr) [MetadataEntry](#metadataentry) : `v_meta`
- [pos](#pos) : `v_pos`

### `TFunc`

(`tfunc` in `type.ml`)

- [arr](#arr) [TFuncArg](#tfuncarg) : `tf_args`
- [Type](#type) : `tf_type`
- [TypedExpr](#typedexpr) : `tf_expr`

### `TFuncArg`

(tuple in `tfunc` in `type.ml`)

- [TVar](#tvar) : argument variable
- [nullable](#nullable) [TypedExpr](#typedexpr) : default value

### `AnonType`

(`tanon` and `anon_status` in `type.ml`)

TODO: references, fields?

- ... `a_fields`
- [enum](#enum) : `a_status`
  - 0 `Closed`
  - 1 `Opened`
  - 2 `Const`
  - 3 `Extend(tl)`
    - [Type](#type) : `tl`
  - 4 `Statics(t)`
    - [TClass](#tclass) : `t`
  - 5 `EnumStatics(t)`
    - [TEnum](#tenum) : `t`
  - 6 `AbstractStatics(t)`
    - [TAbstract](#tabstract) : `t`

### `TypedExprDef`

(`texpr_expr` and `tfield_access` in `type.ml`)

- [enum](#enum)
  - 0 `TConst(c)`
    - [TConstant](#tconstant) : `c`
  - 1 `TLocal(v)`
    - [TVarRef](#tvarref) : `v`
  - 2 `TArray(e1, e2)`
    - [TypedExpr](#typedexpr) : `e1`
    - [TypedExpr](#typedexpr) : `e2`
  - 3 `TBinop(op, e1, e2)`
    - [Binop](#binop) : `op`
    - [TypedExpr](#typedexpr) : `e1`
    - [TypedExpr](#typedexpr) : `e2`
  - 4 `TField(e, FInstance(c, params, cf))`
    - [TypedExpr](#typedexpr) : `e`
    - [ClassRef](#classref) : `c`
    - [arr](#arr) [Type](#type) : `params`
    - [ClassFieldRef](#classfieldref) : `cf`
  - 5 `TField(e, FStatic(c, cf))`
    - [TypedExpr](#typedexpr) : `e`
    - [ClassRef](#classref) : `c`
    - [ClassFieldRef](#classfieldref) : `cf`
  - 6 `TField(e, FAnon(cf))`
    - [TypedExpr](#typedexpr) : `e`
    - ... TODO: reference field?
  - 7 `TField(e, FDynamic(s))`
    - [TypedExpr](#typedexpr) : `e`
    - [pstr](#pstr) : `s`
  - 8 `TField(e, FClosure(null, null, cf))` (no class, i.e. TAnon)
    - [TypedExpr](#typedexpr) : `e`
    - ... TODO: reference field?
  - 9 `TField(e, FClosure(c, params, cf))`
    - [TypedExpr](#typedexpr) : `e`
    - [ClassRef](#classref) : `c`
    - [arr](#arr) [Type](#type) : `params`
    - [ClassFieldRef](#classfieldref) : `cf`
  - 10 `TField(e, FEnum(e, ef))`
    - [TypedExpr](#typedexpr) : `e`
    - [EnumRef](#enumref) : `e`
    - [EnumFieldRef](#enumfieldref) : `ef`
  - 11 `TTypeExpr(m)` (class)
    - [ClassRef](#classref) : `m`
  - 12 `TTypeExpr(m)` (enum)
    - [EnumRef](#enumref) : `m`
  - 13 `TTypeExpr(m)` (typedef)
    - [DefRef](#defref) : `m`
  - 14 `TTypeExpr(m)` (abstract)
    - [AbstractRef](#abstractref) : `m`
  - 15 `TParenthesis(e)`
    - [TypedExpr](#typedexpr) : `e`
  - 16 `TObjectDecl(fields)`
    - [arr](#arr) [TObjectField](#tobjectfield) : `fields`
  - 17 `TArrayDecl(el)`
    - [arr](#arr) [TypedExpr](#typedexpr) : `el`
  - 18 `TCall(e, el)`
    - [TypedExpr](#typedexpr) : `e`
    - [arr](#arr) [TypedExpr](#typedexpr) : `el`
  - 19 `TNew(c, params, el)`
    - [ClassRef](#classref) : `c`
    - [arr](#arr) [Type](#type) : `params`
    - [arr](#arr) [TypedExpr](#typedexpr) : `el`
  - 20 `TUnop(op, postfix, e)`
    - [Unop](#unop) : `op`, `postfix`
    - [TypedExpr](#typedexpr) : `e`
  - 21 `TFunction(tfunc)`
    - [TFunc](#tfunc) : `tfunc`
  - 22 `TVar(v, null)` (no expr),
  - 23 `TVar(v, expr)` (with expr)
    - [TVar](#tvar) : `v`
    - (if 23) [TypedExpr](#typedexpr) : `expr`
  - 24 `TBlock(el)`
    - [arr](#arr) [TypedExpr](#typedexpr) : `el`
  - 25 `TFor(v, e1, e2)`
    - [TVar](#tvar) : `v`
    - [TypedExpr](#typedexpr) : `e1`
    - [TypedExpr](#typedexpr) : `e2`
  - 26 `TIf(econd, eif, null)` (no else),
  - 27 `TIf(econd, eif, eelse)` (with else)
    - [TypedExpr](#typedexpr) : `econd`
    - [TypedExpr](#typedexpr) : `eif`
    - (if 27) [TypedExpr](#typedexpr) : `eelse`
  - 28 `TWhile(econd, e, NormalWhile)`,
  - 29 `TWhile(econd, e, DoWhile)`
    - [TypedExpr](#typedexpr) : `econd`
    - [TypedExpr](#typedexpr) : `e`
  - 30 `TSwitch(e, cases, null)` (no default),
  - 31 `TSwitch(e, cases, edef)` (default case)
    - [TypedExpr](#typedexpr) : `e`
    - [arr](#arr) [TCase](#tcase) : `cases`
    - (if 31) [TypedExpr](#typedexpr) : `edef`
  - 32 `TTry(e, [catch])` (one catch)
    - [TypedExpr](#typedexpr) : `e`
    - [TCatch](#tcatch) : `catch`
  - 33 `TTry(e, catches)` (zero or multiple catches)
    - [TypedExpr](#typedexpr) : `e`
    - [arr](#arr) [TCatch](#tcatch) : `catches`
  - 34 `TReturn(null)` (void return)
  - 35 `TReturn(e)` (expression return)
    - [TypedExpr](#typedexpr) : `e`
  - 36 `TBreak`
  - 37 `TContinue`
  - 38 `TThrow(e)`
    - [TypedExpr](#typedexpr) : `e`
  - 39 `TCast(e, null)` (unsafe cast),
  - 40 `TCast(e, m)` (safe cast to class),
  - 41 `TCast(e, m)` (safe cast to enum),
  - 42 `TCast(e, m)` (safe cast to typedef),
  - 43 `TCast(e, m)` (safe cast to abstract)
    - [TypedExpr](#typedexpr) : `e`
    - (if 40) [ClassRef](#classref) : `m`
    - (if 41) [EnumRef](#enumref) : `m`
    - (if 42) [DefRef](#defref) : `m`
    - (if 43) [AbstractRef](#abstractref) : `m`
  - 44 `TMeta(m, e)`
    - [MetadataEntry](#metadataentry) : `m`
    - [TypedExpr](#typedexpr) : `e`
  - 45 `TEnumParameter(e, ef, index)`
    - [TypedExpr](#typedexpr) : `e`
    - [EnumFieldRef](#enumfieldref) : `ef`
    - [uleb128](#uleb128) : `index`
  - 46 `TEnumIndex(e)`
    - [TypedExpr](#typedexpr) : `e`
  - 47 `TIdent(s)`
    - [pstr](#pstr) : `s`

### `TObjectField`

(tuple in `texpr_expr.TObjectDecl` in `type.ml`)

- [pstr](#pstr) : field name
- [pos](#pos) : position of name
- [bool](#bool) : quoted
- [TypedExpr](#typedexpr) : expression

### `TCase`

(tuple in `texpr_expr.TSwitch` in `type.ml`)

- [arr](#arr) [TypedExpr](#typedexpr) : matched values
- [TypedExpr](#typedexpr) : expression

### `TCatch`

(tuple in `texpr_expr.TTry` in `type.ml`)

- [TVar](#tvar) : exception variable
- [TypedExpr](#typedexpr) : catch expression

### `TypedExpr`

(`texpr` in `type.ml`)

- [TypedExprDef](#typedexprdef) : `eexpr`
- [Type](#type) : `etype`
- [pos](#pos) : `epos`

### `BaseType`

(`tinfos` in `type.ml`)

- [doc](#doc) : `mt_doc`
- [arr](#arr) [MetadataEntry](#metadataentry) : `mt_meta`
- [TypeParams](#typeparams) : `mt_params`
- [arr](#arr) [ClassUsing](#classusing) : `mt_using`

Fields which are not serialised:

- `mt_module` - belongs to the module defined in the current [TypeDeclarations](#typedeclarations) chunk
- `mt_path`, `mt_pos`, `mt_name_pos`, `mt_private` - serialised in the [TypeDeclarations](#typedeclarations) chunk

### `ClassUsing`

(tuple in `tinfos` in `type.ml`)

- [ClassRef](#classref)
- [pos](#pos)

### `ClassField`

(`tclass_field` in `type.ml`)

- [pstr](#pstr) : `cf_name`
- [Type](#type) : `cf_type`
- [pos](#pos) : `cf_pos`
- [pos](#pos) : `cf_name_pos`
- [doc](#doc) : `cf_doc`
- [arr](#arr) [MetadataEntry](#metadataentry) : `cf_meta`
- [enum](#enum) : `cf_kind`
  - 0 `Method(MethNormal)`
  - 1 `Method(MethInline)`
  - 2 `Method(MethDynamic)`
  - 3 `Method(MethMacro)`
  - 10-234 `Var(r, w)` - index = 10 + r * 15 + w, where r or w is:
    - 0 `AccNormal`
    - 1 `AccNo`
    - 2 `AccNever`
    - 3 `AccCtor`
    - 4 `AccResolve`
    - 5 `AccCall`
    - 6 `AccInline`
    - 7 `AccRequire(r, msg)`
    - (for r, w that are `AccRequire`)
      - [pstr](#pstr) : `r`
      - [nullable](#nullable) [pstr](#pstr) : `msg`
- [TypeParams](#typeparams) : `cf_params`
- [nullable](#nullable) [TypedExpr](#typedexpr) : `cf_expr`
- [nullable](#nullable) [TFunc](#tfunc) : `cf_expr_unoptimized`
- [arr](#arr) [ClassFieldRef](#classfieldref) : `cf_overloads`
- [bools](#bools)(5) : `cf_flags`
  - `CfPublic`
  - `CfExtern`
  - `CfFinal`
  - `CfOverridden`
  - `CfModifiesThis`

### `ClassType`

(`tclass` and `tclass_kind` in `type.ml`)

- [BaseType](#basetype) : (infos)
- [enum](#enum) : `cl_kind`
  - 0 `KNormal`
  - 1 `KTypeParameter(constraints)`
    - [arr](#arr) [Type](#type) : `constraints`
  - 2 `KExpr(e)`
    - [Expr](#expr) : `e`
  - 3 `KGeneric`
  - 4 `KGenericInstance(cl, params)`
    - [ParamClassType](#paramclasstype) : `cl`, `params`
  - 5 `KMacroType`
  - 6 `KGenericBuild(fields)`
    - [arr](#arr) [Field](#field) : `fields`
  - 7 `KAbstractImpl(a)`
    - [AbstractRef](#abstractref) : `a`
- [bools](#bools)(3)
  - `cl_extern`
  - `cl_final`
  - `cl_interface`
- [nullable](#nullable) [ParamClassType](#paramclasstype) : `cl_super`
- [arr](#arr) [ParamClassType](#paramclasstype) : `cl_implements`
- [arr](#arr) [ClassField](#classfield) : `cl_ordered_fields`
- [arr](#arr) [ClassField](#classfield) : `cl_ordered_statics`
- [nullable](#nullable) [Type](#type) : `cl_dynamic`
- [nullable](#nullable) [Type](#type) : `cl_array_access`
- [nullable](#nullable) [ClassFieldRef](#classfieldref) : `cl_constructor`
- [nullable](#nullable) [TypedExpr](#typedexpr) : `cl_init`
- [arr](#arr) [ClassFieldRef](#classfieldref) : `cl_overrides`

Fields which are not serialised:

- `cl_statics`, `cl_fields` - can be deserialised from the ordered lists
- `cl_build`, `cl_restore` - functions
- `cl_descendants` - populated automatically in postprocessing

### `ParamClassType`

(`(tclass * tparams)` in `type.ml`)

- [ClassRef](#classref)
- [arr](#arr) [Type](#type)

### `EnumField`

(`tenum_field` in `type.ml`)

- [pstr](#pstr) : `ef_name`
- [Type](#type) : `ef_type`
- [pos](#pos) : `ef_pos`
- [pos](#pos) : `ef_name_pos`
- [doc](#doc) : `ef_doc`
- [TypeParams](#typeparams) : `ef_params`
- [arr](#arr) [MetadataEntry](#metadataentry) : `ef_meta`

Fields which are not serialised:

- `ef_index` - enum constructors are declared in order, first `ef_index` is `0`

### `EnumType`

(`tenum` in `type.ml`)

- [BaseType](#basetype) : (infos)
- [DefType](#deftype) : `e_type`
- [bools](#bools)(1)
  - `e_extern`
- [arr](#arr) [EnumField](#enumfield) : values of `e_constrs`, in the order of `e_names`

Fields which are not serialised:

- `e_names` - [EnumField](#enumfield) stores the constructor name

### `DefType`

(`tdef` in `type.ml`)

- [BaseType](#basetype) : (infos)
- [Type](#type) : `t_type`

### `AbstractType`

(`tabstract` in `type.ml`)

- [BaseType](#basetype) : (infos)
- [arr](#arr) [AbstractBinop](#abstractbinop) : `a_ops`
- [arr](#arr) [AbstractUnop](#abstractunop) : `a_unops`
- [nullable](#nullable) [ClassRef](#classref) : `a_impl`
- [Type](#type) : `a_this`
- [arr](#arr) [Type](#type) : `a_from`
- [arr](#arr) [AbstractFromTo](#abstractfromto) : `a_from_field`
- [arr](#arr) [Type](#type) : `a_to`
- [arr](#arr) [AbstractFromTo](#abstractfromto) : `a_to_field`
- [arr](#arr) [ClassFieldRef](#classfieldref) : `a_array`
- [nullable](#nullable) [ClassFieldRef](#classfieldref) : `a_read`
- [nullable](#nullable) [ClassFieldRef](#classfieldref) : `a_write`

### `AbstractBinop`

(tuple in `tabstract` in `type.ml`)

- [Binop](#binop)
- [ClassFieldRef](#classfieldref)

### `AbstractUnop`

(tuple in `tabstract` in `type.ml`)

- [Unop](#unop) : operator and prefix/postfix flag
- [ClassFieldRef](#classfieldref)

### `AbstractFromTo`

(tuple in `tabstract` in `type.ml`)

- [Type](#type)
- [ClassFieldRef](#classfieldref)
