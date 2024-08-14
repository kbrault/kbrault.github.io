+++
title = "Euler's Number and Continuous Compounding in Rust: A Journey from Overcomplication to Simplicity or when std:: is enough."
date = 2024-08-13

description = "While custom methods offer valuable educational insights, Rust’s built-in functionalities provide a more efficient and precise solution for practical applications."

[taxonomies]
tags = ["math", "rust"]

[extra]
quick_navigation_buttons = true
footnote_backlinks = true
toc = false
+++

While working on a personal finance application in Rust, I needed to calculate a continuously compounding interest rate on a very large series. After several attempts, the result was not the fastest. Or rather, not as fast as a low-level language like Rust would lead you to expect. On closer inspection, the Euler constant of the standard library was extremely precise. I had no interest in using the precision of this constant, especially if it cost me in terms of performance. So I decided to use a less precise but more powerful representation of e (that I thought) ... what a mistake! It took at least this experience to remind me how optimized low-level computing is, and how I, as a small developer, must remain modest and not reinvent the wheel.

## Euler's Number e

Euler's number e is a transcendental number. It frequently appears in problems involving growth processes, finance and complex numbers. It is defined mathematically by the following infinite series:
$$
e = \sum_{n=0}^{\infty} \frac{1}{n!} \\
  = 1 + \frac{1}{1!} + \frac{1}{2!} + \frac{1}{3!} + \frac{1}{4!} + \cdots \\
$$
$$
  \approx 2.71828
$$

This series converges to the value of e as more terms are included. In other words, the larger n is, the more accurate the approximation of e is. But on the other hand, the larger n is, the more complex the series is to compute. And that was my problem, I (falsely) assumed that the e constant of the Rust standard library was to expensive in terms of performance.

## Approximating Euler's e Number
To compute e for practical applications, we often use a truncated series expansion. For a finite number of terms k, the approximation of e is given by:
$$
e \approx \sum_{n=0}^{k} \frac{1}{n!} 
$$
This approach helps in understanding how e behaves with a finite number of terms and can be implemented as follows in Rust:
```rust
fn compute_e_series(terms: u32) -> f64 {
    let mut sum = 0.0;
    let mut factorial = 1.0;

    for i in 0..terms {
        if i > 0 {
            factorial *= i as f64; 
        }
        sum += 1.0 / factorial; 
    }

    sum
}
fn main() {
    let e_series = compute_e_series(20); // 20 terms in the series (... + 1/20!)
    println!("Approximate value of e using series: {}", e_series);
}
```
This function iteratively computes the factorial and sums the reciprocals to approximate e. As the number of terms increases, the approximation approaches the true value of e.

I did some benchmark to compare the approximate value of e using this serie representation versus the built-in function, the result was flattering : 
- Approximate value of e using series: 2.7182818284590455
- Time taken using series: 4.11µs
- Value of e using built-in function: 2.718281828459045
- Time taken using built-in function: 9.95µs

Assuming that a 20-term approximation of e is 2 times more efficient than the built-in function, here I am using this function in my application of Continuous Compounding.

## Continuous Compounding and Its Calculation

Continuous compounding is a financial concept where interest is compounded at every possible instant. The formula to compute the future value A of an investment with continuous compounding is :
$$
A=P e ^ {rt}
$$
where:
- A is the future value
- P is the principal amount or initial investment
- r is the annual interest rate
- t is the time in years

I implemented this in Rust to calculate future values using the approximation of e:
```rust
fn continuous_compounding(principal: f64, rate: f64, time: f64, e_value: f64) -> f64 {
    principal * e_value.powf(rate * time)
}

fn main() {
    let principal = 1000.0; // Initial investment
    let rate = 0.05;        // 5% annual interest rate
    let time = 10.0;        // Investment period of 10 years
    
    let future_value = continuous_compounding(principal, rate, time, compute_e_series(20));

    println!("Future value with approximation of e: {:.2}", future_value);

    // return : Future value with approximation of e: 1648.72
}
```
## But it doesn't work ...
The aim of my application is to find the best interest rate and investment period on numerous products. So I have to make thousands of calculations on thousands of different series. And then it's the tragedy ! The performances are catastrophic. It's impossible to reduce the number of terms used to approximate e (the result won't be good), so what's going on? 

I decided to compare performances by stressing both my approximation and the built-in function.

## Comparative Analysis: Approximation vs. Built-in Constant

```rust
use std::time::{Duration, Instant};

fn compute_e_series(terms: u32) -> f64 {
    let mut sum = 0.0;
    let mut factorial = 1.0;

    for i in 0..terms {
        if i > 0 {
            factorial *= i as f64; 
        }
        sum += 1.0 / factorial;
    }

    sum
}

fn continuous_compounding(principal: f64, rate: f64, time: f64, e_value: f64) -> f64 {
    principal * e_value.powf(rate * time)
}

fn average_time<F>(func: F, iterations: usize) -> Duration
where
    F: Fn() -> (),
{
    let mut total_nanos = 0u128;

    for _ in 0..iterations {
        let start_time = Instant::now();
        func();
        total_nanos += start_time.elapsed().as_nanos();
    }

    Duration::from_nanos((total_nanos / iterations as u128) as u64)
}

fn main() {
    let principal = 1000.0; 
    let rate = 0.05; 
    let time = 10.0; 
    let true_e = std::f64::consts::E; 
    let terms = 20; 
    let iterations = 1000000; // Number of iterations for performance measurement

    println!("Performance with manual approximation ({} terms):", terms);
    let avg_duration_approx = average_time(
        || {
            let approximation = compute_e_series(terms);
            let _ = continuous_compounding(principal, rate, time, approximation);
        },
        iterations,
    );

    println!(
        "Average time taken for approximation with {} terms: {:?}",
        terms, avg_duration_approx
    );
    println!();

    println!("Performance with built-in constant e:");
    let avg_duration_builtin = average_time(
        || {
            let _ = continuous_compounding(principal, rate, time, true_e);
        },
        iterations,
    );

    println!(
        "Average time taken with built-in e: {:?}",
        avg_duration_builtin
    );
}

```

    Performance with manual approximation (20 terms):
    Average time taken for approximation with 20 terms: 308ns

    Performance with built-in constant e:
    Average time taken with built-in e: 60ns

## Conclusion

This exploration highlights the trade-offs between custom approximation methods and built-in constants. 

While custom methods offer valuable educational insights, Rust’s built-in functionalities provide a more efficient and precise solution for practical applications.

But let's be honnest : I can try to match the performance of standard library functions by implementing them by myself. But chances are I would end up with a worse version of them (see above). And even worst, I can implement them wrong by missing some corner cases or choosing a wrong optimization (is 20 terms enough for every computations of e ?)

Standard library has access to faster assembly instructions I cannot normally access in the language. And that, is why I'm digging deeper and deeper. High level languages are cool but understanding how it works under the hood is even better ! 