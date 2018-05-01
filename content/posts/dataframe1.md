---
title: "Dataframes: Traits, Enums, Generics, and Dynamic Typing"
date: 2018-03-28T09:57:53-04:00
draft: false
tags: [
    "rust",
]
categories: [
    "code",
]
---
tldr: I'm attempting to build a dataframe in Rust. I implemented a pattern using traits, generics, and enums in conjunction to deal with columns of different datatypes while allowing runtime reflection for accessing the data stored in a column.

- [background](#background)
- [constraints and motivation](#constraints-and-motivation)
- [llamas2](#llamas2) (the dataframe PoC I'm building now)
  - [initial attempts](#initial-attempts)
  - [emulating dynamic types](#emulating-dynamic-types)
- [llamas 1 and beetl](#llamas-1-and-beetl)

A quick guide for different readers:

- for those who want to see code (and maybe give feedback!), the meat is in [emulating dynamic types](#emulating-dynamic-types). Go [here](#code-example) for just the code snippet, if you find my words confusing. Go [here](https://github.com/hwchen/llamas2/blob/master/examples/array.rs) for examples in the [repo](https://github.com/hwchen/llamas2)
- for those who want to read more background on how I'm using dataframes in work, [background](#background) and [constraints and motivation](#constraints-and-motivation) are the parts to read. These sections are more about software architecture and design, I think.
- for those who want to read my rambly thoughts and thought processes on iterating code, [llamas 1 and beetl](#llamas-1-and-beetl) will serve you well. Caution, this section is the least developed, the most rambly, and the least edited; it's for me to log my own design process.

This is my first attempt at a longer article explaining my code, so please bear with my errors and/or lack of clarity, I hope that they will improve with practice.

[llamas2](https://github.com/hwchen/llamas2), the dataframe I'm building, is currently a proof of concept.

Discuss on [reddit](https://www.reddit.com/r/rust/comments/8g2274/dataframes_traits_enums_generics_and_dynamic/)

# background

I'm writing a dataframe library, currently named [llamas2](https://github.com/hwchen/llamas2). I don't have any grand plans for it (yet), but I'm inspired by Pandas, which I use at work for small to medium ETLs.

The flexibility of a Pandas dataframe works well with the flexibility of Python. However, there are some performance issues (e.g. high memory usage) and idiosyncracies (e.g. no null values for integers). Work is underway on [Pandas 2](https://pandas-dev.github.io/pandas2/), which will address many of these issues by moving to a C++ backend.

Inspired by the design docs of Pandas 2, I thought it would be a cool idea to see how far I could get writing my own dataframe in Rust, just to see what would be possible.

# constraints and motivation

I'm using dataframes for ETL, so my priorites may be different from other users of dataframes.

Top priorites:

- reshaping (melting and pivoting)
- splitting string cols
- apply (or pandas map)
- group by
- filtering
- reading and writing csv (perhaps compressed also)
- generating sql for creating tables
- (performance and ergonomics of course)

non-focus areas:

- numerical computing (but maybe in the future)
- access by index
- iterating over rows of the dataframe
- ergonomics to python/pandas standard of ease of use

In terms of performance, my major design goal was to have columns which are implemented as an Array which combines a null bitvec representation with a Vec of a primitive type. From my understanding, this would result in the most compact representation (and the best alignment?), as compared to having each value be stored as an enum, like some other libraries.

I've seen e.g. `InnerType::Float(x)` in Utah, or `Nullable::Value(T)` and `Nullable::Null` in brassfibres. In addition to both not having the most compact representation, the usage of `InnerType` would seem to allow the use of mixed types within a series, which I would not want to allow.

My other inspiration is from databases. At work we use columnar database (Monet in particular) as a backend to an OLAP service. And my desire to learn more about databases also led me to [bradfield](https://bradfieldcs.com), where I took a computer architecture and a databases course. As the project for the databases course, I also wrote a [toy sql database executor](https://github.com/hwchen/lemurdb) in Rust.

Before moving onto the meat of this article, here's links to some other libraries solving data and etl problems.

- [datafusion](https://github.com/datafusion-rs/datafusion-rs) has a dataframe-like representation, but is meant to be used with sql and query planner on the frontend. Alternative to Spark.
- [brassfibre](https://github.com/sinhrks/brassfibre)
- [utah](https://github.com/kernelmachine/utah)

# llamas2

## initial attempts

I'd gotten away from working on llamas for almost a year, when I saw Andy Grove's [datafusion](https://github.com/datafusion-rs/datafusion) project. Browsing through the code, I saw that column types were held in an enum, with the underling physical representaion as a `Vec<T>`.

Snippet from [datafusion](https://github.com/datafusion-rs/datafusion/blob/0cb48011c5c035c2a131ebc5fa0f8dd2265243ac/src/data.rs#L64). (note: datafusion has since moved on to using apache arrow for the underlying representation)

```rust
#[derive(Debug,Clone)]
pub enum ColumnData {
    BroadcastVariable(Value),
    Boolean(Vec<bool>),
    Float(Vec<f32>),
    Double(Vec<f64>),
    Int(Vec<i32>),
    UnsignedInt(Vec<u32>),
    Long(Vec<i64>),
    UnsignedLong(Vec<u64>),
    String(Vec<String>),
    ComplexValue(Vec<ColumnData>)
}
```

Re-inspired by reading through the code, I decided to try my hand again at llamas, this time using enums as the basic interface to columns, instead of traits and structs.

However, I ran into the same issue as the last time I tried llamas using enums: It was easy for the dataframe to hold columns of different datatypes, but it was difficult to access the type of the underlying `Vec`. For me, this was a high priority as I'd need something akin to the pandas `apply` method on a `Series`. In Python, there's not a problem passing in a function, since the types don't need to be specified ahead of time. But in Rust the signature of the fn needs to be known ahead of time! How could I resolve this?

Methods on enums will often use a match statement to destructure `Self` and choose the correct path of computation. For example, [implementing](https://doc.rust-lang.org/book/first-edition/error-handling.html#the-error-trait) the `Error` trait for a custom `MyError` enum uses this strategy. I wanted something more though; I would need to be able to dynamically dispatch depending on the primitive type encapsulated within an enum variant, not the enum variant itself. It almost feels backwards, but for a method like `apply`, it's checking if the passed in fn will work with the existing enum variant.

I almost gave up at this point, since there was no enum machinery I knew of that would help me achieve this. But I decided to do one more extensive search for dynamic dispatch and methods.

In my search, I came across this reddit thread on [function overloading](https://www.reddit.com/r/rust/comments/7zrycu/so_function_overloading_is_part_of_stable_rust/). I probably read it through once or twice, thinking that it was just another example of the use of traits on structs, which I was trying to get away from. It was otherwise what I was looking for, though, which was that a trait defined with a generic would allow not only a method to be defined using that generic, but it would allow implementation of a method *for specific instances of the generic type!*.

But finally a lightbulb went off in my head, and I realized that there was no reason that I couldn't use traits on enums! (I don't know if this is an anti-pattern, and I would love feedback on this. See below section on [llamas 1 and beetl](#llamas-1-and-beetl) to see some reasoning on why I wanted to use enums to represent different column datatypes. I'm happy to be shown otherwise).

## emulating dynamic types

The feeling of this pattern is that there are two levels of dispatches:

1. a dispatch based on compile-time polymorphism (The trait with a generic type). This is used for methods which reference the type which the column is built on (i.e. a column of the datatype unsigned integer holds data in a `Vec<u32>`.
2. a dispatch based on run-time checks (matching on an enum variant). For methods which reference the type underlying a column, matching on an enum variant will ensure that the instance of the polymorphic method is being routed to the correct enum variant.

Point 2 also makes it easy for a `DataFrames` struct to hold different datatype variants.

A code example so that this will make some sense!

###### code example
```rust
struct DataFrame {
    pub columns: HashMap<String, Array>,
}

enum Array {
    Int32(ArrayData<i32>),
    Float32(ArrayData<f32>),
}

trait DataType<T> {
    fn apply<F>(&self, f: F) -> Result<Array, Error>
        where F: Fn(&T) -> T;
}

impl DataType<i32> for Array {
    fn apply<F>(&self, f: F) -> Result<Array, Error>
        where F: Fn(&i32) -> i32
    {
        match *self {
            Array::Int32(ref array_data) => Ok(Array::Int32(array_data.apply(f))),
            _ => Err(format_err!("Fn type mismatch, array is {}", self.dtype())),
        }
    }

impl DataType<f32> for Array {
    fn apply<F>(&self, f: F) -> Result<Array, Error>
        where F: Fn(&f32) -> f32
    {
        match *self {
            Array::Float32(ref array_data) => Ok(Array::Float32(array_data.apply(f))),
            _ => Err(format_err!("Fn type mismatch, array is {}", self.dtype())),
        }
    }
```

This is a simplified snippet from llamas2. Some points to note while reading through the code:

- `Array` is the "logical" representation of a column. Users will know that they have a column of the datatype Int32, but this could in fact be backed by any physical representation. It's of course simplest to use a `Vec<i32>`, but if I want to, say, handle nulls with a bit vector, `ArrayData` could also hold that. Or if I wanted to use interned strings instead of a `Vec<String>` for a categorical datatype, the backends could be switched while showing the same categorical datatype interface to the user.

- If you want to manipulate the data in an Array, you will need to know the type that each cell is represented by. The `apply` method takes a function that must know that type, and use it for both input and output.

- If trait `Datatype` did not reference a generic `T`, then `apply` would be constrained to only one type.

- If there was no trait but the `apply` method was generic like so:

```rust
impl Array {
    fn apply<F, T>(&self, f: F) -> Result<Array, Error>
        where F: Fn(&T) -> T
    {
        match *self {
            Array::Int32(ref array_data) => Ok(Array::Int32(array_data.apply(f))),
            _ => Err(format_err!("Fn type mismatch, array is {}", self.dtype())),
        }
    }
```
  there would be no way to match an item's type (`T`) to the array datatype (the enum variant), since in the fn `T` could be any type.

- When `impl DataType<f32> for Array`, Rust is performing a kind of function overloading using traits. I'm able to implement `DataType<T>` for each instance of `T`. Since `apply` will now know exactly the type of the `Fn` (including its input and output), the method can now use matching and destructuring of the Array enum to determine whetherthe instance of `Array` that the `apply` method was called on is compatible. Any incompatible variants will give a runtime error.

- `format_err!` is a quick way to create errors using [failure](https://github.com/rust-lang-nursery/failure)

If you want to play with examples, clone [llamas2](https://github.com/hwchen/llamas) and check out the [array example](https://github.com/hwchen/llamas2/blob/master/examples/array.rs) which shows some of the code in action.

Any feedback is welcome! I'd love to know if I've made some mistake (the compiler has already given me lots of feedback...), or if there's a more elegant solution.

(Note:) I've just been reminded of the existence of `std::any::Any`. I tried it before and I don't think it quite did what I wanted, but I'm happy to revisit it if feedback indicates that it would be a fruitful route.

The next section is pretty rambly and un-edited, so feel free to skip to the [end](#End).

# llamas 1 and beetl {#llamas-1-and-beetl}

I've actually been at this idea of dataframes for over a year now. My first attempt was [llamas](https://github.com/hwchen/llamas), which was a dataframe with the same design considerations as llamas2, but the different datatype columns were implemented using traits and structs.

Traits:
```rust
pub trait Column {}

pub trait DataType {
    type Item;

    fn values(&self) -> Series<Self::Item>
        where Self::Item: Clone;

    fn get(&self, index: usize) -> Option<Option<&Self::Item>>;

/// For DataType methods that use &mut, which means that they
/// can't be implemented on &Column types, only Column and &mut
/// Column
pub trait DataTypeMut: DataType {
    fn push(&mut self, item: Option<Self::Item>);
    fn apply<F>(&mut self, f: F) where
        Self: Sized,
        F: Fn(Self::Item) -> Self::Item + ::std::marker::Sync;
}
```
Example of Int8 datatype struct and trait impl:

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Int8Column {
    values: Vec<i8>,
    // Mask uses a bitvec overlaid onto values to know which indices hold
    // a null value. false in the bitvec maps to null in values.
    mask: BitVec,
}

impl Column for Int8Column{}

impl DataType for Int8Column {
    type Item = i8;

    fn get(&self, index: usize) -> Option<Option<&i8>> {
        if let Some(mask) = self.mask.get(index) {
            if !mask {
                return Some(None);
            }
        } else {
            return None;
        }
        Some(self.values.get(index))
    }

    fn values(&self) -> Series<Self::Item> {
        Series::new(self)
    }
}

impl DataTypeMut for Int8Column {
    fn push(&mut self, item: Option<i8>) {
        match item {
            Some(item) => {
                self.values.push(item);
                self.mask.push(true);
            },
            None => {
                self.values.push(0);
                self.mask.push(false);
            },
        }
    }

    fn apply<F>(&mut self, f: F)
        where F: Fn(i8) -> i8 + ::std::marker::Sync
    {
        // TODO best way to apply mask? zip values, or refer to mask by index?

        let mask = &self.mask;
        self.values
            .par_iter_mut()
            .enumerate()
            .filter(|&(i,_)| mask[i] )
            .for_each(|(_, x)| *x = f(*x));
    }
}
```

Why did I stop developing llamas version 1? I think a combination of things:

- I was more inexperienced
- I got stuck on an implementation detail for `String`s
- I thought it was ugly to have a struct `Int8Column`; I felt that it should be something more like `Column<T>`.

If I remember correctly, the issue of where to put the generic and/or associated type came down to a balance of needing the columns of the dataframe to be of different types while also being able to access the primitive type of the physical implementation. For example, if the dataframe holds a `Vec<Column<T>>`, every `Column` must have the same type `T`.

My workaround was to have each `XColumn` struct implement two traits: `Column`, an empty trait which would allow a dataframe to hold a `Vec<Box<Column>>` trait object, and `DataType`, which would hold the methods that would do the real work, and which would have an associated type which would allow access to the primitive type of the data. Access to the primitive type was necessary for methods such as `apply()`, which would take a function of the associated type.

Discouraged by what were, at the time for me, extreme type convolutions, I took a break and made a procedural macro to melt Array of Struct data: [beetl](https://github.com/hwchen/beetl)

Reviewing this code again now, I feel like there's awkwardness but don't know if the design can't move forward with some cleanup. There may be an immediate blocker that I forgot about, or perhaps something like melting in the future may be extremely inconvenient. An perhaps I had some inconveniences around using trait objects.

# End

I'm not the most clever programmer, but I'm peristent! I don't know that this method of dynamic typing is elegant (or even correct), but I'm slowly moving forward towards my goal of a dataframe in Rust, and learning a lot along the way!

Discuss on [reddit](https://www.reddit.com/r/rust/comments/8g2274/dataframes_traits_enums_generics_and_dynamic/)

Next time: writing a macro to melt.
