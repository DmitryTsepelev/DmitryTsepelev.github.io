---
layout: post
title: "How to make Ruby interpreter run program written in a natural language"
date: 2022-05-10 9:00:00 +0300
permalink: 'natural-language-programming-with-ruby'
description: 'Building a Ruby DSL that can understand and execute a program written in an almost natural language'
tags: ruby interpreters
---

Many programming languages pretend to be _almost natural_, they call it a "regular English". So does Ruby. Come on, does it sound like a language we speak? 🙂

```ruby
class UserController < ApplicationController
end
```

Let's make a language that will be trully natural! I want my programs to look like this but run them using Ruby interpreter:

```ruby
assign variable a value 1
assign variable b value 2
sum a with b
```

---

## Teaching Ruby interpreter to run program written in regular English

Try to put the following code to `natural.rb` and run it with Ruby interpreter (`irb ./natural.rb`):

```ruby
assign variable a value 1
assign variable b value 2
sum a with b
```

The error you'll see is `<main>: undefined method value for main:Object (NoMethodError)`. Why doesn't it complain about all other weird words, like `assign` or `b`?

The reason is that the snippet above is a completely valid Ruby code from the parser perspective. For instance, the line `sum a with b` can be read like this:

- we either call method `b` or access the variable `b`;
- the result is passed to the method `with`;
- the result of `with` call is passed to the method `a`;
- the result of `a` call is passed to the method `sum`.

As a result, interpreter fails when it cannot access the first method from the left: in our case—it's `value`.

> You might think that the language we use is not super natural. I completely agree and initially wanted to do something like `assign 1 to variable a`. The problem is that this code is illegal in Ruby: parser wants a comma after the number.

This is how we can inspect the order of method calls:

```ruby
def assign(*)
  puts "assign"
end

def variable(*)
  puts "variable"
end

def a
  puts "a"
end

assign variable a

# => a
# => variable
# => assign
```

Thanks to our star–argument–based executor (he–he), this code is not failing anymore, but also does not do anything useful. Let's fix that—meet our super naive and basic [implementation](https://gist.github.com/DmitryTsepelev/01702a27e86dd774d44998c3a3894dce#file-02_naive-rb):

```ruby
@variables = {}
@unknown_token = nil
@current_value = nil
@with = nil

def assign(*); end

def variable(*)
  @variables[@unknown_token] = @current_value
end

def value(value)
  @current_value = value
end

def method_missing(m, *args, &block)
  @unknown_token = m
end

def sum(*)
  result = @variables[@unknown_token] + @with
  print "#{result}\n"
end

def with(*)
  @with = @variables[@unknown_token]
end

# Program

assign variable a value 1
assign variable b value 2
sum a with b
```

Run it, and you'll see 3 printed to your console.

Let's read the code from the right to the left, like the interpreter does. We start with a `value` method that accepts a number and stores it in the global variable:

```ruby
@current_value = nil

def value(value)
  @current_value = value
end
```

Looks like we just need to define a method for each word that exists in our language! What's next? Oh wait, `a` is a variable name, and we cannot define methods for all possible variable names! Fear not, we can use `method_missing` to handle that.

