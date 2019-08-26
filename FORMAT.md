# `hxb` format

## Primitives

### `u8`

Unsigned 8-bit integer.

### `leb128`

Signed [LEB128](https://en.wikipedia.org/wiki/LEB128).

### `uleb128`

Unsigned [LEB128](https://en.wikipedia.org/wiki/LEB128).

### `str`

- uleb128 string length
- ... string data

### `arr T`

- uleb128 array length
- ... 0 or more T

### `bools(n)`

`floor((n + 7) / 8)` bytes. `n` boolean flags stored in successive bytes. LSB first, little-endian.

### `enum`

- `u8` for enum case
- ... associated data for case, if any

### `nullable T`

- i8 present
- (if present) T

### `cache T`

- uleb128 - `0` when not cached, `1+` stored index
- (if not cached) T

### `delta`

- leb128 - signed offset from last decoded delta

### `pos(p)`

No data when positions not encoded. Otherwise:

- cache str p.file
- delta p.min
- delta p.max

### `doc`

No data when docs not encoded. Otherwise:

- nullable str doc

## Haxe (Expr)

### `Constant`

- enum
  - 0 CInt(v)
    - str v
  - 1 CFloat(v)
    - str v
  - 2 CIdent(v)
    - cache str v
  - 3 CRegexp(v, opt)
    - str v
    - str opt
  - 4 CString(v, DoubleQuotes)
    - str v
  - 5 CString(v, SingleQuotes)
    - str v

### `Binop`

- enum
  - 0 OpAdd
  - 1 OpMult
  - 2 OpDiv
  - 3 OpSub
  - 4 OpAssign
  - 5 OpEq
  - 6 OpNotEq
  - 7 OpGt
  - 8 OpGte
  - 9 OpLt
  - 10 OpLte
  - 11 OpAnd
  - 12 OpOr
  - 13 OpXor
  - 14 OpBoolAnd
  - 15 OpBoolOr
  - 16 OpShl
  - 17 OpShr
  - 18 OpUShr
  - 19 OpMod
  - 10 OpInterval
  - 21 OpArrow
  - 22 OpIn
  - 40-62 OpAssignOp(...)

### `Unop(op, postfix)`

- enum op
  - 0 prefix OpIncrement
  - 1 prefix OpDecrement
  - 2 prefix OpNot
  - 3 prefix OpNeg
  - 4 prefix OpNegBits
  - 40-44 postfix ...

### `Expr(e)`

- enum ExprDef e.expr
- pos e.pos

### `ExprDef`

- enum
  - 0 EConst(c)
    - Constant c
  - 1 EArray(e1, e2)
    - Expr e1
    - Expr e2
  - 2 EBinop(op, e1, e2)
    - Binop op
    - Expr e1
    - Expr e2
  - 3 EField(e, field)
    - Expr e
    - str field
  - 4 EParenthesis(e)
    - Expr e
  - 5 EObjectDecl(fields)
    - arr ObjectField fields
  - 6 EArrayDecl(values)
    - arr Expr values
  - 7 ECall(e, params)
    - Expr e
    - arr Expr params
  - 8 ENew(t:TypePath, params)
    - TypePath t
    - arr Expr params
  - 9 EUnop(op, postFix, e)
    - Unop op, postFix
    - Expr e
  - 10 EVars(vars)
    - arr Var vars
  - 11 EFunction(FAnonymous, f)
  - 12 EFunction(FNamed, f)
  - 13 EFunction(FArrow, f)
    - Function f
    - (if 12) str name
    - (if 12) bools(1)
      - inlined
  - 14 EBlock(exprs)
    - arr Expr exprs
  - 15 EFor(it, expr)
    - Expr it
    - Expr expr
  - 16 EIf(econd, eif, null) (no else)
  - 17 EIf(econd, eif, eelse) (with else)
    - Expr econd
    - Expr eif
    - (if 17) Expr eelse
  - 18 EWhile(econd, e, true) (normal)
  - 19 EWhile(econd, e, false) (do-while)
    - Expr econd
    - Expr e
  - 20 ESwitch(e, cases, null) (no default)
  - 21 ESwitch(e, cases, edef) (default case)
    - Expr e
    - arr Case cases
    - (if 21) Expr edef
  - 22 ETry(e, [catch]) (single catch)
    - Expr e
    - Catch catch
  - 23 ETry(e, catches) (zero or multiple catches)
    - Expr e
    - arr Catch catches
  - 24 EReturn(null) (void return)
  - 25 EReturn(e) (value return)
    - Expr e
  - 26 EBreak
  - 27 EContinue
  - 28 EUntyped(e)
    - Expr e
  - 29 EThrow(e)
    - Expr e
  - 30 ECast(e, null) (unsafe cast)
  - 31 ECast(e, t) (safe cast)
    - Expr e
    - (if 31) ComplexType t
  - 32 EDisplay(e, displayKind)
    - Expr e
    - DisplayKind displayKind
  - 33 EDisplayNew(t)
    - TypePath t
  - 34 ETernary(econd, eif, eelse)
    - Expr econd
    - Expr eif
    - Expr eelse
  - 35 ECheckType(e, t)
    - Expr e
    - TypePath t
  - 36 EMeta(s, e)
    - MetadataEntry s
    - Expr e

### `Case(c)`

- arr Expr c.values
- nullable Expr c.guard
- nullable Expr c.expr

### `Var(v)`

- cache str v.name
- nullable ComplexType v.type
- nullable Expr v.expr
- bools(1)
  - v.isFinal

### `Catch(c)`

- cache str c.name
- ComplexType c.type
- Expr c.expr

### `ObjectField(f)`

- cache str f.name
- Expr f.expr
- bools(1)
  - f.quotes == Quoted

### `DisplayKind`

- enum
  - 0 DKCall
  - 1 DKDot
  - 2 DKStructure
  - 3 DKMarked
  - 4 DKPatern(false)
  - 5 DKPatern(true)

### `ComplexType`

- enum
  - 0 TPath(p)
    - TypePath p
  - 1 TFunction(args, ret)
    - arr ComplexType args.concat(ret)
  - 2 TAnonymous(fields)
    - arr Field fields
  - 3 TParent(t)
    - ComplexType t
  - 4 TExtend(p, fields)
    - arr TypePath p
    - arr Field fields
  - 5 TOptional(t)
    - TypePath t
  - 6 TNamed(n, t)
    - str n
    - TypePath t
  - 7 TIntersection(tl)
    - arr ComplexType tl

### `TypePath(p)`

- str `(p.pack.concat([p.name]).concat(p.sub != null ? [p.sub] : [])).join(".")`
- arr TypeParam p.params

### `TypeParam`

- enum
  - 0 TPType(t)
    - ComplexType t
  - 1 TPExpr(e)
    - Expr e

### `TypeParamDecl(t)`

- cache str t.name
- nullable arr ComplexType t.constraints
- nullable arr cache TypeParamDecl t.params
- nullable arr MetadataEntry t.meta

### `Function(f)`

- arr FunctionArg f.args
- nullable ComplexType f.ret
- nullable Expr f.expr
- nullable arr cache TypeParamDecl f.params

### `FunctionArg(a)`

- cache str a.name
- bools(1)
  - a.opt
- nullable ComplexType a.type
- nullable Expr a.value
- nullable arr MetadataEntry a.meta

### `MetadataEntry(e)`

- str e.name
- nullable arr Expr e.params
- pos e.pos

### `Field(f)`

- cache str f.name
- doc f.doc
- arr Access f.access
- enum f.kind
  - 0 FVar(t, e)
    - nullable ComplexType t
    - nullable Expr e
  - 1 FFun(f)
    - Function f
  - 2 FProp(get, set, t, e)
    - PropAccess get
    - PropAccess set
    - nullable ComplexType t
    - nullable Expr e
- pos f.pos
- nullable arr MetadataEntry f.meta

### `PropAccess`

- enum
  - 0 get
  - 1 set
  - 2 null
  - 3 default
  - 4 never

### `Access`

TODO: bitfield?

- enum
  - 0 APublic
  - 1 APrivate
  - 2 AStatic
  - 3 AOverride
  - 4 ADynamic
  - 5 AInline
  - 6 AMacro
  - 7 AFinal
  - 8 AExtern

### `FieldType`

## Haxe (Type)

### `Type`

- enum
  - 0 TMono(t)
    - cache Type t
  - 1 TEnum(t, params)
    - cache EnumType t
    - arr cache Type params
  - 2 TInst(t, params)
    - cache ClassType t
    - arr cache Type params
  - 3 TType(t, params)
    - cache DefType t
    - arr cache Type params
  - 4 TFun(args, ret)
    - arr FunctionArg args
    - cache Type ret
  - 5 TAnonymous(anon)
    - cache AnonType anon
  - 6 TDynamic(type)
    - nullable cache Type type
  - 7 TLazy
    - resolve, then encode
  - 8 TAbstract(t, params)
    - cache AbstractType t
    - arr cache Type params

### `AnonType(t)`

- arr ClassField t.fields
- enum t.status
  - 0 AClosed
  - 1 AOpened
  - 2 AConst
  - 3 AExtend(tl)
    - arr cache Type tl
  - 4 AClassStatics(t)
    - cache ClassType t
  - 5 AEnumStatics(t)
    - cache EnumType t
  - 6 AAbstractStatics(t)
    - cache AbstractType t

### `TypeParameter(p)`

- cache str p.name
- cache Type p.type

### `ClassField(f)`

- cache str f.name
- cache Type f.type
- bools(3)
  - f.isPublic
  - f.isExtern
  - f.isFinal
- arr TypeParameter f.params
- arr MetadataEntry f.meta
- enum f.kind
  - 0 FMethod(MethNormal)
  - 1 FMethod(Inline)
  - 2 FMethod(Dynamic)
  - 3 FMethod(Macro)
  - 10-234 FVar(r, w) - index = 10 + r * 15 + w, where r or w is:
    - 0 AccNormal
    - 1 AccNo
    - 2 AccNever
    - 3 AccResolve
    - 4 AccCall
    - 5 AccInline
    - 6 AccRequire(r, msg)
    - 7 AccCtor
    - (for r, w that are AccRequire)
      - cache str r
      - nullable str msg
- nullable TypedExpr f.expr
- pos f.pos
- doc f.doc
- arr cache ClassField f.overloads

### `EnumField(f)`

- cache str f.name
- cache Type f.type
- pos f.pos
- arr MetadataEntry f.meta
- uleb128 f.index
- doc f.doc
- arr TypeParameter f.params

### `BaseType(c)`

- pos c.pos
- cache str c.name
- cache str c.module
- cache str c.pack (after joining on ".")
- bools(2)
  - c.isPrivate
  - c.isExtern
- arr TypeParameter c.params
- arr MetadataEntry c.meta
- doc c.doc

### `ClassType(c)`

- BaseType c
- bools(2)
  - c.isFinal
  - c.isInterface
- enum c.kind
  - 0 KNormal
  - 1 KTypeParameter(constraints)
    - arr cache Type constraints
  - 2 KExtension(cl, params), 5 KGenericInstance(cl, params):
    - cache ClassType cl
    - arr cache Type params
  - 3 KExpr(e)
    - Expr e
  - 4 KGeneric
  - 6 KMacroType
  - 7 KAbstractImpl(a)
    - cache AbstractType a
  - 8 KGenericBuild
- arr ParamClassType c.interfaces
- nullable ParamClassType c.superClass
- arr cache ClassField c.statics
- arr cache ClassField c.fields
- nullable cache ClassField c.constructor
- nullable TypedExpr c.init
- arr cache ClassField c.overrides

### `EnumType(c)`

- BaseType c
- arr EnumField c.constructors.values()
- arr cache str names

### `DefType(c)`

- BaseType c
- cache Type c.type

### `AbstractType(c)`

- BaseType c
- cache Type c.type
- nullable cache ClassType c.impl
- arr AbstractBinop c.binops
- arr AbstractUnop c.unops
- arr AbstractFrom c.from
- arr AbstractTo c.to
- arr cache ClassField c.array
- nullable cache ClassField c.resolve
- nullable cache ClassField c.resolveWrite

### `AbstractBinop(op)`

- Binop op.op
- cache ClassField op.field

### `AbstractUnop(op)`

- Unop op.op, op.postfix
- cache ClassField op.field

### `AbstractFrom(f)`

- cache Type f.t
- nullable cache ClassField f.field

### `AbstractTo(t)`

- cache Type t.t
- nullable cache ClassField t.field

### `ParamClassType(c, params)`

- cache ClassType c
- arr cache Type params

### `ModuleType`

- enum
  - 0 TClassDecl(c)
    - cache ClassType c
  - 1 TEnumDecl(c)
    - cache EnumType c
  - 2 TTypeDecl(c)
    - cache DefType c
  - 3 TAbstractDecl(c)
    - cache AbstractType c

### `FileEntry`

- enum
  - 0 ModuleType(m)
    - ModuleType m

### `File`

- ... signature "hxb1"
- bools(2) config
  - store docs
  - store positions
- ... FileEntry until EOF
