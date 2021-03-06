# Details on Traject Indexing: from custom logic to Macros

Traject macros are a way of providing re-usable index mapping rules. Before we discuss how they work, we need to remind ourselves of the basic/direct Traject `to_field` indexing method.

## How direct indexing logic works

Here's the simplest possible direct Traject mapping logic, duplicating the effects of the `literal` macro:

~~~ruby
to_field("title") do |record, accumulator, context|
  accumulator << "FIXED LITERAL"
end
~~~

That `do` is just ruby `block` syntax, whereby we can pass a block of ruby code as an argument to to a ruby method. We pass a block taking three arguments, labeled `record`, `accumulator`, and `context`, to the `to_field` method. The third 'context' object is optional, you can define it in your block or not, depending on if you want to use it. 

The block is then stored by the Traject::Indexer, and called for each record indexed, with three arguments provided. 

### record argument

The record that gets passed to your block is a MARC::Record object (or, theoretically, any object that gets returned by a traject Reader). Your logic will usually examine the record to calculate the desired output. 

### accumulator argument

The accumulator argument is an Array. At the end of your custom code, the accumulator Array should hold the output you want send off to the field specified in `to_field`. 

The accumulator is a reference to a ruby Array, and you need to **modify** that Array, manipulating it in place with Array methods that mutate the array, like `concat`, `<<`, `map!` or even `replace`. 

You can't simply assign the accumulator variable to a different Array; you need to modify the Array *in place*. 

    # Won't work, assigning variable
    to_field('foo') do |rec, acc| 
      acc = ["some constant"] } # WRONG!
    end

    # Won't work, assigning variable
    to_field('foo') do |rec, acc|
      acc << 'bill'
      acc << 'dueber'
      acc = acc.map{|str| str.upcase}
    end   # WRONG! WRONG! WRONG! WRONG! WRONG!  


    # Instead, do, modify array in place
    to_field('foo') {|rec, acc| acc << "some constant" }
    to_field('foo') do |rec, acc|
      acc << 'bill'
      acc << 'dueber'
      acc.map!{|str| str.upcase} # NOTE: "map!" not "map"
    end

### context argument