> `method_missing` is invoked when Ruby object gets a message it cannot respond, read more [here](https://apidock.com/ruby/BasicObject/method_missing)

Let's try to put the unexpected methods name to another global variable:

```ruby
@unknown_token = nil

def method_missing(m, *args, &block)
  @unknown_token = m
end
```

Our next goal is to implement a method `variable`. We could accept the result of the previous method, but, since it was `method_missing`, we have to do a little trick: we will just read variable name and its value from global state. Also, we have a hash variable called (surprize! 🙂) `@variables` to store our variables:

```ruby
@variables = {}

def variable(*)
  @variables[@unknown_token] = @current_value
end
```

Finally, `assign` does nothing, so we're going to keep it empty. As a result, after first two lines `@variables` will be `{ a: 1, b: 2 }`. Let's take brief a look at how sum works:

1. `b` makes `method_missing` to put `:b` to the `@unknown_token`;
2. `with` reads the value from `@variables` using `@unknown_token` and stores it in a global variable called `@with`;
3. `a` makes `method_missing` to put `:a` to the `@unknown_token`;
4. `sum` reads the value from `@variables` using `@unknown_token`, sums it with the value from `@with` and prints the result.

Isn't it cool? It is, but this code not looks like something I'd like to maintain, because we have implicit dependencies between method call (e.g, `variable` method assumes that `method_missing` was called earlier). What happens if we try to execute something like `sum a`? It will return exception with a message that does not help user to fix the problem: `method_missing: can't modify frozen NilClass: nil`. Can we execute the line `variable a value 1`? Oh yes we can, even though it makes completely no sense!

You might have noticed that we're not going to process all possible phrases, and that's true 🙂 We're going to support only a small subset of them, but it's still going to be fun!

## Stack–based phrase processor

Our main goal for this section is to create explicit dependencies between methods and variables they use: we need to bring some _encapsulation_ in. Also, we need to help our user (i.e., _natural language programmer_) to understand why his code is not valid and how to fix it.

Let's try a different approach: the leftmost method call on each line (i.e., `assign` and `sum`) will try to execute everything on the right, while all other methods will just collect instructions somewhere. In other words, methods will _push_ instructions to some data structure one by one from the right to the left while `assign` will _pull_ them from the right to the left and perform the action. Do you know the name of the data structure? It's a _stack_!

> [Stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) is a data structure that contains a list of elements and has two operations: push to add element to the top and pull to get the element that was pushed last.

Here is the [implementation](https://gist.github.com/DmitryTsepelev/01702a27e86dd774d44998c3a3894dce#file-03_stack-rb):

```ruby
@variables = {}

Value = Struct.new(:value)
Token = Struct.new(:name)
Keyword = Struct.new(:type)

class Stack < Array
  def pop_if(expected_class)
    return pop if last.is_a?(expected_class)
    raise "Expected #{expected_class} but got #{last.class}"
  end

  def pop_if_keyword(keyword_type)
    pop_if(Keyword).tap do |keyword|
      unless keyword.type == keyword_type
        raise "Expected #{keyword_type} but got #{keyword.type}"
      end
    end
  end
end

@stack = Stack.new

def assign(*)
  @stack.pop_if_keyword(:variable)
  token = @stack.pop_if(Token)
  assignment = @stack.pop_if(Value)

  @variables[token.name] = assignment.value
end

def variable(*)
  @stack << Keyword.new(:variable)
end

def value(value)
  @stack << Value.new(value)
end

def method_missing(token, *args, &block)
  @stack << Token.new(token)
end

def sum(*)
  left = @stack.pop_if(Token)
  @stack.pop_if_keyword(:with)
  right = @stack.pop_if(Token)
  print @variables[left.name] + @variables[right.name]
end

def with(*)
  @stack << Keyword.new(:with)
end

# Program

assign variable a value 1
assign variable b value 2
sum a with b
```

First of all, we define three structs to represent possible objects of our language:

- `Value` holds our values to assign to variables;
- `Token` is something that was catch by `method_missing` (for now—only variable names);
- `Keyword` is something known we expect to see in our expressions.

`Value` and `Keyword` look very similar, but they represent different things: `Keyword` stores the name of the unknown method while `Value` is used for something that was _after_ the `value` method.

After that, we define our custom `Array` subclass to use as stack. There are two additional methods:

- `pop_if(expected_class)` checks if the top value is the object of passed class and returns it raising the error otherwise;
- `pop_if_keyword(keyword_type)` does almost the same, but accepts only `Keyword` instances with the specific type.

Then, our `variable`, `value`, `method_missing` and `with` methods do nothing except pushing the appropriate objects to the global variable `@stack`. For instance, the line `assign variable a value 1` will do the following (see the GIF below):

1. `value 1` adds `Value.new(1)` to the stack;
2. `a` adds `Token.new(:a)` to the stack;
3. `variable` adds `Keyword.new(:variable)` to the stack.

`assign` tries to pop data it expects from the stack and, if nothing was raised, registers variable the `@variables`:

```ruby
def assign(*)
  @stack.pop_if_keyword(:variable)
  token = @stack.pop_if(Token)
  assignment = @stack.pop_if(Value)

  @variables[token.name] = assignment.value
end
```

![stack execution](/assets/stack.gif)

I'll leave you a pleasure to figure out how `sum` works and focus on another problem: now we have the explicit connection between _command_ functions (`assign` and `sum`) and their data. However, defining them is still a lot of work; what if we could have some kind of DSL to define such commands? Fortunately, we're writing Ruby and we have a metaprogramming to help us!

### DSL for commands

Let's start with the whole [snippet](https://gist.github.com/DmitryTsepelev/01702a27e86dd774d44998c3a3894dce#file-04_dsl-rb) as usual (please note that I omitted `Stack` class as well as `variable`, `value`, `method_missing` and `with` methods, from the snippet—they didn't change):

```ruby
@variables = {}
@stack = Stack.new

# Command definition DSL

class Command
  attr_reader :execution_block

  def initialize(stack, variables)
    @stack = stack
    @variables = variables
    @expectations = []
  end

  def build(&block)
    self.tap { |command| command.instance_eval(&block) }
  end

  def args
    @expectations.each_with_object([]) do |expectation, args|
      if expectation.is_a?(Keyword)
        @stack.pop_if_keyword(expectation.type)
      else
        args << @stack.pop_if(expectation)
      end
    end
  end

  private

  def token
    @expectations << Token
  end

  def value
    @expectations << Value
  end

  def keyword(type)
    @expectations << Keyword.new(type)
  end

  def execute(&block)
    @execution_block = block
  end
end

def command(command_name, &block)
  command = Command.new(@stack, @variables).build(&block)

  define_method(command_name) do |*|
    command.execution_block.call(@variables, *command.args)
  end
end

# Command definitions

command(:assign) do
  keyword(:variable)
  token
  value

  execute do |variables, token, value|
    variables[token.name] = value.value
  end
end

command(:sum) do
  token
  keyword(:with)
  token

  execute do |variables, left, right|
    result = variables[left.name] + variables[right.name]
    print "#{result}\n"
  end
end

# Program

assign variable a value 1
assign variable b value 2
sum a with b
```

Let's start with the class that will hold our command data—it's called `Command`. Each command expects some objects on the stack, so we define a list of methods to register these _expectations_. Look at the `keyword` example:

```ruby
def keyword(type)
  @expectations << Keyword.new(type)
end
```

Also, there is a method to store the execution block:

```ruby
def execute(&block)
  @execution_block = block
end
```

When the time comes to execute this command, we match command expectations with stack to prepare aruments to pass to the `@execution_block`. `pop_if` and `pop_if_keyword` take care about cases when stack does not contain the expected value:

```ruby
def args
  @expectations.each_with_object([]) do |expectation, args|
    if expectation.is_a?(Keyword)
      @stack.pop_if_keyword(expectation.type)
    else
      args << @stack.pop_if(expectation)
    end
  end
end
```

Now let's make this class to work as a part of our DSL:

```ruby
def build(&block)
  self.tap { |command| command.instance_eval(&block) }
end
```

When someone passes a block to the `build` method, this block will be executed in the _context_ of this class using [instance_eval](https://apidock.com/ruby/Object/instance_eval). As a result, all instance methods become available inside the block.

The `command` function initializes the `Command` instance, defines a new method with the body that executes the block from the commands with args we've built in the `#args` method:


```ruby
def command(command_name, &block)
  command = Command.new(@stack, @variables).build(&block)

  define_method(command_name) do |*|
    command.execution_block.call(@variables, *command.args)
  end
end
```

Finally, we need to define our commands using the DSL we prepared:

```ruby
command(:assign) do
  keyword :variable
  token
  value

  execute do |variables, token, value|
    variables[token.name] = value.value
  end
end
```

The command called `assign` expects a keyword with a `variable` type, some token and some value. Token and value will be passed to the execute block, which performs the actual work of assigning the variable.

Let's prove that our DSL can help us build additional feature for our language. For instance, we can easily add the deduct command:

```ruby
def from(*)
  @stack << Keyword.new(:from)
end

command(:deduct) do
  token
  keyword :from
  token

  execute do |variables, left, right|
    result = variables[right.name] - variables[left.name]
    print "#{result}\n"
  end
end

assign variable x value 12
assign variable y value 5
deduct y from x
```

Looking good! The next problem we need to tackle is `method_missing` defined on the top level (my Ruby interpreter kept yelling on me for that 🙂). The issue is that this approach makes writing code hard: for instance, when you accidentally call method on the `nil` you go to the `method_missing` rather than get the error itself. It would be nice to encapsulate it somehow, so let's introduce a container for that and call it a Virtual Machine.

### Building our small Virtual Machine

Please meet our [very virst](https://gist.github.com/DmitryTsepelev/01702a27e86dd774d44998c3a3894dce#file-05_vm-rb) Virtual Machine, that will execute the program written in the "natural" language:

```ruby
class VM
  attr_reader :variables, :stack

  def initialize
    @variables = {}
    @stack = Stack.new
  end

  def run(&block)
    instance_eval(&block)
  end

  class << self
    def command(command_name, &block)
      define_method(command_name) { |*| Command.build(&block).run(self) }
    end

    def run(&block)
      new.run(&block)
    end
  end

  # Commands: same as before

  # command(:assign)
  # command(:sum)

  # Primitives: same as before

  # def variable(*)
  # def value(value)
  # def method_missing(token, *args, &block)
  # def with(*)
  # def from(*)
end

# Program

VM.run do
  assign variable a value 1
  assign variable b value 2
  sum a with b
end

```

The main change is that the "natural" code will be executed inside the `VM.run do` block. In order to make it work we use the same trick as we did for `Command`. All methods like `variable` and command definitions are moved to the `VM` class and made available inside the block using `instance_eval`:

```ruby
def run(&block)
  instance_eval(&block)
end
```

Finally, `@stack` and `@variables` are not global anymore and stored inside the `VM` instanсe. We could call it a day, but what if we want our `VM` to execute code in different languages, which are also configurable via the special DSL? Here is how our current language can be represented:

```ruby
lang = Lang.define do
  command :assign do
    keyword :variable
    token
    value

    execute { |vm, token, value| vm.assign_variable(token, value) }
  end

  command :sum do
    token
    keyword :with
    token

    execute do |vm, left, right|
      result = vm.read_variable(left) + vm.read_variable(right)
      print "#{result}\n"
    end
  end
end

VM.run(lang) do
  assign variable a value 1
  assign variable b value 2
  sum a with b
end
```

The main benefit of this approach is that our "primitives" (`variable`, `with`, etc.) will be defined dynamically based on the syntax of the language.

### Building the language in the runtime

As usual, let's start with the whole [snippet](https://gist.github.com/DmitryTsepelev/01702a27e86dd774d44998c3a3894dce#file-06_lang-rb):

```ruby
class Lang
  def self.define(&block)
    new.tap { |lang| lang.instance_eval(&block) }
  end

  def command(command_name, &block)
    command = Command.build(command_name, &block)
    register_keywords(command)
    commands[command_name] = command
  end

  def keywords
    @keywords ||= []
  end

  def commands
    @commands ||= {}
  end

  private

  def register_keywords(command)
    command.expectations
      .filter { |expectation| expectation.is_a?(Keyword) }
      .reject { |keyword| keywords.include?(keyword.type) }
      .each { |keyword| keywords << keyword.type }
  end
end

class VM
  def self.run(lang, &block)
    lang.commands.each do |command_name, command|
      define_method(command_name) { |*| command.run(self) }
    end

    new(lang).run(&block)
  end

  attr_reader :variables, :stack

  def initialize(lang)
    @lang = lang
    @variables = {}
    @stack = Stack.new
  end

  def run(&block)
    instance_eval(&block)
  end

  def assign_variable(token, value)
    @variables[token.name] = value.value
  end

  def read_variable(token)
    @variables[token.name]
  end

  def value(value)
    @stack << Value.new(value)
  end

  def method_missing(unknown, *args, &block)
    klass = @lang.keywords.include?(unknown) ? Keyword : Token
    @stack << klass.new(unknown)
  end
end

# Language definition

lang = Lang.define do
  command :assign do
    keyword :variable
    token
    value

    execute { |vm, token, value| vm.assign_variable(token, value) }
  end

  command :sum do
    token
    keyword :with
    token

    execute do |vm, left, right|
      result = vm.read_variable(left) + vm.read_variable(right)
      print "#{result}\n"
    end
  end
end

# Program

VM.run(lang) do
  assign variable a value 1
  assign variable b value 2
  sum a with b
end
```

`Lang` class stores the definition of our new language. Please note, that we use the standard trick with `instance_eval` to use this class as a part of the DSL:

```ruby
def self.define(&block)
  new.tap { |lang| lang.instance_eval(&block) }
end
```

The only method we are going to use inside the block that will be passed to `define` is `command`. It accepts the name of the command to define and a block. Both arguments are passed to the command builder:

```ruby
def command(command_name, &block)
  command = Command.build(command_name, &block)
  register_keywords(command)
  commands[command_name] = command
end
```

When command is built, we need to store it in the list of commands and add keywords that are used inside the command to the list of keywords used in the language:

```ruby
def register_keywords(command)
  command.expectations
    .filter { |expectation| expectation.is_a?(Keyword) }
    .reject { |keyword| keywords.include?(keyword.type) }
    .each { |keyword| keywords << keyword.type }
end
```

Now we can define our language, that will accept two keywords (`variable` and `with`) and execute two commands (`assign` and `sum`):

```ruby
lang = Lang.define do
  command :assign do
    keyword :variable
    token
    value

    execute { |vm, token, value| vm.assign_variable(token, value) }
  end

  command :sum do
    token
    keyword :with
    token

    execute do |vm, left, right|
      result = vm.read_variable(left) + vm.read_variable(right)
      print "#{result}\n"
    end
  end
end
```

Now we need to make changes in the `VM` class. Language just became the separate object, so all the methods corresponding to commands are gone. Let's change the `run` method to define them based on the language we run:

```ruby
def self.run(lang, &block)
  lang.commands.each do |command_name, command|
    define_method(command_name) { |*| command.run(self) }
  end

  new(lang).run(&block)
end
```

As a result, `assign` and `sum` methods will be added to the `VM` class. I know that it's not the ideal implementation, since we never remove these methods from the `VM` class, but this implementation is good enough for our purposes and I want to make things less complex 🙂

Another change is that we add some "low–level" operations to our VM, which can be used inside our commands:

```ruby
def assign_variable(token, value)
  @variables[token.name] = value.value
end

def read_variable(token)
  @variables[token.name]
end

def value(value)
  @stack << Value.new(value)
end
```

Finally, let's change our `method_missing` implementation to handle both tokens and keywords. We have a list of keywords defined in the `Lang` instance, so we can easily make a difference between these two:

```ruby
def method_missing(unknown, *args, &block)
  klass = @lang.keywords.include?(unknown) ? Keyword : Token
  @stack << klass.new(unknown)
end
```

Now we are all set! Let's define _another_ language with a different syntax and make sure it works:

```ruby
another_lang = Lang.define do
  command(:set) do
    keyword :variable
    token
    keyword :to
    value

    execute { |vm, token, value| vm.assign_variable(token, value) }
  end

  command(:access) do
    keyword :variable
    token

    execute do |vm, token|
      result = vm.read_variable(token)
      print "#{result}\n"
    end
  end
end

VM.run(another_lang) do
  set variable a to value 42
  access variable a # => 42
end
```

### Travel planning

You might think that assigning and reading variables is all that we can do with this code. Let me prove you wrong: I want to create a language that can help us with travel planning. I want the following program to tell me that the travel from London to Glasgow takes 22 hours:

```ruby
VM.run(lang) do
  route from london to glasgow takes 22
  route from paris to prague takes 12
  how long will it take to get from london to glasgow
end
```

> I'd really like to do `route from london to glasgow takes 22 hours` but we cannot do that because it will be invalid Ruby 😞

Can we make it work? Sure! The only problem is that, we cannot use `takes` as a method that consumes a value after it, because we hardcoded a method `value` to do that.

Let's change our command to accept the argument to the value method. It will be used as a method name for assigning values (the full snippet is [here](https://gist.github.com/DmitryTsepelev/01702a27e86dd774d44998c3a3894dce#file-07_graph-rb)):

```ruby
class Command
  attr_reader :execution_block, :value_method_names

  # ...

  def value_method_names
    @value_method_names ||= []
  end

  private

  def value(method_name)
    value_method_names << method_name
    expectations << Value
  end

  # ...
end
```

After that, we need to change our `Lang` class to define corresponding methods on the `VM` class:

```ruby
class VM
  def self.run(lang, &block)
    lang.commands.each do |command_name, command|
      define_method(command_name) { |*| command.run(self) }

      command.value_method_names.each do |value_method_name|
        define_method(value_method_name) do |value|
          @stack << Value.new(value)
        end
      end
    end

    new(lang).run(&block)
  end

  # no changes, but `value` method is removed
end
```

The last thing to do is to define our new language:

```ruby
lang = Lang.define do
  command :route do
    keyword :from
    token
    keyword :to
    token
    value :takes

    execute do |vm, city1, city2, distance|
      distances = vm.read_variable(:distances) || {}
      distances[[city1, city2]] = distance
      vm.assign_variable(:distances, Value.new(distances))
    end
  end

  command :how do
    keyword :long
    keyword :will
    keyword :it
    keyword :take
    keyword :to
    keyword :get
    keyword :from
    token
    keyword :to
    token

    execute do |vm, city1, city2|
      distances = vm.read_variable(:distances) || {}
      distance = distances[[city1, city2]].value
      puts "Travel from #{city1.name} to #{city2.name} takes #{distance} hours"
    end
  end
end
```

As you see, we use `VM#read_variable` and `VM#assign_variable` as a "low–level API" to store and manipulate distances. To make things more fancy, we could implement [Dijkstra](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) to find a shortest route in the graph, but this is a bit out of our current scope 🙂

---

That's all for today, I hope you enjoyed the ride! We learned how to use metaprogramming to build complex DSLs in Ruby. Obviously, this is not a real natural language processing and languages we can built using this DSL are very limited, but I had a lot of fun preparing these examples so decided to share it with the world.

> A gemified version of this approach is available [here](https://github.com/DmitryTsepelev/natural_dsl).

If you want to get your hands dirty—I collected a couple of ideas I decided to not implement:

1. Allow assign the value of one variable to another (e.g., `assign variable b value a`)

2. Make command DSL less verbose:
```ruby
command(:assign, keyword(:variable).token.value) do |vm, token, value|
  vm.assign_variable(token, value)
end
```

Hint: you could use [CPS](https://en.wikipedia.org/wiki/Continuation-passing_style) to implement `keyword(:variable).token.value`.
