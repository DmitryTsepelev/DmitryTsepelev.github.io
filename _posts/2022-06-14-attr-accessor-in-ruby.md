---
layout: post
title: "Understading why attr_accessor in Ruby is faster than a regular method"
date: 2022-06-14 9:00:00 +0300
permalink: 'attr-accessor-in-ruby'
description: 'A comparison of how different attr_accessor from custom getters in setters in terms of implementation and performance'
tags: ruby interpreters
---

_Setters_ and _getters_ (or _accessors_) are very common in various object–oriented programming languages. Some languages have shortcuts to define them, others require programmers to write them by hand (hi, Java!). Ruby does have these shortcuts: `attr_reader` generates getters, `attr_writer` generates setter, and, finally, `attr_accessor` generates both:

```ruby
class User
  attr_accessor :email

  # the same functionality:
  def email
    @email
  end

  def email=(value)
    @email = value
  end
end
```

## Writing attr_accessor by hand

Sometimes people who explain metaprogramming use `attr_accessor` as the example. It's a convenient way to explain the concept: `attr_accessor` is a method that sits in the _singleton class_ and, when called, defines methods dynamically:

```ruby
module MyAttrAccessor
  def my_attr_accessor(attr_name)
    my_attr_reader(attr_name)
    my_attr_writer(attr_name)
  end

  def my_attr_reader(attr_name)
    instance_variable_name = attr_to_variable_name(attr_name)

    define_method(attr_name) do
      instance_variable_get(instance_variable_name)
    end
  end

  def my_attr_writer(attr_name)
    instance_variable_name = attr_to_variable_name(attr_name)

    define_method("#{attr_name}=") do |value|
      instance_variable_set(instance_variable_name, value)
    end
  end

  private

  def attr_to_variable_name(attr_name)
    "@#{attr_name}"
  end
end

Object.extend(MyAttrAccessor)
```

## A couple of benchmarks

One day I wondered if `attr_accessor` is _just a shortcut_ or it has a different implementation from what we've seen above. Before digging into the Ruby source code, let's write a benchmark to compare `attr_accessor` with methods written by hand:

```ruby
require "benchmark"

class NativeAccessor
  attr_accessor :value
end

class CustomAccessor
  def value
    @value
  end

  def value=(v)
    @value = v
  end
end

n = 10_000_000

Benchmark.bm do |x|
  na = NativeAccessor.new
  x.report("attr_writer") do
    n.times { |i| na.value = i }
  end

  ca = CustomAccessor.new
  x.report("value=") do
    n.times { |i| ca.value = i }
  end
end

Benchmark.bm do |x|
  na = NativeAccessor.new
  x.report("attr_reader") do
    n.times { |i| na.value }
  end

  ca = CustomAccessor.new
  x.report("value") do
    n.times { |i| ca.value }
  end
end
```

Here are the results:

```bash
                 user     system      total        real
attr_writer  0.316426   0.002821   0.319247 (  0.319398)
value=       0.424947   0.004253   0.429200 (  0.429279)

                 user     system      total        real
attr_reader  0.292869   0.002937   0.295806 (  0.295810)
value        0.349404   0.000952   0.350356 (  0.350363)
```

Writer is 34% faster, while reader shows 18% better performance. Why results are so different?

## Going down to the bytecode

Let's take a look at the bytecode for both implementations. Lets' start with a `CustomAccessor` class:

```ruby
code = <<~RUBY
  class CustomAccessor
    def value
      @value
    end

    def value=(v)
      @value = v
    end
  end

  ca = CustomAccessor.new
  ca.value = 1
  ca.value
RUBY

puts RubyVM::InstructionSequence.disasm(RubyVM::InstructionSequence.compile(code))
```

Here is the bytecode:

