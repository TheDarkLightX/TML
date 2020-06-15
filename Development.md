# Introduction

# Universe

# Builtins

There're two types of builtins at the moment from the implementation perspective: 
a) head/left-hand-side (lhs) and  
b) right-hand-side (rhs).  

> there're a good deal more: external/internal; then bdd-s, terms or raw terms 
(I guess compressed/decompressed too) - frankly I'm not really sure of the actual 
details here till we implement some examples for each.
  
##### How to implement 

- `str_bltins` (input.h) - add your builtin to the set.  
(TODO: we should reorg this to support externally loaded builtins)
- `alt` contains the builtin data (alt is the holder of the builtin data) - 
for the rhs builtins.  
- builtin is a term, but initially it was decided to allow only 
**one builtin per alt** (I can't find it but I'm pretty sure:). I'm not sure if 
this is wise, but it's a limitation atm (technically speaking there's no 
reason why we couldn't keep a list of builtins per alt). However, you can have 
one builtin in the head (lhs) and one on the rhs, as those are kept separately 
(more below).  
- `a.idbltin` (i.e. `alt.idbltin`) - is `>=0` if a term is a builtin. And 
`lexeme bltintype = dict.get_bltin(a.idbltin)` to get the type/name.   
- `a.bltinargs` - has all the builtin args.
- `a.bltinsize` - has # of vars.
- `bltinout = a.bltinargs.back()` - e.g. to get the last `?out` var (in case 
builtin has one). And `a.vm.at(bltinout)` to get the arg position in the alt. 
Then you can use that (alongside `a.varslen`, `a.bm`) for any bdd ops needed.
- rhs builtins are typically implemented inside the `alt_query()` 
(see `count, rnd`).
- head/lhs builtin info is kept with the table (as it's a rule), same members.
- best place to implement it is within `fwd()` (e.g. `lprint, halt, fail`). 
- lprint implementation details: this depends on what we want to achieve, but 
basically we're watching the builtin table's bdd for any changes (or when done), 
then decompressing, filtering for any new items and outputing (in this case). 
Decompressing is slow so do it only if something changes (or when done), not on
each step (it's required here so we can't avoid it).
- TBD.
  
##### Type Inference & Builtins

You also need to implement *type inference* for any new builtin.  

See `count` specifics in the *infer_types.cpp* (`get_alt_types`).  
This is a bit more involving (more below) but simply put if the builtin 
influences or adds a new arg/type, then builtin is the one responsible for it. 
Typically that involves calling the `bitsmeta::sync_types` with the alt/arg +
also with the table/arg (behind the alt), if arg is in the head.   

This makes sense for `count` as count introduces the `?out` var, which is 
numeric.  

##### Type Inference and Ops

This also goes for any new ops/features.   

Similarly this should be done for arithmetics (or any new operators etc.), as 
depending on the op the type needs to be set (whether that's the result or 
whatever).  


# Variable Bits

- fixed `bits` (and universe counters `nums chars syms`) is no longer defined. 
So everything that uses bits (which is bdd-s or related to it) needs to use 
*variable bits* now.  

- `bits` were used under the hood, as it was global. `variable bits` need to 
be specified and passed into each relevant call.  

- bdd-s (and args, types) are typically defined either at a) alt level, or at b) 
table level. Also any custom handling (like arithmetics) defines it's own bdd-s.
I call those *bdd-holders* (or *bdd-context* is better probably?).  

- for each of those we need to have a *variable-bits* structure properly 
initialized - which is `bitsmeta`. The `bitsmeta` keeps all the relevant data 
about arguments and types. Where you used `bits` before, you need to pass 
`.bm` now (loosely speaking).  

- `bits` equivaluent is `bm.types[arg].bitness` - i.e. you need an argument,
bits is meaningful only 'per arg'. One consequence is that you always have to 
iterate over args first, then get the bits then iterate over bits (arithmetics).  

- `args * bits` equivalent (of total # of bits) is `bm.args_bits`.  

- `bm.get_args()` to get # of arguments, which is equivalent to either 
`a.varslen` or `table.len`.

- When you call `from_sym | from_sym_eq | leq_const ...` you need to pass to 
it a `bitsmeta`. `bitsmeta` is sort of a *context* for any bdd-related call. 
Similarly for any low-level bdd op you'd need to use `bitsmeta.pos()`. I.e. 
anything you do with bits, bdd-s, requires a `bitsmeta` instance (or two). 
There's an overload variant for most functions (i.e. you can pass either `a.bm`
or just `a`, usually the last param). Note: ideally this should go e.g. into a 
base (template) for alt/table, so you won't need to pass anything, but the code 
reshuffling needed was just too much for this pass (needs some more thinking).  

