---
title: "Extending Ruby with Rust"
date: 2019-10-31T11:46:44+05:30
draft: false
tags:
  - ruby
  - rust
---

Ruby is a language I adore, prototyping stuff is blazingly fast. It is as if you are writing pseudocode which you can simply run. By all means, this is not surprising as it is a language that is focused on making the programmers happy. This might seem all sunshine and butterflies but Ruby is relatively slow in comparison with other languages that are optimized for speed and concurrency. Sadly, this is the cost we have to put up with when we try to be happy.

## Bothered with slow Ruby code?
 Honestly, I admit that so far I haven't been bottlenecked by Ruby for all my development purposes but if it all I do, need to know what all options do I have to pivot from it. There are high hopes for [Ruby 3](https://youtu.be/WZu-WVzbEOA) which is in the pipeline and is promised to be 3 times as fast and with better concurrency support. But still, if speed is what you are concerned with, the interpreted language will always be a bottleneck.
 
One thing to do is to rewrite the program using a language that addresses the problem you are facing. This would be a huge task to undertake and the business goals might not adhere to the time/cost to be spent on it. It would be a whole lot better if we could stick with Ruby goodness and then just rewrite the slow parts. If that approach can be taken we can consider writing a separate service altogether or implementing a native extension to talk with our Ruby code.

