+++
date = "2014-09-18"
title = "Bulk unit testing with Elixir macros"
description = "A clean and scalable way to leverage Elixir's macros when unit testing functions with a large number of input/output cases."
keywords = ["elixir", "macro", "unit testing", "test", "bulk"]
categories = ["elixir"]
+++
There's plenty written on how to choose the best targets for unit testing, how far to take it, best practices for layout and test structure, and much more - so I won't discuss that.

I also won't touch on when to use macros in general - that's covered quite well in the [Macros](http://elixir-lang.org/getting_started/meta/2.html) part of the [Elixir Getting Started guide](http://elixir-lang.org/getting_started/1.html).

This is about what you could do *after* you've chosen the scope of your testing, but even [ExUnit](http://elixir-lang.org/docs/stable/ex_unit/) macros like `test` and `assert` become too verbose through sheer weight of input/output scenarios mapped to individual tests.

Here's the approach I found is the neatest way to structure those tests.

---

### Input

We'll list the inputs and their corresponding expected outputs in a text file, let's say `test/input.txt` to keep it short, which looks like this:

``` no-highlight
MyModule.my_function/1 a scenario || an input string || [an: :expected, output: <<1, 2, 3>>]
MyModule.my_function/1 another scenario || another input string || [another: :expected, output: <<4, 5, 6>>]
[ ... ]
```

Each line follows this format:

`test case name <separator> input <separator> expected output`

### The tests

Directly in the `test/my_module_test.exs` file you'd have this snippet, outside of any functions:

``` elixir
for line <- File.stream!(Path.join([__DIR__, "input.txt"]), [], :line) do
  [name, input, expected] =
    line |> String.split("||") |> Enum.map(&String.strip(&1))
  test name do
    {expected, []} = Code.eval_string(unquote(expected))
    result = MyModule.my_function(unquote(input))
    assert ^expected = result
  end
end
```

For each line in the input text file:

1. Split the line into its components: `name`, `input`, `expected`
2. Create a new test case named `name`, which:
  * Converts `expected` into Elixir terms using `Code.eval_string/1`.
  * Runs the function being tested, passing it the argument `input`.
  * Compares the actual result and the expected one.

`Code.eval_string/1` is used to process complex values, like the keyword list expected output. Note that it's not needed for the input because that's a simple string in this example.

In this example if we compare the expected output with the actual result without `Code.eval_string/1` we'd be comparing a keyword list with a string representation of a keyword list, and it would always fail!

---

### Benefits

**Concise!** The main reason we're going through this trouble. This saves a fair bit of time and test clutter compared to manually writing out each input/output scenario as a test.

**Individual tests with their own names.** This is why one of the fields in the input text file is the test name. You get precise feedback about which cases are failing.

**Tests fail independently.** Because each line translates to a separate test, each one runs independently. If we were looping through the lines and running them with a private helper function, it would stop on the first failure!

**Accurate test count.** You'll get a better idea of your test count and coverage when each input/output scenario is its own test.

**Easily scales to multiple functions.** If testing multiple functions, their individual test data is abstracted away in separate input text files and the `test.exs` stays compact.

Feel free to drop a message below if you have a better solution, or would like to add anything!