```
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(13,8)> (catch: FALSE)
local table (size: 1, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 1] ca@0
0000 putspecialobject                       3                         (   1)[Li]
0002 putnil
0003 defineclass                            :CustomAccessor, <class:CustomAccessor>, 0
0007 pop
0008 opt_getinlinecache                     17, <is:0>                (  11)[Li]
0011 putobject                              true
0013 getconstant                            :CustomAccessor
0015 opt_setinlinecache                     <is:0>
0017 opt_send_without_block                 <calldata!mid:new, argc:0, ARGS_SIMPLE>
0019 setlocal_WC_0                          ca@0
0021 getlocal_WC_0                          ca@0                      (  12)[Li]
0023 putobject_INT2FIX_1_
0024 opt_send_without_block                 <calldata!mid:value=, argc:1, ARGS_SIMPLE>
0026 pop
0027 getlocal_WC_0                          ca@0                      (  13)[Li]
0029 opt_send_without_block                 <calldata!mid:value, argc:0, ARGS_SIMPLE>
0031 leave

== disasm: #<ISeq:<class:CustomAccessor>@<compiled>:1 (1,0)-(9,3)> (catch: FALSE)
0000 definemethod                           :value, value             (   2)[LiCl]
0003 definemethod                           :value=, value=           (   6)[Li]
0006 putobject                              :value=
0008 leave                                                            (   9)[En]

== disasm: #<ISeq:value@<compiled>:2 (2,2)-(4,5)> (catch: FALSE)
0000 getinstancevariable                    :@value, <is:0>           (   3)[LiCa]
0003 leave                                                            (   4)[Re]

== disasm: #<ISeq:value=@<compiled>:6 (6,2)-(8,5)> (catch: FALSE)
local table (size: 1, argc: 1 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 1] v@0<Arg>
0000 getlocal_WC_0                          v@0                       (   7)[LiCa]
0002 dup
0003 setinstancevariable                    :@value, <is:0>
0006 leave                                                            (   8)[Re]
```

The first section represents the top level program, and second one stands for the class body. Third and forth define the bytecode for our custom setter and getter. Let's do the same for the `attr_accessor`:


```ruby
code = <<~RUBY
  class NativeAccessor
    attr_accessor :value
  end

  na = NativeAccessor.new
  na.value = 1
  na.value
RUBY

puts RubyVM::InstructionSequence.disasm(RubyVM::InstructionSequence.compile(code))
```

Here's the result:

```
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(7,8)> (catch: FALSE)
local table (size: 1, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 1] na@0
0000 putspecialobject                       3                         (   1)[Li]
0002 putnil
0003 defineclass                            :NativeAccessor, <class:NativeAccessor>, 0
0007 pop
0008 opt_getinlinecache                     17, <is:0>                (   5)[Li]
0011 putobject                              true
0013 getconstant                            :NativeAccessor
0015 opt_setinlinecache                     <is:0>
0017 opt_send_without_block                 <calldata!mid:new, argc:0, ARGS_SIMPLE>
0019 setlocal_WC_0                          na@0
0021 getlocal_WC_0                          na@0                      (   6)[Li]
0023 putobject_INT2FIX_1_
0024 opt_send_without_block                 <calldata!mid:value=, argc:1, ARGS_SIMPLE>
0026 pop
0027 getlocal_WC_0                          na@0                      (   7)[Li]
0029 opt_send_without_block                 <calldata!mid:value, argc:0, ARGS_SIMPLE>
0031 leave

== disasm: #<ISeq:<class:NativeAccessor>@<compiled>:1 (1,0)-(3,3)> (catch: FALSE)
0000 putself                                                          (   2)[LiCl]
0001 putobject                              :value
0003 opt_send_without_block                 <calldata!mid:attr_accessor, argc:1, FCALL|ARGS_SIMPLE>
0005 leave                                                            (   3)[En]
```

The only difference is that `attr_accessor` does not generate any bytecode! What does it mean? It means that we have to go to the Ruby source code!

## How VM executes attr_reader and attr_writer

Let's find out what happens when `attr_accessor` method is called. Turns out that this method is a C method that is added to the `Module` [here](https://github.com/ruby/ruby/blob/ac75c710cccf7adc5b12edc8d18263fef9ab3207/object.c#L4388):

```c
rb_define_method(rb_cModule, "attr_accessor", rb_mod_attr_accessor, -1);
```

