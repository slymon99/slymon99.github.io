---
layout: post
title:  "Optimizing a sudoku solver in Rust"
date:   2020-08-10 12:00:00 -0400
---

## Sudoku Solvers
I enjoy writing sudoku solvers and improving them as a way to learn a new language (Rust, in this case). They aren't too large in scale, but are complex enough to expose you to a good amount of langauge features. They also can be iterated on and improved significantly.


## Final Source
[A fair bit of this code is fairly horrifying](https://github.com/slymon99/sudoku/blob/master/src/lib.rs)
## First Pass

```rust
pub struct Sudoku {
    pub board: Vec<u32>,
}

pub fn solve_sudoku(input: &mut Sudoku) -> bool {
    match find_first_empty(&input.board) {
        None => true,
        Some((row, col)) => {
            for option in options_for(&input.board, row, col){
                let idx = (row * 9 + col) as usize;
                input.board[idx] = option;
                if solve_sudoku(input) {
                    return true;
                }
                input.board[idx] = 0;
            }
            return false;
        }
    }

```

Here's my first, very rough pass, with a few helper functions omitted. 

1. Find the first empty space on the board. If there isn't an empty space, the puzzle is solved[^1]

2. For each possible option, fill in the empty space with that option. If the recursive call returns true, that means this option lead to a correct solution, so leave the board as is and return true.

3. If the recursive call does not return true, keep trying the other options.


Let's try out this solution on 50 puzzles (using [criterion](https://docs.rs/criterion) for benchmarking).

```
sudoku/solve/50         time:   [16.480 s 21.725 s 27.108 s]
                        thrpt:  [1.8445  elem/s 2.3015  elem/s 3.0340  elem/s]
```

Yikes. The three numbers given represent a 95% confidence interval. However you cut it, taking several hundred *milliseconds* is not the type of performance we are looking for. 

## Memoization

There's a ton of repeated work in our recursive calls. We calculate the available options, scanning through the row, column, and square, every single time. We can precalculate these options and simply update them on every iteration to save time.

```rust
pub struct Sudoku {
    pub board: Vec<u32>,
    pub row_memo: HashMap<u32, HashSet<u32>>,
    pub col_memo: HashMap<u32, HashSet<u32>>,
    pub square_memo: HashMap<(u32, u32), HashSet<u32>>,
}

pub fn solve_sudoku(input: &mut Sudoku) -> bool {
    match find_first_empty(&input.board) {
        None => true,
        Some((row, col)) => {
            for option in &options_for(&input, row, col) {
                let idx = (row * 9 + col) as usize;
                input.board[idx] = *option;
                remove_option(input, row, col, *option);
                let sol = solve_sudoku(input);
                if sol {
                    return sol;
                }
                input.board[idx] = 0;
                add_option(input, row, col, *option);
            }
            false
        }
    }
```

Not much has changed in our main loop, but our helper functions now look at the memos instead of scanning the entire board.

```rust
fn options_for(board: &Sudoku, row: u32, col: u32) -> HashSet<u32> {
    board
        .row_memo
        .get(&row)
        .unwrap()
        .intersection(&board.col_memo.get(&col).unwrap())
        .copied()
        .collect::<HashSet<u32>>()
        .intersection(&board.square_memo.get(&(row / 3, col / 3)).unwrap())
        .copied()
        .collect()
```

This function is fairly horrifying[^2], and does some extra allocations to take the intersection of the three memos (row, column, and square). But it does avoid scanning on every iteration, so it should give us a decent speedup.

```
sudoku/solve/50         time:   [5.3321 s 6.3623 s 7.4572 s]
                        thrpt:  [6.7049  elem/s 7.8588  elem/s 9.3771  elem/s]
                 change:
                        time:   [-77.715% -70.714% -59.563%] (p = 0.00 < 0.05)
                        thrpt:  [+147.30% +241.46% +348.74%]
                        Performance has improved.
```

Criterion shows a 70% decrease in time. But our total times are still quite slow.

## Profiling the memoized application

We can dive into `perf` to get a feel for where our compiled application is spending most of it's time.

```
  13.20%  sudoku   sudoku            [.] hashbrown::map::HashMap<K,V,S>::remove
  13.20%  sudoku   sudoku            [.] sudoku::lib::solve_sudoku
  12.99%  sudoku   sudoku            [.] hashbrown::map::HashMap<K,V,S>::insert
   9.96%  sudoku   sudoku            [.] core::hash::impls::<impl core::hash::Hash for u32>::hash
   8.23%  sudoku   sudoku            [.] hashbrown::map::HashMap<K,V,S>::contains_key
   8.01%  sudoku   sudoku            [.] hashbrown::raw::RawTable<T>::insert
   7.79%  sudoku   sudoku            [.] hashbrown::map::HashMap<K,V,S>::get_mut
   6.28%  sudoku   sudoku            [.] hashbrown::map::HashMap<K,V,S>::get_mut
   5.84%  sudoku   sudoku            [.] hashbrown::raw::RawTable<T>::reserve_rehash
   3.46%  sudoku   sudoku            [.] std::collections::hash::map::HashMap<K,V,S>::get
   2.38%  sudoku   sudoku            [.] hashbrown::raw::RawTable<T>::try_with_capacity
   1.73%  sudoku   sudoku            [.] hashbrown::map::make_hash
```

That's a lot of hashmap allocations. As someone who does a lot of python programming, it's fairly natural to to optimize everything with hashmaps / sets because the syntax and ergonimcs are so easy. I was also not able to find out a cleaner way of doing multiple set intersection without extra allocations. If this was a problem that requried these hashmaps and sets (plenty of DFS optimizations will require memoizing arbitrary data), I'd need to explore a better way of using these.

### Bitmaps

Once I step away from the knee-jerk response of using hashmaps and sets, there is a much clearer solution - bitmaps. After all, each of these hashsets only contains the numbers 1-9 and never anything else, so hashing was fairly pointless. Also, we can just use vecs instead of hashmaps to store each row/column/square memo (since there are always nine rows, columns, and squares).

```rust
pub struct Sudoku {
    pub board: Vec<u32>,
    pub row_memo: Vec<u32>,
    pub col_memo: Vec<u32>,
    pub square_memo: Vec<u32>,
}

pub fn solve_sudoku(input: &mut Sudoku) -> bool {
    match find_first_empty(&input.board) {
        None => true,
        Some((row, col)) => {
            for option in 1..=9 {
                if 1 << option - 1 & options_for(&input, row, col) == 0 {
                    let idx = (row * 9 + col) as usize;
                    input.board[idx] = option;
                    remove_option(input, row, col, option);
                    let sol = solve_sudoku(input);
                    if sol {
                        return sol;
                    }
                    input.board[idx] = 0;
                    add_option(input, row, col, option);
                }
            }
            false
        }
    }
}
```

This implementation has no allocations once the puzzle has started, just vec comparisons and bit operations, which the compiler should be able to heavily optimize. The changes are to store row/cell/square information in `u32`s, with a 1 indicating that bit was not available. You can see the full implementation of options_for, remove_option, and add_option by looking at the [source](https://github.com/slymon99/sudoku/blob/master/src/lib.rs).

And the speed improvements? Now that we are actually testing faster, we can test on a larger set of puzzles.

```
sudoku/solve/999        time:   [10.729 s 10.752 s 10.790 s]
                        thrpt:  [92.587  elem/s 92.913  elem/s 93.111  elem/s]]
```

More than a 10x speedup. 11ms a puzzle is getting decent. 

### Parallelization

The nice thing about rust is that since so much is threadsafe by default, it's super easy to parallelize. In this case, I can use [Rayon](https://docs.rs/rayon) to convert my test harnesses iterator into a parallel iterator.

```
fn solve_all_from_string_par(s: &str) -> bool {
    s.split('\n').collect::<Vec<_>>()
        .into_par_iter()
        .map(|line| solve_sudoku(&mut sudoku_from_line(line)))
        .all(|x| x)
}
```

```
sudoku/solve/999        time:   [3.2678 s 3.3386 s 3.4025 s]
                        thrpt:  [293.61  elem/s 299.23  elem/s 305.71  elem/s]
                 change:
                        time:   [-69.659% -68.949% -68.371%] (p = 0.00 < 0.05)
                        thrpt:  [+216.16% +222.05% +229.59%]
                        Performance has improved.
```

A 3x performance increase for about five minutes of work. Not bad.

There are more optimizations to make, but I'm going to leave this project here from now. We've achieved over a *100x* speedup.





[^1]: This program assumes that the puzzle is not filled in
[^2]: I'm still learning rust