The third optional argument is a [Traject::Indexer::Context](./lib/traject/indexer/context.rb)  ([rdoc](http://rdoc.info/github/traject/traject/Traject/Indexer/Context)) object. Most of the time you don't need it, but you can use it for some sophisticated functionality.  These are some useful methods available:

* `context.clipboard` A hash into which you can stuff values that you want to pass from one indexing step to another. For example, if you go through a bunch of work to query a database and get a result you'll need more than once, stick the results somewhere in the clipboard. This clipboard is record-specific, and won't persist between records. 
* `context.position` The position of the record in the input file (e.g., was it the first record, second, etc.). Useful for error reporting.
* `context.output_hash` A hash mapping the field names (generally defined in `to_field` calls) to an array of values to be sent to the writer associated with that field. This allows you to modify what goes to the writer without going through a `to_field` call -- you can just set `context.output_hash['myfield'] = ['my', 'values']` and you're set. See below for more examples.
* `context.skip!(msg)` An assertion that this record should be ignored. No more indexing steps will be called, no results will be sent to the writer, and a `debug`-level log message will be written stating that the record was skipped.


## Gotcha: Use closures to make your code more efficient

A _closure_ is a computer-science term that means "a piece of code
that remembers all the variables that were in scope when it was
created." In ruby, lambdas and blocks are closures. Method definitions
are not, which most of us have run across much to our chagrin.

Within the context of `traject`, this means you can define a variable
outside of a `to_field` or `each_record` block and it will be avaiable
inside those blocks. And you only have to define it once.

That's useful to do for any object that is even a bit expensive
to create -- we can maximize the performance of our traject
indexing by creating those objects once outside the block,
instead of inside the block where it will be created
once per-record (every time the block is executed):

Compare:

```ruby
# Create the transformer for every single record
to_field 'normalized_title' do |rec, acc|
  transformer = My::Custom::Format::Transformer.new # Oh no! I'm doing this for each of my 10M records!
  acc << transformer.transform(rec['245'].value)
end

# Create the transformer exactly once
transformer = My::Custom::Format::Transformer.new # Ahhh. Do it once.
to_field 'normalized_title' do |rec, acc|
  acc << transformer.transform(rec['245'].value)
end
```

Certain built-in traject calls have been optimized to be high performance
so it's safe to do them inside 'inner loop' blocks. That includes `Traject::TranslationMap.new` and `Traject::MarcExtractor.cached("xxx")` 
(NOTE: #cached rather than #new there)


## From block to lambda

In the ruby language, in addition to creating a code block as an argument
to a method with `do |args| ... end` or `{|arg| ...  }`, we can also create
a code block to hold in a variable, with the `lambda` keyword:

    always_output_foo = lambda do |record, accumulator|
      accumulator << "FOO"
    end

In traject, `to_field` is written so that, as a convenience, it can take a lambda expression stored in a variable as an alternative to a block:

    to_field("always_has_foo"), always_output_foo

Why is this a convenience? Well, ordinarily it's not something we
need, but in fact it's what allows traject 'macros' to be re-useable
code templates. 


## Macros

A Traject macro is a way to automatically create indexing rules via re-usable "templates".

Traject macros are methods that return ruby lambda/proc objects, possibly creating them based on parameters passed in. 

For example, here is the implementation of the  `literal` method/macro:

~~~ruby
def literal(value)
  return lambda do |record, accumulator, context|
     # because a lambda is a closure, we can define it in terms
     # of the 'value' from the scope it's defined in!
     accumulator << value
  end
end
to_field("fieldname"), literal("my_fav_literal")
~~~

So a Traject macro is a method that may have parameters and, based on those parameters, returns a lambda; the lambda is then passed to the `to_field` indexing method, or similar methods.

How do you make these methods available to the traject indexer?

Define it in a module:

~~~ruby
# in a file literal_macro.rb
module LiteralMacro
  def literal(value)
    return lambda do |record, accumulator, context|
       # because a lambda is a closure, we can define it in terms
       # of the 'value' from the scope it's defined in!
       accumulator << value
    end
  end
end
~~~

And then use ordinary ruby `require` and `extend` to add it to the current Indexer file, by simply including this
in one of your config files:

~~~
require `literal_macro.rb`
extend LiteralMacro

to_field("fieldname"), literal("my_fav_literal")
~~~

That's it.  You can use the traject command line `-I` option to set the ruby load path, so your file will be findable via `require`.  Or you can distribute it in a gem, and use straight rubygems and the `gem` command in your configuration file, or Bundler with traject command-line `-g` option.

## Using a lambda _and_ a block

Traject macros (such as `extract_marc`) create and return a lambda. If
you include a lambda _and_ a block on a `to_field` call, the block
gets the accumulator as it was filled in by the former.

```ruby
# Get the titles and lowercase them
to_field 'lc_title', extract_marc('245') do |rec, acc, context|
  acc.map!{|title| title.downcase}
end

# Build my own lambda and use it
mylam = lambda {|rec, acc|  acc << 'one'} # just add a constant
to_field('foo'), mylam do |rec, acc, context|
  acc << 'two'
end #=> context.output_hash['foo'] == ['one', 'two']

# You might also want to do something like this
to_field('foo'), macro_returning_dup_values do |rec, acc|
  acc.uniq!
end
```

## Maniuplating `context.output_hash` directly

If you ask for the context argument, a [Traject::Indexer::Context](./lib/traject/indexer/context.rb) ([rdoc](http://rdoc.info/gems/traject/Traject/Indexer/Context)), you have access to `context.output_hash`, which is
the hash of already transformed output that will be sent to Solr (or any other Writer).

You can examine `context.output_hash` to see any already transformed output and use it as the source for new output. 

You can *write* to `context.output_hash` directly, which can be useful for computations that affect more than one output field at once. 

**Note**: Make sure you always assign an _Array_ to each `context.output_hash` value, e.g., `context.output_hash['foo']`, not a single value!

```ruby

# Wrong - do NOT assign a value of anything other than an Array
context.output_hash['fieldname'] = 'fuzzy_wuzzies'

# Correct
context.output_hash['fieldname'] = ['fuzzy_wuzzies']
```



## each_record

`each_record` is similar to `to_field` in that it defines logic executed for each record.  It differs from `to_field` because the output of `each_record` is not associated with a specific output field.

Thus, `each_record` blocks have no `accumulator` argument: instead they either take a single `record` argument; or both a `record` and a `context`. 

`each_record` is useful for logging or notifiying, computing intermediate
results, or writing to more than one field at once. 

~~~ruby
each_record do |record, context|
  if is_it_bad?(record)
    context.skip!("Skipping bad record")
  else
    context.clipboard[:expensive_result] = calculate_expensive_thing(record)
  end
end

each_record do |record, context|
  (val1, val2) = calculate_two_things_from(record)

  context.output_hash["first_field"] ||= []
  context.output_hash["first_field"] << val1

  context.output_hash["second_field"] ||= []
  context.output_hash["second_field"] << val2
end
~~~

traject doesn't come with any macros written for use with `each_record`, but they could be created:  such macros would be methods that return a lambda given the appropriate args from `each_record`. 

## More tips and gotchas about indexing steps

* **All your `to_field` and `each_record` steps are run _in the order in which they were initially evaluated_**. That means that the order you call your config files can potentially make a difference if you're screwing around stuffing stuff into the context clipboard or whatnot.

* **`to_field` can be called multiple times on the same field name.** If you call the same field name multiple times, all the values will be sent to the writer.

* **Once you call `context.skip!(msg)` no more index steps will be run for that record**. So if you have any cleanup code, you'll need to make sure to call it yourself.

* **By default, `traject` indexing runs multi-threaded**. In the current implementation, the indexing steps for one record are *not* split across threads, but different records can be processed simultaneously by more than one thread. That means you need to make sure your code is thread-safe (or always set `processing_thread_pool` to 0). 