We can write a native extension or better yet make use of [FFI](https://github.com/ffi/ffi) to bridge the gap between interpreted Ruby and the compiled language we opt for. The gem helps us to abstract out a lot of glue code [otherwise](https://www.rubyguides.com/2018/03/write-ruby-c-extension/). But the question now is what language do we opt for? There are many excellent extensions written with C and it'd be a good choice for an experienced C programmer; Not for me though. Having been spoonfed and spoilt using Ruby, putting me in charge of manually managing memory would be too much of a responsibility to take up. I'm used to working above system-level abstractions and age-old memory safety issues with *memory leaks*, *use after free*, *null pointer*, *buffer overreads/overrites*, etc add on to the pressure of messing up my program. 

## Give it a Rust, will you?

Happens to be that there is a language out there which is **as fast as C** and guarantees **memory safety** as well as **thread-safety**. Takes a little bit of pressure off me for sure. To top over that the language doesn't involve you to manually manage memory and has no garbage collector to slow you down to help enable this. Interesting right? This is why I'm starting to love Rust. I still have a hard time to start getting along with its strict and intelligent compiler. But the errors put out during compile time are useful to understand what you are messing up with. Eventually, it would become like pair programming with the compiler.

> *"Traditionally, this realm of programming is seen as arcane, accessible only to a select few who have devoted the necessary years learning to avoid its infamous pitfalls. And even those who practice it do so with caution, lest their code is open to exploits, crashes, or corruption. Rust breaks down these barriers by eliminating the old pitfalls and providing a friendly, polished set of tools to help you along the way."* **- Excerpt from: [Rust docs](https://doc.rust-lang.org/book/foreword.html) on system-level programming.**

## Show me some code already!

Let us consider a fun program that converts a fairly large book: [The Adventures of Sherlock Holmes](https://norvig.com/big.txt) into Pig Latin.
The rules to convert onto Pig Latin are quite simple and here is our Ruby solution for it.

```ruby
# The first consonant of each word is moved to the end of the word and “ay” is
# added, so “first” becomes “irst-fay.” Words that start with a vowel have “hay”
# added to the end instead (“apple” becomes “apple-hay”).
    
class PigLatinConverter
  def initialize(source_filename, target_filename)
    @source = source_filename
    @target = target_filename
  end

  def convert
    File.foreach(@source) do |line|
      piglatin_sentence = convert_piglatin(line)
      File.write(@target, piglatin_sentence + "\n", mode: 'w')
    end
  end

  def convert_piglatin(sentence)
    sentence.split.map do |word|
      piglatinize(word)
    end.join(' ')
  end

  def piglatinize(word)
    c = word[0]

    if  c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u' ||
        c == 'A' || c == 'E' || c == 'I' || c == 'O' || c == 'U'
      "#{word}-hay"
    else
      "#{word[1..]}-#{word[0]}ay"
    end
  end
end



PigLatinConverter.new('big.txt', 'piglatinized.txt').convert
```

Checking how much time it does take to run on my machine (wiz. MacBook Air 2017 with 1.8Ghz core i5 & 8Gb memory).

```
❯ time ruby pig_latin_converter.rb 
ruby pig_latin_converter.rb  4.61s user 9.21s system 93% cpu 14.725 total
```
Looks like it takes about 14s to complete the conversion. Alright then, time to migrate on to the promised Rust land. Let's see how much more performant can we get by bridging onto Rust and also check how much of a chore it can be.

## Entering the arcane realm with Ruby-FFI

Let us start by creating a new rust package with the help of cargo.

```
❯ cargo new piglatin --lib
Created library `piglatin` package
```

Time to churn out some Rust code. Although I'm not much a fan of TDD and sticks with BDD, the former has been helpful for me as a beginner in Rust.
So now we replace the test in generated `lib.rs` file with our intended function.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_piglatinizes() {
        let sentence = "hello darkness my old friend";
        assert_eq!(
            "ello-hay arkness-day y-may old-hay riend-fay", 
            piglatinize(sentence)
        );
    }
}
```

Running `cargo test` gives us:

```
error[E0425]: cannot find function `piglatinize` in this scope
  --> src/lib.rs:6:59
   |
 6 |         assert_eq!("ello-hay arkness-day y-may old-hay riend-fay", piglatinize(sentence));
   |                                                                    ^^^^^^^^^^^ not found in this scope
```

Time to define such a function. It's just the matter of converting the logic from the earlier Ruby function.


```rust
fn piglatinize(line: &str) -> String {
    let mut res = Vec::new();

    for word in line.split_whitespace() {
        if let Some(c) = word.chars().next() {
            if c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u' ||
               c == 'A' || c == 'E' || c == 'I' || c == 'O' || c == 'U' {
                res.push(format!("{}-hay", word));
            } else {
                res.push(format!("{}-{}ay", &word[1..], &c));
            }
        }
    }

    res.join(" ")
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn it_piglatinizes() {
        let sentence = "hello darkness my old friend";
        assert_eq!(
            "ello-hay arkness-day y-may old-hay riend-fay", 
            piglatinize(sentence)
        );
    }
}
```

Let's try running `cargo test` again.

```
running 1 test
test tests::it_piglatinizes ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Sweet! What we need next is a function that we can probably attach onto our `PigLatinConverter` class and is capable of reading, converting and then writing the piglatinized text.
Here goes our implementation for it.

```rust
use std::fs::File;
use std::io::{Write, BufRead, BufReader, LineWriter};

....
....

fn convert_piglatin(src_filename: &str, dest_filename: &str) {
    let src_file = File::open(src_filename).unwrap();
    let reader = BufReader::new(src_file);

    let dest_file = File::create(dest_filename).unwrap();
    let mut writer = LineWriter::new(dest_file);

    // Read the file line by line using the lines() iterator from std::io::BufRead.
    for (_index, line) in reader.lines().enumerate() {
        let line = line.unwrap();
        let mut piglatinized_line = piglatinize(&line[..]);
        piglatinized_line.push_str("\n");
        writer.write_all(piglatinized_line.as_bytes()).unwrap();
    }

    // we have to flush or drop the `LineWriter` to finish writing.
    writer.flush().unwrap();
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::prelude::*;

    ....
    ....

    #[test]
    fn it_creates_piglatinized_file(){
        let mut file = File::create("test_file.txt").unwrap();
        file.write_all(b"I've come to talk with you again").unwrap();

        convert_piglatin("test_file.txt", "test_converted.txt");

        let mut converted_file = File::open("test_converted.txt").unwrap();
        let mut contents = String::new();
        converted_file.read_to_string(&mut contents).unwrap();
        assert_eq!(
            contents,
            "I\'ve-hay ome-cay o-tay alk-tay ith-way ou-yay again-hay\n"
        );
    }
}
``` 
Creating a test file in b/w the test to get this working surely is not the idiomatic way to do this. There would be fixture support to do that. But we can avoid such conventions for now and get this working.

Running `cargo test` again results in:

```
running 2 tests
test tests::it_piglatinizes ... ok
test tests::it_creates_piglatinized_file ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Nice! All that there is left to do is to interface with our ruby code. Checking the [examples](https://github.com/ffi/ffi/wiki/Examples) from ruby-ffi shows us that we need to do the following on our code.

```ruby
require 'ffi'

module PigLatin
  extend FFI::Library
  ffi_lib 'piglatin/target/release/libpiglatin.' + FFI::Platform::LIBSUFFIX
  attach_function :convert_piglatin, [:string, :string], :void
end

class PigLatinConverter
  include PigLatin

  def initialize(source_filename, target_filename)
    @source = source_filename
    @target = target_filename
  end

  def convert
    convert_piglatin(@source, @target)
  end
end

PigLatinConverter.new('big.txt', 'piglatinized.txt').convert
```

`FFI::Platform::LIBSUFFIX` will provide the suffix to the dynamic library adopted by your operating system. This would be `.dylib` for Mac, `.dll` for Windows and `.so` for Linux. In my case I would need to build my cargo library as a dylib inorder to be linked with the ruby runtime. For that I would have to update `Cargo.toml` file as:

```toml
[lib]
name = "piglatin"
crate-type = ["dylib"]
```

Time to build the library with `cargo build --release` and go about running our ruby program to see what happens.

```
 ❯ ruby pig_latin_converter.rb
Traceback (most recent call last):
        2: from pig_latin_converter.rb:3:in `<main>'
        1: from pig_latin_converter.rb:6:in `<module:PigLatin>'
/Users/akshaysasidharan/.rvm/gems/ruby-2.6.3/gems/ffi-1.11.1/lib/ffi/library.rb:273:in `attach_function': Function 'convert_piglatin' not found in [piglatin/target/release/libpiglatin.dylib] (FFI::NotFoundError)
```
Too bad, looks like we did mess up something. Our ruby runtime is not able to attach the specified function. After a little bit of googling, came to find out that we have to provide linkage to the functions defined in Rust using [extern](https://doc.rust-lang.org/std/keyword.extern.html).

The modified rust function will look like this:

```rust

#[no_mangle]
pub extern fn convert_piglatin(src_filename: &str, dest_filename: &str) {
    ....
}
```

Back rebuilding our shared executable with `cargo build --release` and running our ruby script.

```
 ❯ ruby pig_latin_converter.rb
ruby(89787,0x10be925c0) malloc: can't allocate region
*** mach_vm_map(size=140496861253632) failed (error code=3)
ruby(89787,0x10be925c0) malloc: *** set a breakpoint in malloc_error_break to debug
memory allocation of 140496861252417 bytes failed[1]    89787 abort      ruby pig_latin_converter.rb

```
Say hello to the arcane side. I had no clue what went wrong here, again after a bit of googling around, I came across these two stackoverflow answers ([this](https://stackoverflow.com/questions/30440068/segmentation-fault-when-calling-a-rust-lib-with-ruby-ffi) and [this](https://stackoverflow.com/questions/24145823/how-do-i-convert-a-c-string-into-a-rust-string-and-back-via-ffi)) which potentially listed out the problem I was facing with. 

TLDR; The implementation of strings are different in both of the languages. The Ruby string which is being borrowed by Rust needs to be considered as `*const ::libc::c_char` type. We can construct a `&str` from `*const c_char` using an *unsafe* `CStr::from_ptr` static method.

Therefore now we need to add crate `libc` as a dependency in `Cargo.toml`.

```toml
[dependencies]
libc = "0.2"
```

The updated Rust program goes here:

```rust
use libc;
use std::ffi::CStr;
use std::fs::File;
use std::io::{Write, BufRead, BufReader, LineWriter};

fn piglatinize(line: &str) -> String {
    let mut res = Vec::new();

    for word in line.split_whitespace() {
        if let Some(c) = word.chars().next() {
            if c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u' ||
               c == 'A' || c == 'E' || c == 'I' || c == 'O' || c == 'U' {
                res.push(format!("{}-hay", word));
            } else {
                res.push(format!("{}-{}ay", &word[1..], &c));
            }
        }
    }

    res.join(" ")
}

type RubyString = *const libc::c_char;

fn ruby_string_to_ref_str<'a>(r_string: RubyString) -> &'a str {
  unsafe { CStr::from_ptr(r_string) }.to_str().unwrap()
}

#[no_mangle]
pub extern fn convert_piglatin(src_filename: RubyString, dest_filename: RubyString) {
    let src_filename = ruby_string_to_ref_str(src_filename);
    let dest_filename = ruby_string_to_ref_str(dest_filename);

    let src_file = File::open(src_filename).unwrap();
    let reader = BufReader::new(src_file);

    let dest_file = File::create(dest_filename).unwrap();
    let mut writer = LineWriter::new(dest_file);

    // Read the file line by line using the lines() iterator from std::io::BufRead.
    for (_index, line) in reader.lines().enumerate() {
        let line = line.unwrap();
        let mut piglatinized_line = piglatinize(&line[..]);
        piglatinized_line.push_str("\n");
        writer.write_all(piglatinized_line.as_bytes()).unwrap();
    }

    // we have to flush or drop the `LineWriter` to finish writing.
    writer.flush().unwrap();
}

```

For convenience, we have type aliased for Ruby string. We have an unsafe block here as the strings are from Ruby runtime and Rust has no way to know if the string borrowed is valid or its lifetime for that matter. It is the responsibility of Ruby runtime to manage the memory for it.

Recompiling our library post this change.

```
cargo build --release
   Compiling piglatin v0.1.0 (/Users/akshaysasidharan/Projects/rust-ruby/piglatin/piglatin)
    Finished release [optimized] target(s) in 2.27s

```

Whoa! Moment of truth now. (Dramatic drum roll cues in..)

```
 ❯ time ruby pig_latin_converter.rb
ruby rust_pig_latin_converter.rb  0.72s user 0.53s system 86% cpu 1.449 total
```

Wait what?! This is almost **90%** speed increase from our original Ruby program. That wasn't too hard to achieve and this process should get easier once we get the hang of it.

## Conclusion

For a little bit of trouble that we went through in building a native extension, the benefit of speed largely outweighs. To attach onto Rust interface we just had written some Ruby code, also thanks to Ruby-FFI. This is something I would now try to leverage if at all I feel being slowed down with Ruby going forward. Further to add, Ruby-FFI is portable across different platforms and different Ruby VMs such as MRI, JRuby, and Rubinius. 

But interfacing with Rust doesn't just end here. There are other awesome libraries out there that help to achieve the same thing such as [Helix](https://github.com/tildeio/helix) and [Rutie](https://github.com/danielpclark/rutie). Maybe we should compare and contrast these using the above program to check what we'd like more. Well, that would make for another post for another day. Code long and prosper until then.