The `rb_mod_attr_accessor` function is defined [here](https://github.com/ruby/ruby/blob/ac75c710cccf7adc5b12edc8d18263fef9ab3207/object.c#L2210):


```c
static VALUE
rb_mod_attr_accessor(int argc, VALUE *argv, VALUE klass)
{
  int i;
  VALUE names = rb_ary_new2(argc * 2);

  for (i=0; i<argc; i++) {
    ID id = id_for_attr(klass, argv[i]);

    rb_attr(klass, id, TRUE, TRUE, TRUE);
    rb_ary_push(names, ID2SYM(id));
    rb_ary_push(names, ID2SYM(rb_id_attrset(id)));
  }
  return names;
}
```

It takes a list of names (in `argv`) and defines `rb_attr` in the specified class (`klass`) for each one. The function called `rb_attr` [contains](https://github.com/ruby/ruby/blob/80ad0e751f4c9aa13a581b61b348c34ede7f3956/vm_method.c#L1712) the actual definition:

```c
void
rb_attr(VALUE klass, ID id, int read, int write, int ex)
{
  ID attriv;
  rb_method_visibility_t visi;
  const rb_execution_context_t *ec = GET_EC();
  const rb_cref_t *cref = rb_vm_cref_in_context(klass, klass);

  if (!ex || !cref) {
    visi = METHOD_VISI_PUBLIC;
  } else {
    switch (vm_scope_visibility_get(ec)) {
    case METHOD_VISI_PRIVATE:
      if (vm_scope_module_func_check(ec)) {
        rb_warning("attribute accessor as module_function");
      }
      visi = METHOD_VISI_PRIVATE;
      break;
    case METHOD_VISI_PROTECTED:
      visi = METHOD_VISI_PROTECTED;
      break;
    default:
      visi = METHOD_VISI_PUBLIC;
      break;
    }
  }

  attriv = rb_intern_str(rb_sprintf("@%"PRIsVALUE, rb_id2str(id)));
  if (read) {
    rb_add_method(klass, id, VM_METHOD_TYPE_IVAR, (void *)attriv, visi);
  }
  if (write) {
    rb_add_method(klass, rb_id_attrset(id), VM_METHOD_TYPE_ATTRSET, (void *)attriv, visi);
  }
}
```

This method does quite a lot: sets up visibility, remembers execution context and, most importantly, adds methods for setter and getter. This behaviour is controlled by `read` and `write` arguments, `attr_accessor` passes both while `attr_reader` and `attr_reader` set one of these flags to false.

We are going to stop here, because `rb_add_method` (which is defined [here](https://github.com/ruby/ruby/blob/80ad0e751f4c9aa13a581b61b348c34ede7f3956/vm_method.c#L1094)) is used for adding all methods to classes. However, let's understand the meaning of the third argument (`VM_METHOD_TYPE_IVAR` and `VM_METHOD_TYPE_ATTRSET`) passed to it. Ruby has 12 method [types](https://github.com/ruby/ruby/blob/e89d80702bd98a8276243a7fcaa2a158b3bfb659/method.h#L109) and turns out that setters and getters have their own type!

This type is used when bytecode is [executed](https://github.com/ruby/ruby/blob/5f10bd634fb6ae8f74a4ea730176233b0ca96954/vm_eval.c#L211):

```c
static VALUE
vm_call0_body(rb_execution_context_t *ec, struct rb_calling_info *calling, const VALUE *argv)
{
  // ...
  switch (vm_cc_cme(cc)->def->type) {
    // ...
    case VM_METHOD_TYPE_ATTRSET:
      vm_call_check_arity(calling, 1, argv);
      VM_CALL_METHOD_ATTR(ret,
                          rb_ivar_set(calling->recv, vm_cc_cme(cc)->def->body.attr.id, argv[0]),
                          (void)0);
      goto success;
    case VM_METHOD_TYPE_IVAR:
      vm_call_check_arity(calling, 0, argv);
      VM_CALL_METHOD_ATTR(ret,
                          rb_attr_get(calling->recv, vm_cc_cme(cc)->def->body.attr.id),
                          (void)0);
      goto success;
  }
}
```

This is the reason why native accessors are faster: they do not execute any bytecode so there is no overhead on setting a context for the execution, it's just a plain C call. In opposite, when we define a regular method (using `def` or `define_method`), a bytecode will be generated and `VM_METHOD_TYPE_BMETHOD` will be used as the method type.