- which .bm (bitsmeta) to use: it's usually clear from the context, if you're
dealing with a table (if `arg` is tbl arg or `args` is tbl.len) use `tbl.bm`,
if `a.vm` was used or `a.varslen` is passed - use `a.bm` (total # of args is
usually a very good indicator).  

- for ex/perm it's a slightly different story. To use ex/perm, you don't need 
anything special (and chances are that's all you'll ever need to do). To 
initialize ex/perm of any kind you always need to pass two `.bm` structs, as
you're mapping from one set of arguments to another (varying in #, positions, 
bitness etc.). **This may be of interest to arithmetics**, as there we have 
neither tbl or alt args/bits, but a custom arrangement (so ex/perm will probably
have to be made from scratch).  

- when creating a custom table, i.e. it's not done in `from_raw_term` (as is 
the case for `load_string`, `transform_grammar` or `transform_bin`), types 
/ args need to be specified properly (e.g. arithmetics again, see those 
implementations for more).  

- *variable bits* also affect how things need to be initialized, and in which 
order. **All `bitsmeta`** structs (tbl or alt) **need to be constructed & 
initialized before any bdd op** is done (i.e. before the first bdd call). If any
changes are made to the `bitsmeta` (type, bitness, # of args, ordering) => we
need to permute all bdd-s for that bdd-context (tbl, alt or custom). That is 
supported too (`AddBits` and `add_bit`, `permute_type`, more below) but it's 
costly (and needed only in special cases). That's the reason for added 
complexity in `add_prog` initialization (we first init bitsmeta for all, grammar
too, do type inference, init tables/alts, and only then proceed w/ get_rules).
This doesn't affect custom bdd contexts (like arithmetics) as long as you init 
before doing any bdd ops.  

- `nums` in `bitsmeta`: what was `nums` before, it's now 'per arg'. When 
creating custom bdd context (or custom table) you need to supply both `types` 
and `nums`. Nums is basically the upper limit of the universe for that arg, only
for numerics (i.e. the max numeric value, per arg). Chars is known, and syms 
(for symbols) is still not implemented properly yet, all syms share one universe.


### Types  

Types are essential for `bitsmeta` (there's a type for each arg).  
  
- `arg_type` - is the base type structure for predefined or built-in types 
(sort of primitives + symbols + strings). Naming will change, to be renamed to
(`type`, `builtin_type : type`). That'd be extended further with e.g. 
something like `sub_type`, `record_type` etc. to support custom types.  

- we need to define how custom types will be specified (parser).  

- types have to match (for related types), see readme.md also.  

- bitness is either specified (`:type[bitness]`), inferred (from related args),
or calculated from consts used (for numerics mostly, syms also to come).  

- symbols are not finished yet properly, all share the same universe (due to the 
`dict` dependency). We need to split symbols 'per arg' instead. TBD.  

- symbols vs strings - and `str | sym` - TBD.   

- dependent types are 'matched' (during inference) but we still keep copies of 
all those identical types all over the place. Couple reasons: a) laziness (I'm 
planning to reorganize, keep just representative types & just change those); b)
types have very small footprint (could go into one int if needed) and we rarely
change them (once set up, only in `add_bit`); c) if we add *casting* dependent
types will no longer be 'equated' and we can have similar types but with 
differences. So, I'm unsure if we really want to 'compress' that any further.  

- there's one difference in output and universe handling from fixed to variable. 
Universe can no longer be anything, like both numeric and char and sym at the 
same time (for one arg). That rarely showed up before but it can be seen with 
`transform_bin` (and `tml.g`), temp `r` tables have final output with 'mixed 
universe'. With types and current variable bits, universe is 'strongly typed' 
and an arg can have only one specific type (num, chr or str). If we'd really 
need to have it 'mixed' then we can add a `union_type` to specifically enable 
it.

TBD.

### Type Inference  

- Inference is done via `sync_types` (bitsmeta) and `map_type` (infer_types) 
(`sync_types` syncs similar arg/types, while `map_type` builds dependency maps 
and `propagate_types` makes sure all args have the latest / best type).  

- Inference is quite detailed, it goes and matches alts, bodies/tables, takes all
ops (e.g. relational `< <= ==`) & builtins specifics into account. That was 
necessary to be able to run any old TML w/o any manual typing.  

- some things are assumed (which may not always be desired), e.g. if we have 
`r(?v1 ?v1)` we assume that type(r, 0)==type(r, 1), as that's logical. Also,
if same vars are used in different places, we assume that those arguments have
the same type. That is simplifed, helps, but may not always be what we want
(though I don't see how to avoid it, it's a logical issue of such tml - or we 
may be missing some advanced features, but given current scope we can't do more
IMO). 

