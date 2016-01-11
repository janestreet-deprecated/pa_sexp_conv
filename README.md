Syntax extension for sexplib
============================

What is Pa\_sexp\_conv?
-----------------------

Pa\_sexp\_conv is a Camlp4 syntax extension which can be used to
automatically generate code from type definitions for efficiently
converting OCaml-values to S-expressions and vice versa.

In combination with the parsing and pretty-printing functionality of
[sexplib](https://github.com/janestreet/sexplib) this frees users from
having to write their own I/O-routines for data structures they
define.  The tight integration with the OCaml type system also allows
for automatically verifying complex semantic properties when
converting from S-expressions to OCaml values.  Possible errors during
automatic conversions from S-expressions to OCaml-values are reported
in human-readable ways with exact location information.  The library
also offers functionality for extracting and replacing sub-expressions
in S-expressions.

Usage
-----

The following three new constructs are added to the language:

```ocaml
with sexp
with sexp_of
with of_sexp
```

The first one implies the last two statements.  When using these constructs
right after a type definition, function definitions will be automatically
generated which perform S-expression conversions.  For example, consider
the following type definition:

```ocaml
type t = A | B with sexp
```

The above will generate the functions `sexp_of_t` and `t_of_sexp`.
The preprocessor also supports automatic addition of conversion functions
to signatures.  Just add "`with sexp`" to the type in a signature, and the
appropriate function signatures will be generated.

Converters for standard types (`int`, `list`, `Hashtbl.t`, etc.)
become visible to the macro-generated code by opening the standard
module before their first use in a type definition.  Users will
therefore usually want to place the following at the top of their
files:

```ocaml
open Sexplib.Std
```

Syntax Specification of S-expressions
-------------------------------------

### Conversion of OCaml-tuples

OCaml-tuples are simple lists of values in the same order as in the tuple.
E.g. (OCaml representation followed by S-expression after arrow):

```ocaml
(3.14, "foo", "bar bla", 27)  <===>  (3.14 foo "bar bla" 27)
```

### Conversion of OCaml-records

OCaml-records are represented as lists of pairs in S-expression syntax.
Each pair consists of the name of the record field (first element), and its
value (second element).  E.g.:

```ocaml
{
  foo = 3;
  bar = "some string";
}
<===>
(
  (foo 3)
  (bar "some string")
)
```

Type specifications of records allow the use of a special type `sexp_option`
which indicates that a record field should be optional.  E.g.:

```ocaml
type t =
  {
    x : int option;
    y : int sexp_option;
  } with sexp
```

The type `sexp_option` is equivalent to ordinary options, but is treated
specially by the code generator.  The above would lead to the following
equivalences of values and S-expressions:

```ocaml
{
  x = Some 1;
  y = Some 2;
}
<===>
(
  (x (some 1))
  (y 2)
)
```

And:

```ocaml
{
  x = None;
  y = None;
}
<===>
(
  (x none)
)
```

  Note how `sexp_option` allows you to leave away record fields that should
default to `None`.  It is also unnecessary (and actually wrong) now to write
down such a value as an option, i.e. the `some`-tag must be dropped if the
field should be defined.

The types `sexp_list`, `sexp_array`, and `sexp_bool` can be used in ways
similar to the type `sexp_option`.  They assume the empty list, empty array,
and false value respectively as default values.

More complex default values can be specified explicitly using several
constructs, e.g.:

```ocaml
let z_test v = v > 42

type t =
  {
    x : int with default(42);
    y : int with default(3), sexp_drop_default;
    z : int with default(3), sexp_drop_if(z_test);
  } with sexp
```

The `default` record field extension above is supported by the underlying
preprocessor library `type_conv` and specifies the intended default value for
a record field in its argument.  Sexplib will use this information to generate
code which will set this record field to the default value if an S-expression
omits this field.  If a record is converted to an S-expression, record fields
with default values will be emitted as usual.  Specifying `sexp_drop_default`
will add a test for polymorphic equality to the generated code such that a
record field containing its default value will be suppressed in the resulting
S-expression.  This option requires the presence of a default value.

`sexp_drop_if` on the other hand does not require a default.  Its argument
must be a function, which will receive the current record value.  If the
result of this function is `true`, then the record field will be suppressed
in the resulting S-expression.

The above extensions can be quite creatively used together with manifest
types, functors, and first-class modules to make the emission of record
fields or the definition of their default values configurable at runtime.

### Conversion of sum types

Constant constructors in sum types are represented as strings.  Constructors
with arguments are represented as lists, the first element being the
constructor name, the rest being its arguments.  Constructors may also
be started in lowercase in S-expressions, but will always be converted to
uppercase when converting from OCaml-values.

For example:

```ocaml
type t = A | B of int * float * t with sexp

B (42, 3.14, B (-1, 2.72, A))  <===>  (B 42 3.14 (B -1 2.72 A))
```

The above example also demonstrates recursion in data structures.

### Conversion of variant types

The conversion of polymorphic variants is almost the same as with sum types.
The notable difference is that variant constructors must always start with
an either lower- or uppercase character, matching the way it was specified
in the type definition.  This is because OCaml also distinguishes between
upper- and lowercase variant constructors.  Note that type specifications
containing unions of variant types are also supported by the S-expression
converter, for example as in:

```ocaml
type ab = [ `A | `B ] with sexp
type cd = [ `C | `D ] with sexp
type abcd = [ ab | cd ] with sexp
```

### Conversion of OCaml-lists and arrays

OCaml-lists and arrays are straightforwardly represented as S-expression lists.

### Conversion of option types

The option type is converted like ordinary polymorphic sum types, i.e.:

```ocaml
None        <===>  none
Some value  <===>  (some value)
```

There is a deprecated version of the syntax in which values of option type
are represented as lists in S-expressions:

```ocaml
None        <===>  ()
Some value  <===>  (value)
```

Reading of the old-style S-expression syntax for option values is
only supported if the reference `Conv.read_old_option_format` is set
to `true` (currently the default, which may change).  A conversion
exception is raised otherwise.  The old format will be written only if
`Conv.write_old_option_format` is true (also currently the default).
Reading of the new format is always supported.

### Conversion of polymorphic values

There is nothing special about polymorphic values as long as there are
conversion functions for the type parameters.  E.g.:

```ocaml
type 'a t = A | B of 'a with sexp
type foo = int t with sexp
```

In the above case the conversion functions will behave as if `foo` had been
defined as a monomorphic version of `t` with `'a` replaced by `int` on the
right hand side.

If a data structure is indeed polymorphic and you want to convert it, you will
have to supply the conversion functions for the type parameters at runtime.
If you wanted to convert a value of type `'a t` as in the above example,
you would have to write something like this:

```ocaml
sexp_of_t sexp_of_a v
```

where `sexp_of_a`, which may also be named differently in this particular
case, is a function that converts values of type `'a` to an S-expression.
Types with more than one parameter require passing conversion functions for
those parameters in the order of their appearance on the left hand side of
the type definition.

### Conversion of opaque values

Opaque values are ones for which we do not want to perform conversions.
This may be, because we do not have S-expression converters for them, or
because we do not want to apply them in a particular type context. e.g. to
hide large, unimportant parts of configurations.  To prevent the preprocessor
from generating calls to converters, simply apply the qualifier `sexp_opaque`
as if it were a type constructor, e.g.:

```ocaml
type foo = int * stuff sexp_opaque with sexp
```

Thus, there is no need to specify converters for type `stuff`, and if there
are any, they will not be used in this particular context.  Needless to say,
it is not possible to convert such an S-expression back to the original value.
Here is an example conversion:

```ocaml
(42, some_stuff)  ===>  (42 <opaque>)
```

### Conversion of exceptions

S-expression converters for exceptions can be automatically registered using
the "`with sexp`" macro, e.g.:

```ocaml
module M = struct
  exception Foo of int with sexp
end
```

Such exceptions will be translated in a similar way as sum types, but their
constructor will be prefixed with the fully qualified module path (here:
`M.Foo`) so as to be able to discriminate between them without problems.

The user can then easily convert an exception matching the above one to
an S-expression using `sexp_of_exn`.  User-defined conversion functions
can be registered, too, by calling `add_exn_converter`.  This should make
it very convenient for users to catch arbitrary exceptions escaping their
program and pretty-printing them, including all arguments, as S-expressions.
The library already contains mappings for all known exceptions that can
escape functions in the OCaml standard library.

---------------------------------------------------------------------------

Contact Information and Contributing
------------------------------------

In the case of bugs, feature requests, contributions and similar, please
contact the maintainers:

  * Jane Street Group, LLC <opensource@janestreet.com>

Up-to-date information should be available at:
* <https://github.com/janestreet/sexplib>
