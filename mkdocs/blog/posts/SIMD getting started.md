---
date:
  created: 2026-02-01
authors:
  - weestack
comments: true
blog_toc: true
categories:
  - Rust
tags:
    - Rust
    - SIMD
    - CPU Instructions
---

# SIMD getting started
SIMD in simple terms incorporated cpu instructions where computations can take place in parallel, normally instructions happen one at a time but with SIMD we are able to perform multiple calculations at the same time, example 8 at the same time optimizing performance and power usage.   
This post will focus on NEON, rather than [Rust's STD SIMD](https://doc.rust-lang.org/std/simd/index.html), for a few reasons, I like specialization over generalization.

## Following the guide
Arm Developers has a guide on getting started with NEON, by taking an AVX (x86_64) example and converting it to NEON by looking up counterparts in [SIMD.info](SIMD.info).  
[Reference program com arm.com](https://learn.arm.com/learning-paths/cross-platform/simd-info-demo/simdinfo-example1/).

```C
#include <xmmintrin.h>
#include <stdio.h>

int main() {
    __m128 a = _mm_set_ps(16.0f, 9.0f, 4.0f, 1.0f);
    __m128 b = _mm_set_ps(4.0f, 3.0f, 2.0f, 1.0f);

    __m128 cmp_result = _mm_cmpgt_ps(a, b);

    float a_arr[4], b_arr[4], cmp_arr[4];
    _mm_storeu_ps(a_arr, a);
    _mm_storeu_ps(b_arr, b);
    _mm_storeu_ps(cmp_arr, cmp_result);

    for (int i = 0; i < 4; i++) {
        if (cmp_arr[i] != 0.0f) {
            printf("Element %d: %.2f is larger than %.2f\n", i, a_arr[i], b_arr[i]);
        } else {
            printf("Element %d: %.2f is not larger than %.2f\n", i, a_arr[i], b_arr[i]);
        }
    }

    __m128 add_result = _mm_add_ps(a, b);
    __m128 mul_result = _mm_mul_ps(add_result, b);
    __m128 sqrt_result = _mm_sqrt_ps(mul_result);

    float res[4];

    _mm_storeu_ps(res, add_result);
    printf("Addition Result: %f %f %f %f\n", res[0], res[1], res[2], res[3]);

    _mm_storeu_ps(res, mul_result);
    printf("Multiplication Result: %f %f %f %f\n", res[0], res[1], res[2], res[3]);

    _mm_storeu_ps(res, sqrt_result);
    printf("Square Root Result: %f %f %f %f\n", res[0], res[1], res[2], res[3]);

    return 0;
}
```

Here is my version of a counterpart in rust:
```Rust
use std::arch::aarch64::{
    vaddq_f32, 
    vcgtq_f32, 
    vld1q_f32, 
    vmulq_f32, 
    vsqrtq_f32,
    float32x4_t, 
    vgetq_lane_f32, 
};

fn main() {
    unsafe {
        let a_vec = [16.0f32, 9.0, 4.0, 1.0];
        let b_vec = [4.0f32, 3.0, 2.0, 2.0];
        let a: float32x4_t = vld1q_f32(a_vec.as_ptr());
        let b: float32x4_t = vld1q_f32(b_vec.as_ptr());

        let cmp = vcgtq_f32(a, b);
        let lanes: [u32; 4] = std::mem::transmute(cmp);

        for i in 0..4 {
            if lanes[i] != 0 {
                println!("{} is greater than {}", a_vec[i], b_vec[i]);
            } else {
                println!("{} is not greater than {}", a_vec[i], b_vec[i]);
            }
        }

        let add_result = vaddq_f32(a, b);
        let mul_result = vmulq_f32(add_result, b);
        let sqrt_result = vsqrtq_f32(mul_result);
        println!(
            "Addition results: {} {} {} {}",
            vgetq_lane_f32(add_result, 0),
            vgetq_lane_f32(add_result, 1),
            vgetq_lane_f32(add_result, 2),
            vgetq_lane_f32(add_result, 3)
        );

        println!(
            "Multiplication results: {} {} {} {}",
            vgetq_lane_f32(mul_result, 0),
            vgetq_lane_f32(mul_result, 1),
            vgetq_lane_f32(mul_result, 2),
            vgetq_lane_f32(mul_result, 3)
        );

        println!(
            "Square Root results: {} {} {} {}",
            vgetq_lane_f32(sqrt_result, 0),
            vgetq_lane_f32(sqrt_result, 1),
            vgetq_lane_f32(sqrt_result, 2),
            vgetq_lane_f32(sqrt_result, 3)
        );
    }
}
```

## Breaking it down