- `minvtyps` - is a full dependency list of args/types. Keys hold all 'root' 
types (usually there're only a few of those) and `minvtyps[root]` holds all 
dependent types. `mtyps` is inverse of that, `get_root_type` etc. This 
dependency doesn't imply 'direction' or tables dependency, it's not like `deps`. 

- inference maps (the above) are now used for add_bit (AddBits) as well (which
is more optimal).  

TBD.

### Dynamic Universe

This was the trickiest of all. We can now basically add bits (expand universe),
or change bits order, change vars order etc.

##### bits order
- we can order args/bits in any way we like, at compile time, or at runtime.
E.g. (0..bits0)(0..bits1)...(0...bitsN), or like it is now, args-together 
(same-bit):

|0 | 1 | 2 | 3 |  args |
|:---:|:---:|:---:|:---:|:---:|
|0 | 0 | 0 | 0 | |
|1 | 1 | 1 | 1 | |
|2 | 2 | 2 |   | |
|3 | 3 | 3 |   | |
|4 | 4 | 4 |   | |
|5 | 5 | 5 |   | |
|  | 6 | 6 |   | |
|  | 7 | 7 |   | |
|  | 8 | 8 |   | |
|  | 9 | 9 |   | |
|  |   | 10|   | |
|  |   | 11|   | 0-bit (goes up) |


TBD.

##### addbit

- it can be tested with `--addbit`, one bit will be added (on arbitrary 
tbl/arg), universe will increase and all should go on as before.  

- it is called like `AddBits.permute_type({tab, arg});` in `fwd` or similar. 
Also, some more details in tables_ext.cpp (commented out) and in `fwd`.  

- it adds a bit, propagates the change (to all depending type/args), reinits all
bitsmeta-s, tbl-s, alt-s - and permutes all bdd-s. 

- currently supported is `addbit(1)` (one bit at the time), n-bits will be 
added shortly.  

- to change var positions we need to set the `vargs` and call the same routine
(I'd need to test, tweak that).

##### addbit and sequences

- There were some tricky issues with previous (fixed bits) implementation (bits 
/ nums that changes with new sequence, also `consts bit-encoding` vs `addbit` 
direction didn't match, `from-left` or `from-right`). This should now be fully 
stable and supporting any bits change, layout, order etc.  

- This also solves the 'universe overflow' issue that was reported.  
`{a(1).m(120).} {m(128).}`  

- `get_facts` overwriting / anulling for new sequences (needs to be rechecked).


TBD  

***


# Outputs

`output` is a class wrapping an output stream from TML. It allows to
redirect each of the TML outputs to stdout, stderr, a file, a string
buffer (for reading it later programatically) or to a null.
Outputs are usually configured (targeted) by `options` class.

## creating a new output

Use method `shared_ptr<output> output::create(std::wstring name, std::wstring
target, std::wstring extension);` where all arguments are wstrings.

## targets of outputs

Possible targets are:
- @null
- @stdout
- @stderr
- @buffer
- filename - any string not matching any of above names

## outputs container

There is a container structure for `output` objects.
Pointer to this container is passed to the `options` object which configures the
outputs. Name of the output is matched when a STRING option with the same name
is parsed. More details for configuring outputs see bellow in the *Options*
section.


## default outputs

After creating `outputs` object. You can call its method `init_defaults()` to
create default outputs.

Default outputs are:

|             |                                                   |
| ----------- | ------------------------------------------------- |
| output      | default output for TML print builtins             |
| error       | error messages                                    |
| info        | info messages                                     |
| debug       | debug messages (requires Debug build)             |
| dump        | database dump                                     |
| benchmarks  | measured benchmarks                               |
| transformed | transformed program                               |
| repl-output | repl                                              |
| xsb         | program translated into XSB                       |
| swipl       | program translated into SWIPL                     |
| souffle     | program translated into Soufflé                   |


`outputs` object acts also as a global container. It is possible to use multiple
outputs objects for example when you need to use multiple drivers at the same
time. Switch between `outputs` objects by calling `outputs::use()` or
`o::use(const outputs&)` method.

Example:
```
	outputs oo;
	oo.init_defaults();
	oo.use();
```
This is required when using multiple `outputs` objects.
If there is only a single `outputs` container used in the whole program, calling
`init_defaults()` and `use()` is not necessary since first such object ever
instantiated calls these two methods automatically. It is enough to use just:
```
	outputs oo;
```

## adding new output

Adding new output is possible with calling `outputs::add(shared_ptr<output>)`
or directly: `outputs::add(output::create(name, target, extension))`

## **o** namespace

There is a namespace `o`  to allow quick access to global streams (global
`outputs` object is the first such object or the one which was set global by
calling `outputs::use()`).

For convenience there are methods with quick access to configured wostreams:
- `o::out()`
- `o::err()`
- `o::inf()`
- `o::dbg()`
- `o::repl()`,
- `o::ms()` - this is for **benchmarks** output
- `o::dump()`

There is also `o::to(const std::wstring&)` to get output's wostream by its name.


# Options

There is an `options` class which is used to pass and parse options for
a TML driver. Its constructor or parse function can take parameters with
`int argc/char* argv`, `strings args` or `wstrings args` API.

If you need `options` to configure `outputs`, use a pointer to the container
in constructor or call `set_outputs(outputs*)` before calling `parse()`.

Example:
```
	outputs oo;
	options opts(&oo, argc, argv);
	driver d(opts);
```
or:
```
	std::vectors<std::string>args{ "--no-dump","--transformed","@stdout" };
	outputs oo;
	driver d(options(&oo, args));
```

## reading options

`options` object has methods for reading values: `enabled()`, `disabled()`,
`get_bool()`, `get_int()`, `get_string()`. All take `const wstring& option_name`
as an argument.

Example:
```
	if (o.enabled(L"help")) help();
	else d.run(o.get_int(L"steps));
```

## available options

Run `tml --help` (or `options::help(const wostream&)` programatically) to see
a help generated from all available options and their descriptions.


## adding a new option

Add a new option in the method `void options::setup()`. See its content for
examples.
There is a function `add` adding an option defined by type, names (alternative
names) and a lambda function (event) called when the option is present/parsed.
There are helper macros `add_bool`, `add_output` for adding bool options and
outputs.

Use `option::description(const std::wstring&)` setter for adding a description
text. The text is used for the generated help.

### option types

There are currently three different types for options
- INT (uses result of std::stoi(arg))
- BOOL (true if arg is one of "", "true", "t", "1", "on", "enabled", "yes"
	false if arg is one of "false", "f", "0", "off", "disbled", "no")
- STRING (any value basically)

### output option

Outputs are not special types. They use STRING type. Output name is also the
name of the option which is used to configure output's target.

Use: `--OUTPUT_NAME @target`.

### default value for an option

Best place to add a default value for an option is in a method `void
options::init_defaults()`.

### negating/disabling options

bool options and output options can be negated by adding `disable-`, `no-` or
`dont-` prefix to the option name. Example:
```
  tml --dont-run --no-dump --disable-debug
```


# Driver

`driver` class manages execution of TML programs.

Before running a TML program we need to create an outputs object which contains
output streams.
Then we have various ways to instantiate a driver object.
We can provide a TML program as a string, as a FILE stream and/or we can also
use options object and specify any option including input programs.
Options object parses arguments and configures streams in the outputs
object.

For execution there are two methods
`step(size_t steps = 1, size_t break_on_step=0, bool break_on_FP = false)` or
`run(size_t steps = 0, size_t break_on_step=0, bool break_on_FP = false)`
0 steps means infinite number of steps and 0 break_on_step means to not break

Example:
```
	outputs oo;
	wstring program = L"a(2). b(?x) :- a(?x).";
	vector<wstring> args{
		L"--dump", "@buffer",
		L"--error", "@buffer" };
	driver d(program, options(&oo, args));
	d.run();
	if (d.result) d.dump();
	else wcout << L"unsat";
	wstring err = oo.read(L"error");
	if (err.size()) wcerr << err;
	wstring dump = oo.read(L"dump");
	if (dump.size()) wcout << dump;
```

To extract result there are several functions:
`driver::dump()` prints database to the "dump" output
`driver::out(wostream&)` prints database to a stream
`driver::out_goals(wostream&)` prints extracted proofs

It is also possible to pass a custom raw term printer:
`driver::out(const tables::rt_printer& p)` which gets called for each printed term.

See actual usage of `driver` in `src/main.cpp` and `src/repl.cpp`


# JS (emscripten) build

There is an emscripten binding allowing TML run in browsers or with Node.js.
Enable the JS build with `-DBUILD_JSLIB=1 -DEMSCRIPTEN_DIR=/path/to/emscripten`

See example usage of JS `driver` in `js/test.html` and `js/tmljs` (Node.js)


# REPL

TML executable can be run in a REPL mode. Enable it with `--repl` option.
REPL mode depends on a TML build configured with `-DWITH_THREADS=TRUE`.

After running `tml --repl` enter `q` to quit, `?` or `h` to print help about
other commands, or enter TML program to evaluate.

REPL can also work over an UDP. Enable it with `--udp` option.
You can specify IP address and port by using `--udp-addr` and `--udp-port`.
Settings default to `127.0.0.1:6283`.
