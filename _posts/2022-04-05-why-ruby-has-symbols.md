---
layout: post
title: "Why Ruby has Symbols"
date: 2022-04-05 10:00:00 +0300
permalink: 'why-has-ruby-symbols'
description: 'Difference between strings and symbols in Ruby'
tags: interpreters ruby
---

Many people who come to Ruby start using symbols on their first day. Come on, [Getting Started with Rails](https://guides.rubyonrails.org/getting_started.html#database-migrations) shows you symbols in the first migration you generate! From the first glance, symbols look like strings (`:hi` versus `"hi"`) and can be converted back and forth. The naive explanation of their difference is that strings are used to represent some data (e.g., constant or variable values), while symbols usually appear as a part of the code (e.g., arguments of method calls). However, why do we have both? Let's figure that out!

## How strings and symbols are used in Ruby?

Imagine that you work on a Rails application and want to add users to your data model. Here is the migration:

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email
      t.text :password

      t.timestamps
    end
  end
end
```

When table schema is in place, we need to create the corresponding model:

```ruby
class Article < ApplicationRecord
  validates :email, :password, presence: true
  encrypts :password
end
```

Later we will create a new controller and write specs (right? 🙂), but let's take a break from coding here. How many times `:email` and `:password` were used already? Two times `:email` and three times `:password`, and there will be more in the real application!

However, let's think about the values that are going to be stored in the `email` column. Each user has a unique email and these emails _never appear_ in the codebase, they can only be initialized in memory when either submitted by user or read from the database.

Looks like both Ruby interpreter and virtual machine can get some use of the fact that symbols and strings are used for different purposes. Let's find out how our code is executed and figure it out!

## How Ruby executes our code

When Ruby program is run, it should be compiled to the bytecode first. _Virtual machine_ (usually referred as _VM_) knows how to execute the bytecode on the concrete operating system.

> Compilation step was added in Ruby 1.9

First of all, interpreter _parses_ your program and gets an Abstract Syntax Tree (usually referred as AST) containing tokens that appear in the source code. I'm going to use a gem called [parser](https://github.com/whitequark/parser) to examine an AST of the `Article` class along with it's usage:

```ruby
require 'parser/current'

Parser::CurrentRuby.parse <<~RUBY
  class Article < ApplicationRecord
    validates :email, :password, presence: true
    encrypts :password
  end

  Article.find(id).password
RUBY
```

Here is an AST we get:

```ruby
s(:begin,
  s(:class,
    s(:const, nil, :Article),
    s(:const, nil, :ApplicationRecord),
    s(:begin,
      s(:send, nil, :validates,
        s(:sym, :email),
        s(:sym, :password),
        s(:hash,
          s(:pair,
            s(:sym, :presence),
            s(:true)))),
      s(:send, nil, :encrypts,
        s(:sym, :password)))),
  s(:send,
    s(:send,
      s(:const, nil, :Article), :find,
      s(:send, nil, :id)), :password))
```

Please note, that token `:password` is used in different contexts: as an argument for two methods (`validates` and `encrypts`) and as a method name (`.password`).

> According to [Wiki](https://en.wikipedia.org/wiki/Lexical_analysis#Token), token is a "lexical token or simply token is a string with an assigned and thus identified meaning. It is structured as a pair consisting of a token name and an optional token value"

When AST is built, it is validated to make sure it makes sense (that's called _lexing_) and converted it to the bytecode. Bytecode is a language that can be executed by the Virtual Machine on any hardware you want.

This is how we can see the bytecode representation for the Ruby source:

```ruby
code = <<~RUBY
  class Article < ApplicationRecord
    validates :email, :password, presence: true
    encrypts :password
  end

  Article.find(id).password
RUBY

RubyVM::InstructionSequence.disasm(RubyVM::InstructionSequence.compile(code))
```

...and here is the bytecode itself:


```ruby
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(6,25)> (catch: FALSE)
0000 putspecialobject                       3                         (   1)[Li]
0002 opt_getinlinecache                     11, <is:0>
0005 putobject                              true
0007 getconstant                            :ApplicationRecord
0009 opt_setinlinecache                     <is:0>
0011 defineclass                            :Article, <class:Article>, 16
0015 pop
0016 opt_getinlinecache                     25, <is:1>                (   6)[Li]
0019 putobject                              true
0021 getconstant                            :Article
0023 opt_setinlinecache                     <is:1>
0025 putself
0026 opt_send_without_block                 <calldata!mid:id, argc:0, FCALL|VCALL|ARGS_SIMPLE>
0028 opt_send_without_block                 <calldata!mid:find, argc:1, ARGS_SIMPLE>
0030 opt_send_without_block                 <calldata!mid:password, argc:0, ARGS_SIMPLE>
0032 leave

== disasm: #<ISeq:<class:Article>@<compiled>:1 (1,0)-(4,3)> (catch: FALSE)
0000 putself                                                          (   2)[LiCl]
0001 putobject                              :email
0003 putobject                              :password
0005 putobject                              true
0007 opt_send_without_block                 <calldata!mid:validates, argc:3, kw:[presence], FCALL|KWARG>
0009 pop
0010 putself                                                          (   3)[Li]
0011 putobject                              :password
0013 opt_send_without_block                 <calldata!mid:encrypts, argc:1, FCALL|ARGS_SIMPLE>
0015 leave                                                            (   4)[En]
```

There are two sections starting with `== disasm`. Each section represents a single _lexical scope_: one for the top–level code and one for the code inside class. Each line is a single bytecode instruction. For instance `putobject` adds it's argument to the _stack_, `opt_send_without_block` handles sending a message to another object, `putself` changes the `self` value, etc.

> Technically bytecode (or, more precisely, "code that VM executes") instruction can take more than one byte but that's one of the popular formats.

Why not use the AST itself to execute the program? It is possible, but will perform worse: interpreter would have to keep the huge object graph in the memory and navigate through it each time it needs to execute something. In opposite, bytecode consists of simple instructions that are so small that can even be kept in the processor cache.

## How Ruby VM stores symbols

We know that some tokens are used many times and never change, can we somehow avoid storing all of them in the memory? This is exactly what Ruby does: each token is stored in the hash table and represented as an ID, which takes _way less memory_ to store.

> Hash table is the most important data structure for building programming languages

When symbol is used, Ruby looks for that symbol in the hash table (the read operation has O(1) complexity), and adds it to the table if needed. Let's see if symbols consume less memory using [benchmark/memory](https://github.com/michaelherold/benchmark-memory) gem:

```ruby
require "benchmark/memory"

Benchmark.memory do |x|
  x.report("strings") do
    100_000.times { "string" }
  end

  x.report("symbols") do
    100_000.times { :symbol }
  end

  x.compare!
end
```

The plan was to initiate the same string and the same symbol 100 000 times and count allocations. Turns out that symbol benchmark allocated nothing, because `:symbol` was allocated during the interpretation as we expected:

```bash
Calculating -------------------------------------
             strings     4.000M memsize (     0.000  retained)
                       100.000k objects (     0.000  retained)
                         1.000  strings (     0.000  retained)
             symbols     0.000  memsize (     0.000  retained)
                         0.000  objects (     0.000  retained)
                         0.000  strings (     0.000  retained)

Comparison:
             symbols:          0 allocated
             strings:    4000000 allocated
```

It's easy to make sure that the symbol was allocated only once, while we got a new string during the each iteration. Let's compare `object_id`:

```ruby
5.times.map { "string".object_id }
# => [13700, 13720, 13740, 13760, 13780]

5.times.map { :symbol.object_id }
# => [921948, 921948, 921948, 921948, 921948]
```

Unfortunatelly it's impossible to access the symbols table from the Ruby code (because Ruby does not expose such API). However, we can see a [list representation](https://github.com/ruby/ruby/blob/d92f09a5eea009fa28cd046e9d0eb698e3d94c5c/string.c#L11512) of symbols (I'm taking only 10 because the list is rather huge):

```ruby
Symbol.all_symbols.sample(10)

# =>
#   [:binding_irb,
#   :@errors,
#   :backtrace_locations,
#   :on_string_embexpr,
#   :details_for_unwind,
#   :build_details_for_unwind,
#   :_enumerable_filter_map,
#   :on_const_path_ref,
#   :confirm_multiline_termination,
#   :conflicting]
```

Symbols can be added to the table in the runtime:

```ruby
Symbol.all_symbols.last # => :las
:my_new_symbol
Symbol.all_symbols.last # => :my_new_symbol
```

Why I didn't do something like `Symbol.all_symbols.include?(:my_new_symbol)`? The reason is that such a call will always return `true`: `:my_new_symbol` will be initialized and added to the table _before_ the `.include?` call. At this moment experienced functional programmers should feel disguisted because this sounds like something not pure at all 🙂

> For more details read "Understanding how symbols differ from strings" chapter in [Polished Ruby Programming](https://www.amazon.com/Polished-Ruby-Programming-maintainable-high-performance-ebook/dp/B093TH9P7C) by Jeremy Evants

Let's think why the list of symbols was not empty when we accessed it for first time. Firstly, Ruby standard library contains some symbols which are already loaded. Secondly, if we look at the symbol list closer we might see some old friends like `:sort`, which is the name of the method for sorting enumerables. Turns out that tokens representing constants, variable names, method names, class names and so on are stored _in the same table_!

When we called `Symbol.all_symbols.last` for the first time, it returned `:las`. If we look at the last symbols in the list we'll notice `:la` and `:las`. What is it? Looks like it is the way how hash tables are implemented: Ruby builds a tree using chars as nodes (starting from two first nodes, because otherwise the tree will have 27 nodes on the first level), and symbols become are leafs.

> Garbage collection removes unused data from memory, so we should not worry about it. When trees were high and computers were huge, people did it manually. Well, some people still do it manually when dealing with languages like C.

_Garbage Collector_ (usually referred as _GC_) can collect unused strings, but what about symbols? [Ruby 2.2.0](https://bugs.ruby-lang.org/issues/9634) introduced two types of symbols: ones that are used to create dynamic methods/instance variables/constants are _immortal_ and never collected; all others are _mortal_ and available for the collection.

![a story about manual GC](/assets/kids.jpg)

## Interning strings

The approach when strings are stored in the hash table and referred as integer IDs is called _string interning_. When VM needs to compare two different string objects in the memory, it has to do it byte by byte, which is not very performanct. String interning is a technique when some (or all) strings are stored in the hash table and string IDs are used instead of them. Of course, there is a guarantee that all table values are unique. Comparsion becomes dead simple: we just need to compare two integers.

> This section is heavily inspired by "Interning strings" chapter from [Crafting Interpreters](https://craftinginterpreters.com) by Robert Nystrom, check it out if you want to dig into language implementation details. Well, I'd say check it out anyway—it's just awesome.

Different languages use string interning in different proportions: e.g., Lua interns all strings, Java interns constants, Ruby, as we know, has symbols. By the way, Ruby can _freeze_ strings which makes them immutable, but it does not make them interned to make GC able to collect them.

Is there a special bytecode operator that makes string interned? No, because it's just one of possible optimizations which happens _as a part_ of other higher–level operations. For instance, `# 0001 putobject :email` is likely going to add `:email` to the symbols table if it's not there yet.

Let's open up the source code of Ruby itself and check out how it works. Symbol allocation is happenning in the `rb_id_attrset` function of [symbol.c](https://github.com/ruby/ruby/blob/master/symbol.c#L113), which can called both from compilation and interpretation stages:

```c
ID
rb_id_attrset(ID id)
{
    VALUE str, sym;
    int scope;

    if (!is_notop_id(id)) {
	switch (id) {
	  case tAREF: case tASET:
	    return tASET;	/* only exception */
	}
	rb_name_error(id, "cannot make operator ID :%"PRIsVALUE" attrset",
		      rb_id2str(id));
    }
    else {
	scope = id_type(id);
	switch (scope) {
	  case ID_LOCAL: case ID_INSTANCE: case ID_GLOBAL:
	  case ID_CONST: case ID_CLASS: case ID_JUNK:
	    break;
	  case ID_ATTRSET:
	    return id;
	  default:
	    {
		if ((str = lookup_id_str(id)) != 0) {
		    rb_name_error(id, "cannot make unknown type ID %d:%"PRIsVALUE" attrset",
				  scope, str);
		}
		else {
		    rb_name_error_str(Qnil, "cannot make unknown type anonymous ID %d:%"PRIxVALUE" attrset",
				      scope, (VALUE)id);
		}
	    }
	}
    }

    /* make new symbol and ID */
    if (!(str = lookup_id_str(id))) {
	static const char id_types[][8] = {
	    "local",
	    "instance",
	    "invalid",
	    "global",
	    "attrset",
	    "const",
	    "class",
	    "junk",
	};
	rb_name_error(id, "cannot make anonymous %.*s ID %"PRIxVALUE" attrset",
		      (int)sizeof(id_types[0]), id_types[scope], (VALUE)id);
    }
    str = rb_str_dup(str);
    rb_str_cat(str, "=", 1);
    sym = lookup_str_sym(str);
    id = sym ? rb_sym2id(sym) : intern_str(str, 1);
    return id;
}
```

---

That's all for today, let's summarize what we found here.


|             | String                     | Symbol                               |
| ----------- | -------------------------- | ------------------------------------ |
| Allocation  | allocate memory for each * | never duplicated                     |
| Comparsion  | byte by byte, slow         | integers in the runtime, fast        |
| GC          | can be collected           | only mortal                          |

<sup>*</sup> There is another optimization for strings that makes a copy of a string only when it's modified (it is called Copy on Write or CoW):

Here's the great example of how CoW works by [Kernigh](https://www.reddit.com/user/Kernigh/) (see the whole [response](https://www.reddit.com/r/ruby/comments/twqz89/why_ruby_has_symbols/i3piyiv/?utm_source=reddit&utm_medium=web2x&context=3)):

```ruby
require 'objspace'
GC.disable

# Initialize strings and their tails. Tails share storage.
strings = ('a'..'j').map { |char| char * 2_000_000 }
tails = strings.map { |s| s[1_000_000, 1_000_000] }
GC.start
size0 = ObjectSpace.memsize_of_all

# Trigger CoW by modifying tails:
tails.each { |s| s[0] = 'm' }
GC.start
size1 = ObjectSpace.memsize_of_all

printf "before CoW: %s bytes\n", size0
printf " after CoW: %s bytes\n", size1
```

This script makes 20 million bytes of strings and 10 million bytes of tails, which are sharing storage with previously initialized strings. Then it remembers the memory size and modifies tails, which makes it impossible to share storage with strings (i.e., we trigger a copy operation). Then we take new a memory size and compare:

``bash
before CoW: 23531454 bytes
 after CoW: 33538113 bytes
```

What's the most important thing to remember? Strings and symbols are easy to convert back and forth (`to_s`/`to_sym`) but remember that each string allocation takes memory, while symbols are deduplicated and faster to compare.
