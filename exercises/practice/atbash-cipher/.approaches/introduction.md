# Introduction

## General guidance

- Instead of writing out the whole alphabet, you can use a character range `'a':'z'`, if you like.
- `Iterators.partition` and `join` might be helpful for adding spaces between groups.
- Think about what happens to non-ascii input. What would you like to happen?

## Technique: ciphering a single character

The Atbash cipher requires mapping each occurrence of the nth letter in the alphabet with the (26-n)th letter.

You could acheive this by creating a dictionary something like this:

```julia
const cipher = Dict(zip(vcat('a':'z', 'A':'Z', '0':'9'),
                        vcat('z':-1:'a', 'z':-1:'a', '0':'9')))
```

and translating each character with `cipher[char]`.

Another option is to use the fact that the english alphabetic characters are represented in order in unicode (so the numerical representation of 'b' is one greater than the numerical representation of 'a').

That lets us encode a single character with these steps:

```julia
# Get the index of char in 'a':'z'
idx = char - 'a' + 1
# Mirror the index
reverseidx = 26 - idx
# Calculate the ciphered character
newchar = 'a' + reverseidx
```

This works because a `Char` + an `Int` gives a new `Char` and a `Char` subtracted from another `Char` gives the number of codepoints between them. So `'c' - 'a' == 2` and `'a' + 2 == 'c'`.

We can simplify the code a little:

```julia
'a' + 26 - (c - 'a' + 1)
# Or, simplified again
'a' + 25 - (c - 'a')
# Or, simplified again (25 + 'a' == 'z'):
'a' + ('z' - c)
```

## Approach: symmetrical functional

This approach emphasises that encoding and decoding are the same operation through code re-use.
We get a nice definition for encoding a single character (with a good output type that will force consumers to handle missing values explicitly)
and the `encode(str)` function is quite short and, if you are familiar with functional programming in Julia, fairly easy to read.

Compared to the lower-level approach below, this allocates more and performs extra work by creating various intermediate results that need to be transformed.
Of course, that will only rarely matteer.

The code uses `missing` because unencodable values do go missing and because `skipmissing` is a handy built-in iterator.
I could also have used `nothing` and provided a function like `skipnothing(xs) = (x for x in xs if !isnothing(x))`.

By cmcaine, inspired by
[halfdan](https://exercism.io/tracks/julia/exercises/atbash-cipher/solutions/419b6f4d04974a63b7f8531e8ad2808c)
and
[OTDE](https://exercism.io/tracks/julia/exercises/atbash-cipher/solutions/330cb9e444c2403c898275e71da07caa)'s
solutions.

```julia
using Base.Iterators: partition

"""
    encode(ch::AbstractChar)

Encode a character with the Atbash cipher using the ascii latin alphabet.

Letters are encoded, numbers are passed through unchanged, all other characters
return `missing`. The Atbash cipher only has one key, so encoding and decoding
are identical.
"""
function encode(ch::AbstractChar)
    if 'a' <= ch <= 'z'
        'a' + ('z' - ch)
    elseif 'A' <= ch <= 'Z'
        'a' + ('Z' - ch)
    elseif '0' <= ch <= '9'
        ch
    else
        missing
    end
end

"""
    encode(str; group=true)

Encode a string with the atbash cipher using the ascii latin alphabet.

Letters are encoded, numbers are passed through unchanged, all other characters
are filtered out. The Atbash cipher only has one key, so encoding and decoding
are identical.

`group` controls whether the output is continuous or in groups of 5 letters.
"""
function encode(str; group=true)
    xs = skipmissing(map(encode, collect(str)))
    group ? join(map(join, partition(xs, 5)), ' ') : join(xs)
end

"""
    decode(str)

A convenience function for `encode(str; group=false)`
"""
decode(str) = encode(str; group=false)
```

## Approach: low-level style

An efficient procedural-style solution to this problem.
This code is verbose, but through that verbosity manages to read the input only once and allocates very little memory.

This is the kind of code you might write when you want more control in a hot loop or if you are used to writing in C, Go or Rust.
The verbosity means that all of the relevant logic for `encode` is within the function; you don't need to reference other functions and there's very little chance of accidentally introducing an extra loop.

The `IOBuffer` used here is a convenient way to create a string piece by piece. If you use the more naive approach of concatenating onto a string in a loop then you'll end up generating many strings (each one is immutable, so concatenating two strings creates a third one).

This was slightly adapted from
[miguelraz and ScottPJones' solution](https://exercism.io/tracks/julia/exercises/atbash-cipher/solutions/037a4f733e734097a37b3287e84be40f)
by cmcaine.

```julia
function encode(input)
    buf = IOBuffer()
    # Every 6th character in output should be a space
    nextspace = 6
    for ch in input
        # Encode or exclude characters
        if 'a' <= ch <= 'z'
            ch = 'a' + ('z' - ch)
        elseif 'A' <= ch <= 'Z'
            ch = 'a' + ('Z' - ch)
        elseif '0' <= ch <= '9'
            # Pass through
        else
            continue # Exclude other chars
        end
        # Write a space if it's time
        nextspace -= 1
        if nextspace == 0
            # 5 rather than 6 because we're about to write two chars
            nextspace = 5
            write(buf, ' ')
        end
        write(buf, ch)
    end
    return String(take!(buf))
end

function decode(input)
    buf = IOBuffer()
    for ch in input
        if 'a' <= ch <= 'z'
            ch = 'a' + ('z' - ch)
        elseif ch == ' '
            continue
        end
        write(buf, ch)
    end
    return String(take!(buf))
end
```

## Approach: functional approach

A solution by [halfdan](https://exercism.io/tracks/julia/exercises/atbash-cipher/solutions/419b6f4d04974a63b7f8531e8ad2808c).

```exercism/warning

This solution will mangle non-[ascii][ascii] letter characters and pass-through symbols, separators, etc.
because it assumes that `isletter(c)` is the same as `c in 'a':'z' || c in 'A':'Z'` (it actually matches all unicode letters).
This is a common mistake and can cause real issues!

If you want to ensure your input is ascii you can use `ascii()`, or `isascii()`, or the Strs.jl package.

```

```julia
encode(input) = atbash(input, group=true)
decode(input) = atbash(input, group=false)

function atbash(input::AbstractString; group::Bool=false)
    input = [c for c in input if !(isspace(c) || ispunct(c))]
    result = join(map(atbash, input))
    if group
        join(map(join, Iterators.partition(result, 5)), ' ')
    else
        join(result)
    end
end

function atbash(c::Char)
    isletter(c) || return c
    'z' - (lowercase(c) - 'a')
end
```

## Approach: translation dictionary

Solution by [Nosferican](https://exercism.io/tracks/julia/exercises/atbash-cipher/solutions/4a06872ec87a4c91ab7c32e98682165e)

This solution defines a `Dict` that maps each valid character to its' ciphered version (`Cipher`). The definition for `decode` is then pretty simple.

The only complicated code is the code to partition the ciphertext into blocks of 5, and that could be done by Iterators.partition.

```julia
const Cipher = Dict(zip(vcat('a':'z', 'A':'Z', '0':'9'),
                        vcat('z':-1:'a', 'z':-1:'a', '0':'9')))
encode(obj) = [ get(Cipher, x, "") for x ∈ obj if x ∈ keys(Cipher) ] |>
              (obj -> join((join(obj[idx:min(idx + 4, end)]) for idx ∈ 1:5:length(obj)),
                           " "))
decode(obj) = join(get(Cipher, x, "") for x in obj)
```

[ascii]: https://en.wikipedia.org/wiki/ASCII
