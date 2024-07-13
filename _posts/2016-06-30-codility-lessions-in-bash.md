---
layout: post
title: Solving Codility exercises in shell script
tags: shell
---

This article examines the process of solving Codility programming challenges using Bash scripting. While Bash may not be the most intuitive choice for algorithmic problems, it offers an interesting perspective on problem-solving and can help improve one's command of shell scripting.

The [first exercise](https://codility.com/programmers/task/binary_gap/) is the binary gap. Main job is done by `bc` calculator. Take away from this is the horrible pattern for dealing with trailing characters

```bash
function convert {
  # convert to binary
  local b=$( echo "obase=2;$1" | bc )

  # remove trailing spaces
  local c="${b%"${b##*[!0]}"}"

  # split by 1s and run high water mark algorithm
  local i r=0
  for i in ${c//1/ } ; do
    l=${#i} && (( l > r )) && (( r=l ))
  done

  echo "$r"
}

convert 20
convert 1041
convert 2043403647
```

The [second exercise](https://codility.com/programmers/task/cyclic_rotation/) is quite a bit of pain. Passing arrays into and from functions is complex. I unwrap the values when calling the function and collect it back after shifting away the first argument. Clearly it won't work for functions that accept multiple arrays as arguments. The algorithm by itself is trivial though.

```bash
function rotate {

  # get the first element and shift as array is passed 'unwrapped'
  local k="${1}"
  shift
  # wrap back the array and get its length
  local a=("${@}")
  local n="${#i[@]}"

  # calculate the split, swap sides
  (( k = n - k % n ))
  local p=("${a[@]:0:k}")
  local q=("${a[@]:k}")
  local r=("${q[@]}" "${p[@]}")

  echo "${r[@]}"
}

i=(3 8 9 7 6)
rotate 3 "${i[@]}"

i=("dude" "there" "you go")
rotate 2 "${i[@]}"
```

The [third exercise](https://codility.com/programmers/task/odd_occurrences_in_array/) is trivial once you know the trick that folding arrays with `xor` gives you the `xor` fold of unpaired elements. The logic translates directly into shell script.

```bash
function pair {
  local i r=0
  for i in "$@" ; do
    (( r ^= i ))
  done
  echo "$r"
}

pair 9 3 9 3 9 7 9
```

The [Frog Jump Exercise](https://codility.com/programmers/task/frog_jmp/) is another application of `(( ... ))` magic. Very little is learned except that `local x=1 y=$x` will not work in bash, and to see those kind of errors always run your bash scripts with `set -o nounset`.

```bash
function jump {
  local a=$1 b=$2 c=$3
  local d=$(( b - a ))
  local j=$(( d / c ))
  (( j * c == d )) || (( j += 1 ))
  echo "$j"
}

jump 10 85 30
```

The [Tape Equilibrium Exercise](https://codility.com/programmers/task/tape_equilibrium/) is where we discover that we don't have `abs` function in shell (in korn shell we do though).

```bash
function balance {
  local b=0 a=0 i d

  # compute the total weight of the bar
  for i in "$@" ; do
    (( b += i ))
  done

  # set max to unbalanced (0 -- all) distribution
  local m=$b
  for i in "$@" ; do

    # move a piece from left to right
    (( a += i )) && (( b -= i ))

    #  compute the absolute difference
    (( b > a )) && (( d = b - a )) || (( d = a - b ))

    # update low water mark
    (( d < m )) && (( m = d ))
  done
  echo "$m"
}

balance 3 1 2 4 3
```

To solve [Permutation Missing Element Exercise](https://codility.com/programmers/task/perm_missing_elem/) we can use the fact that the sum of arithmetic progression is a closed form expression and can be computed in O(1). If we compare this theoretical sum to the actual sum of the array the difference will give us the missing argument.

```bash
function missing {
  local s=0 n="$#" i

  for i in "$@" ; do
    (( s += i ))
  done
  echo $(( (n + 2) * (n + 1) / 2 - s ))
}

missing 2 3 1 5
```

The [Frog River One](https://codility.com/programmers/task/frog_river_one/) kind of pushes us towards using proper ADTs, in this case hash maps. I cheated a bit here storing numbers in a `:` separated string `a`. As this datatype doesn't track the size of our 'list', we better store the size in an additional state variable `w`.

```bash
function wait {
  # get the width of the river
  local x=$1
  shift

  # a is a list :1:6:7: with O(n) for membership check
  local i a=':' d c=0 b w=0
  for i in "$@" ; do
    if [[ "$a" != *":${i}:"* ]] ; then
      # add item to list
      a="${a}${i}:"
      (( w += 1 ))
      (( w == x )) && { echo "$c" ; return ; }
    fi
    (( c += 1 ))
  done
}

wait 5 1 3 1 4 2 3 5 4
```

Another [Permutations Exercise](https://codility.com/programmers/task/perm_check/), another cheat :). I just sort the values, which is O(n * log n) operation and check if the sorted order contains gaps. It's not as fast as O(n) but definitely better than naive repeated linear scanning which is O(n * n).

```bash
function permchk {
  local i local a=$( echo "$@" | tr " " "\n" | sort | tr "\n" " " ) c=1
  for i in $a ; do
    (( c == i )) && (( c += 1 )) || { echo false ; return ; }
  done
  echo true
}

permchk 4 3 1 2
permchk 4 1 3
```

The next [Missing Integer Exercise](https://codility.com/programmers/task/missing_integer/) is pretty much the variation to the theme, just add uniqueness to `sort -u` to skip over duplicates

```bash
function findmiss {
  local i local a=$( echo "$@" | tr " " "\n" | sort -u | tr "\n" " " ) c=1
  for i in $a ; do
    (( c == i )) && (( c += 1 )) || { echo $c ; return ; }
  done
}

findmiss 1 3 6 4 1 2
```

So far the ugliest solution for [Max Counts Exercise](https://codility.com/programmers/task/max_counters/) where I try to use fixed length delimited string as poor man's array. The thorny path led me to few realizations. Firstly, `{1..n}` expansions don't "understand" variables therefore `{1..$n}` won't work. Secondly, there's no simple way to repeat a string multiple times like `"hello" * 5` in Python. I had to write a function instead. Thirdly, it's not so easy to work with strings as arrays as constant padding and trimming is not fun and adds significant clutter, so I probably gonna pass on this "data structure" in the future.

```bash
readonly W=5

# repeat string $1 $2 times
function rep {
  local a="" i
  for (( i = 0 ; i < $2 ; i++ )) ; do
    a="$a$1"
  done
  echo "$a"
}

# pad the value with spaces from left
function pad {
  printf "%${W}s" $1
}

function count {
  local n=$1
  local m=$( pad 0 )
  # initialize array "....0....0....0....0"
  local a=$( rep "$m" $n )
  shift

  local i v w b c
  for i in "$@" ; do
    if (( i > n )) ; then
      # refill with max
      a=$( rep "$m" $n )
    else
      (( b = (i - 1) * W ))
      (( c = i * W ))
      # the ugly bit, update the counter, and pad it again
      v=$( pad $(( ${a:b:W} + 1 )) )
      (( v > m )) && m=$v
      a="${a:0:b}${v}${a:c}"
    fi
    echo "$a"
  done
}

count 5 3 4 4 6 1 4 4
```

Following was another [Counting Exercise](https://codility.com/programmers/task/passing_cars/). The solution is trivial, adding the code for completeness' sake.

```bash
function pair {
  local i s=0 r=0
  for i in "$@" ; do
    (( s += i ))
  done
  for i in "$@" ; do
    (( i == 0 )) && (( r += s )) || (( s-- ))
  done
  echo "$r"
}

pair 0 1 0 1 1
```

After that a [simple exercise in rounding up and down](https://codility.com/programmers/task/count_div/). Move the `a` right to the nearest number divisible by `c`, move the `b` left to the nearest number divisible by `c`. Compute the number of intervals fitting in between and add one to count the dots.

```bash
function count {
  local a=$1 b=$2 c=$3 n=0
  (( a % c == 0 )) || (( a = (a / c + 1) * c ))
  (( b = (b / c) * c ))
  echo $(( (b - a) / c + 1 ))
}

count 6 11 2 # 6 8 10
count 6 10 2 # 6 8 10
count 7 12 3 # 9 12
```

At one point I'll need to start using proper data types such as hash map. Well, maybe next time. For [Genomic Range Query](https://codility.com/programmers/task/genomic_range_query/) I've just created 4 arrays for folds, which will do for now, but are clearly very verbose. Learnings from this exercise is that looping through characters in a sting needs a a function call to `scan`, array concatenation `+=(1)` is very different from `+=1`, where the second expression just quietly promotes (demotes?) array to string, and that `&& || && ||` chains don't always work for long statements with arithmetic context `(( ))`. I will need to explore more on the last point.

```bash
function nucleo {
  local n=$1
  shift

  local a=0 c=0 g=0 t=0
  declare -a as=(0) cs=(0) gs=(0) ts=(0)
  for i in $( echo "$n" | fold -w1 ) ; do
    case "$i" in
      A) (( a++ )) ;;
      C) (( c++ )) ;;
      G) (( g++ )) ;;
      T) (( t++ )) ;;
    esac
    as+=($a) ; cs+=($c) ; gs+=($g) ; ts+=($t)
  done

  declare -a b vs
  for i in "$@" ; do
    b=( ${i//:/ } )
    p=${b[0]}
    q=$(( ${b[1]} + 1 ))
    if   (( as[q] - as[p] > 0 )) ; then vs+=(1)
    elif (( cs[q] - cs[p] > 0 )) ; then vs+=(2)
    elif (( gs[q] - gs[p] > 0 )) ; then vs+=(3)
    else                                vs+=(4)
    fi
  done
  echo "${vs[@]}"
}

nucleo CAGCCTA 2:4 5:5 0:6
```

By the way we probably don't need `ts` as we infer existence of `T`s indirectly in the else branch.

[Next exercise](https://codility.com/programmers/task/min_avg_two_slice/) is as boring as the rest of summation exercises. The jobs is greatly obscured by the syntax. Learnings from this one are none, except I am getting less and less gotchas with bash arrays.

```bash
function minavg {
  # scan, use s as threshold
  declare a=(0)
  local i s=0
  for i in "$@" ; do
    (( s+= i ))
    a+=($s)
  done

  # do not divide, work with the average multiplied by 6
  local v r
  for (( i = 2 ; i < ${#a[@]} ; i++ )) {
    (( v = (a[i] - a[i - 2]) * 3 ))
    (( v < s )) && (( s = v )) && (( r = i - 2 ))
    if (( i > 2 )) ; then
      (( v = (a[i] - a[i - 3]) * 2 ))
      (( v < s )) && (( s = v )) && (( r = i - 3 ))
    fi
  }
  echo "$r"
}

minavg 4 2 2 5 1 5 8
```
The [triangle exercise](https://codility.com/programmers/task/triangle/) is actually the first one making bash shine ... a bit. Making an array out of an input string and sort it numerically in one line is not bad at all. Took me literally 2 minutes to solve it, especially because they've put `O(n log n)` into expected complexity. As one of my professors used to say if your math teacher asks you 'which number' the right answer is almost always "0", sometimes "1".

```bash
function triangle {
  local i v
  declare -a t r=()
  for i in "$@" ; do
    t=($( echo "$i" | tr ":" "\n" | sort -n | tr "\n" " " ))
    r+=($(( t[0] + t[1] > t[2] )))
  done
  echo "${r[@]}"
}

triangle 10:2:5 1:8:20 1:1:1
```

Followed up by another [short exercise](https://codility.com/programmers/task/distinct/) leveraging `sort -u`. Nothing learned, except maybe that `uniq` compares adjacent lines only, so there's no way to avoid explicit sorting.

```bash
function distcount {
  declare -a v=($( echo "$@" | tr " " "\n" | sort -u | tr "\n" " " ))
  echo "${#v[@]}"
}

distcount 2 1 1 2 3 1
```

And [another one](https://codility.com/programmers/task/max_product_of_three/) aiming at the same "skill".

```bash
function maxprod {
  declare -a i=($( echo "$@" | tr " " "\n" | sort -n | tr "\n" " " ))
  local a b n=${#i[@]}
  (( a = i[0] * i[1] * i[n - 1] ))
  (( b = i[n -3] * i[n - 2] * i[n - 1] ))
  (( a > b)) && echo "$a" || echo "$b"
}

maxprod -3 1 2 -2 5 6
```

At this point I have to admin I don't commit nearly as many syntactic errors as before. So let's keep going and see where it's gonna take me.

I spent quite some time understanding the [next exercise](https://codility.com/programmers/task/number_of_disc_intersections/) but decided to cheat a bit and take [someone else's solution](http://www.lucainvernizzi.net/blog/2014/11/21/codility-beta-challenge-number-of-disc-intersections/). Interestingly bash really excelled in this exercise. The complete solution is not much longer than Luca's python code. Learnings are numerous. Firstly, you can pipe the loops into construct thus avoiding storage. Secondly, pipes operate with their own namespace so assigning an outer variable from within piped code is horribly complex. Thirdly, once in pipes, stay in pipes. Still, I don't know how to reference input array `$@` from arithmetic context `(( ))`, will have to explore it later.

```bash
function discint {
  declare -a a=("$@")
  local i b c
  for (( i = 0 ; i < ${#@} ; i++ )) ; do
    echo $(( i - a[i] )) "\t1"
    echo $(( i + a[i] )) "\t-1"
  done | grep -v "^$" | sort -k1,1n -k2,2rn | while read _ i ; do
    (( i > 0 )) && (( b += c ))
    (( c += i ))
    echo "${b}"
  done | tail -1
}

discint 1 5 2 1 4 0
```

And after the last high another low with the [fish](https://codility.com/programmers/task/fish/). Considerable amount of time is spent bending the data structures around to reduce the amount of clutter. I took a longer path to create a pipeable function. Still hasn't figured it out how to better return values from a pipe, other than by `tail -1`. Another pain is adding things to an empty array. There must be a better way than checking `-z "${s+_}"`.  This experience reminded me of my time at the university where all our code was highly optimized to make it run faster by completely disregarding readability of the code and using contra-intuitive data types.

```bash
function unwrap {
  tr ' :' "\n\t" | while read -r w d ; do
    (( d == 1 )) && echo $(( w * -1 )) || echo "$w"
  done
}

function fish {
  declare -a s=()
  local i a l=0
  unwrap | while read i ; do
    # push new fish to stack be default
    a=true
    while (( s[0] < 0 )) ; do
      if (( s[0] * -1 > i )) ; then
        # our upstream fish was eaten by the top of the stack
        a=false ; break
      else
        # top of the stack was eaten, reduce count and pop
        (( l-- )) ; s=("${s[@]:1}")
      fi
    done
    # adding new fish to the stack
    if [ $a == true ] ; then
      (( l++ ))
      [ -z "${s+_}" ] && s=($i) || s=("$i" "${s[@]}")
    fi
    # echo current length
    echo "$l"
  done | tail -1
}

echo 4:0 3:1 2:0 1:0 5:0 | fish
```

Followed by [another stack exercise](https://codility.com/programmers/task/brackets/). Again learnings are to use data structures that reduce the need to do operations in bash, like translating brackets to numbers with offset of 3, which lets us easily match opening and closing brackets in one operation.

```bash
function brackets {
  tr "{[(}])" "123456" | fold -w1 | while read c ; do
    if (( c < 4)) ; then
      [ -z "${s+_}" ] && s=($c) || s=($c "${s[@]}")
    else
      (( c - s[0] == 3 )) && s=("${s[@]:1}") || break
    fi
    [ -z "${s+_}" ] && echo true || echo false
  done | tail -1
}

echo "{[()()]}" | brackets
echo "([)()]" | brackets
```

[Next one](https://codility.com/programmers/task/stone_wall/) was quite interesting, as it required a bit of set-theoretic proof to bi-directionally map two sets, thus reducing the problem to a much simpler one. Learning is really just one, but important one, you can create an initializing block inside of piped coded hiding it behind some sort of `[ -z "${s+_}" ]` test. Getting more and more used to pipes and stuff.

```bash
function wall {
  tr " " "\n" | while read h ; do
    [ -z "${s+_}" ] && { declare -a s=() ; local n=0 c=0 ; }
    while (( n > 0 )) && (( s[n-1] > h )) ; do (( n-- )) ; done
    if (( n == 0 )) || (( s[n-1] != h )) ; then
      (( c++ ))
      s[n]=${h}
      (( n++ ))
    fi
    echo "$c"
  done | tail -1
}

echo 8 8 5 7 9 8 7 4 8 | wall
```

The [EquiLeader](https://codility.com/programmers/task/equi_leader/) is easier to solve if we recognize that if a number dominates two parts then is will dominate the complete sequence. Therefore we compute the dominating number and count it in as `$s` and then run through sequence checking the balances

```bash
function eqlead {
  declare -a s=($( echo "$@" | tr " " "\n" | sort | uniq -c | sort -k1,1nr | head -1 ))
  local i a=0 b=$# l=0 r="${s[0]}"

  for i in "$@" ; do
    (( a++ )) ; (( b-- ))
    if (( i == s[1] )) ; then
      (( l++ )) ; (( r-- ))
    fi
    (( l * 2 > a )) && (( r * 2 > b )) && echo $(( a - 1 ))
  done
}

eqlead 4 3 4 4 4 2
```

The [Dominator](https://codility.com/programmers/task/dominator/) should probably come before EquiLeader. Nothing interesting really, except tried not to loop over things explicitly and use pipes as much as possible. Still need to capture intermediate results with `$()` because I don't see any nice way of branching and merging the pipes althogh [this](http://unix.stackexchange.com/a/66419) probably points in the right direction. Another surprise is that `uniq -c` does some unexpected padding on counter column therefore I am not sure how to `cut -f1` out of results and have to go with `read -r n _`. By the way `read -r n` would capture complete lines

```bash
function dominator {
  declare -a c=($( tr ' ' "\n" \
                 | sort \
                 | uniq -c \
                 | sort -k1,1nr \
                 | while read -r n _ ; do echo "$n" ; done \
                 | tr "\n" ' ' ))
  local s=$( echo "${c[@]}" | tr ' ' '+' | bc )
  echo $(( c[0] * 2 >  s ))
}

echo 3 4 3 2 3 -1 3 3 | dominator
```

While looking for solving the [next one](https://codility.com/programmers/task/max_profit/) I discovered a strangely popular topic among bash programmers: how to read the first and the last lines from a file. For documentation purposes `{ IFS= read -r a ; echo "$a" ; tail -1 ; }` worked for me. The algorithm is straightforward. If the new value is below running minimum, update the minimum, otherwise check if trading at this point would increase running max profit, and update it if required.

```bash
function mprofit {
  local i m=$1 b=0
  for i in "${@:1}" ; do
    if (( i > m )) ; then
      (( i - m > b )) && (( b = i - m ))
    else
      (( m = i ))
    fi
  done
  echo "$b"
}

mprofit 23171 21011 21123 21366 21013 21367
```

[Next task](https://codility.com/programmers/task/max_double_slice_sum/) made me realize I have to write routines to reverse arrays myself. Implemented [Kadane's algorithm](https://en.wikipedia.org/wiki/Maximum_subarray_problem).

```bash
function kadane {
  declare -a a=("$@") b=(0)
  local i n=$# c=0 d=0
  for (( i = 1 ; i < n - 1 ; i++ )) ; do
    (( c + a[i] > 0 )) && (( c += a[i] )) || (( c = 0 ))
    (( d < c )) && (( d = c ))
    b+=($d)
  done
  b+=(0)
  echo "${b[@]}"
}

function rev {
  local i
  declare -a a=()
  for i in "$@" ; do [ -z "${a+_}" ] && a=($i) || a=($i "${a[@]}") ; done
  echo "${a[@]}"
}

function mdslice {
  declare -a p=($( kadane "$@" )) q=($( rev $( kadane $( rev "$@" ) )))
  local n=$# m=0 s
  for (( i = 1 ; i < n - 1 ; i++ )) {
    (( s = p[i-1] + q[i+1] ))
    (( s > m )) && (( m = s ))
  }
  echo "$m"
}

mdslice 3 2 6 -1 4 5 -1 2
```

Skipping the [max slice sum](https://codility.com/programmers/task/max_slice_sum/) as this is just a simpler version of the last one.

Following are a set of exercises that I expect will make me use `bc` a lot. The [first one](https://codility.com/programmers/task/min_perimeter_rectangle/) is about finding integer square root.

```bash
function minrec {
  local i n=$( echo "scale=0;sqrt($1)" | bc ) a=$1
  for (( i = n ; i > 0 ; i-- )) ; do
    (( a % i == 0 )) && { echo $(( 2 * (i + (a / i)) )) ; break ; }
  done
}

minrec 30
```

And the following [Count Factors Exercise](https://codility.com/programmers/task/count_factors/) is pretty much the same loop, just a different counter and a different break condition.

```bash
function nfactors {
  local i n=$( echo "scale=0;sqrt($1)" | bc ) a=$1 r=0
  # set the counter to -1 if a is a square to not double count sqrt(a)
  (( n * n == a )) && (( r = -1 ))
  for (( i = n ; i > 0 ; i-- )) ; do
    (( a % i == 0 )) && (( r += 2 ))
  done
  echo "$r"
}

nfactors 24
```

For the [Peaks Exercise](https://codility.com/programmers/task/peaks/) I just took [someone else's code](http://stackoverflow.com/questions/20886486/codility-peaks-complexity), as it was a fairly uninspiring problem. I've added comments to the logic, one interesting nuance of the algorithm is the part with integer divisions in `c / i > f` and `c / i == f` used to check if a peak falls within current group. Very nice indeed. No new bash gotchas.

```bash
function peaks {
  local n=$# i f g s c
  declare -a a=("$@") p=()

  # compute all the peaks
  for (( i = 2 ; i < n - 1; i++ )) ; do
    (( a[i-1] < a[i] )) && (( a[i] > a[i+1] )) && p+=($i)
  done

  for (( i = 1 ; i <= n ; i++ )) ; do
    # for all group sizes i that evenly divide n
    (( n % i != 0 )) && continue

    # number of peaks found f, number of groups g, success s
    f=0 ; g=$(( n / i )) ; s=1

    for c in "${p[@]}" ; do
      # overshot, breaking out
      (( c / i > f )) && { s=0 ; break ; }
      # falls into the group, increase counter
      (( c / i == f )) && (( f++ ))
    done
    # if found less than g set success status to 0
    (( f != g )) && s=0
    # super successful exit
    (( s == 1 )) && { echo "$g" ; break; }
  done
}

peaks 1 2 3 4 3 4 1 2 3 4 6 2
```

The [count semi-primes exercise](https://codility.com/programmers/task/count_semiprimes/) is trivial but it's nice to play with memoization in bash. I've added two indices `PRIMES_INDEX` and `PRIMES` containing respectively indices of prime numbers to quickly answer a question if a number is a prime and to loop quickly through prime numbers.

```bash
declare -a PRIMES_INDEX=(0 0 1 1 0 1) PRIMES=(2 3 5)
P_CURRENT=5

function primes {
  local b=$1 f p
  while (( P_CURRENT * P_CURRENT <= b )) ; do
    (( P_CURRENT++ ))
    f=1
    for p in "${PRIMES[@]}" ; do
      (( P_CURRENT % p == 0 )) && { f=0 ; break; }
    done
    (( PRIMES_INDEX[P_CURRENT] = f ))
    (( f == 1 )) && PRIMES+=($P_CURRENT)
  done
  echo "${PRIMES[@]}"
}

function semipr {
  local i j c p

  for i in "$@" ; do
    r=(${i//:/ })
    (( c = 0 ))
    primes "${r[1]}"
    for (( j = r[0] ; j <= r[1] ; j++ )) ; do
      for p in "${PRIMES[@]}" ; do
        (( p * p > j )) && break;
        if (( j % p == 0 )) && (( PRIMES_INDEX[j / p] == 1 )) ; then
          (( c++ ))
          break
        fi
      done
    done
    echo "$c"
  done
}

semipr 1:26 4:10 16:20 12:2035
```

Very much the same applies for the [count non divisible](https://codility.com/programmers/task/count_non_divisible/), it makes perfect sense to create indices during linear scan to avoid search later on. In this case we needed an index of occurences `os` which for every number `i` gives you the number of times `os[i]` the number `i` occurs in the input array. And the array `cs` is the number of non divisors which is uniformely set to `n` at the beginning, and then reduce while we loop over divisors.

```bash
function nondiv {
  declare -a cs=() os=() rs=()
  local i=0 n=$# j d

  while (( i < 2 * n )) ; do (( os[i++]=0 )) ; done
  for a in "$@" ; do (( os[a]++ )) ; done

  (( i = 0 )) ; while (( i < 2 * n )) ; do (( cs[i++] = n )) ; done

  for (( d = 1 ; d < 2 * n ; d++ )) ; do
    (( os[d] == 0 )) && continue
    for (( j = d ; j < 2 * n ; j += d )) ; do
      (( cs[j] -= os[d] ))
    done
  done

  for a in "$@" ; do
    rs+=("${cs[$a]}")
  done

  echo "${rs[@]}"
}

nondiv 3 1 2 3 6
```

Followed by [chocolates by numbers](https://codility.com/programmers/task/chocolates_by_numbers/), where you have two different velocities around circle, `m` and `n`. Let's linearize positions. So `a[i+1] = a[i] + n`. To check that two linearized positions represent the same place on the circle we wrap around `m`, in other words steps `p` and `q` represent the same position on the circle if `a[p] % m == a[q] % m` or `(a[p] - a[q]) % m == 0`. And for the same two positions to be on the same progression we need the difference to be divisible by `n` as well `(a[p] - a[q]) % n == 0`.

```bash
function gcd {
  local a=$1 b=$2 c
  while (( a % b != 0 )) ; do
    c=$a ; a=$b ; (( b = c % b ))
  done
  echo "$b"
}

function choc {
  local n=$1 m=$2 d=$( gcd "$1" "$2" )
  echo $(( n / d ))
}

choc 10 4
```

The [common prime divisors]() is interesting because of its `reduce` function. GCD of two numbers should be all you need in order to reduce the numbers. So given initial value `a` and `gcd`, the `a / gcd` should be a product of divisors of the very same `gcd`. So we extract common part from `a / gcd` and `gcd` and reduce `a / gcd` by this amount. We proceed till either the number is reduced to 1 or cannot be reduced any further.

```bash
function gcd {
  local a=$1 b=$2 c
  while (( a % b != 0 )) ; do
    c=$a ; a=$b ; (( b = c % b ))
  done
  echo "$b"
}

function reduce {
  local a=$1 g=$2 f d r=1
  # first division
  (( f = a / g ))
  # need to be reduced as f is not a divisor of g
  while (( g % f != 0 )) ; do
    # find the amount the f needs to be reduced by
    d=$( gcd "$g" "$f" )
    # check if can be reduced
    (( d == 1 )) && { r=0 ; break ; } || (( f /= d ))
  done
  echo "$r"
}

function cpd {
  local i a b g p q v
  declare -a r
  for i in "$@" ; do
    r=(${i//:/ }) ; a="${r[0]}" ; b="${r[1]}"
    g=$( gcd "$a" "$b" )
    p=$( reduce "$a" "$g" ) ; q=$( reduce "$b" "$g" )
    (( p == 1 )) && (( q == 1 )) && (( v++ ))
  done
  echo "$v"
}

cpd 15:75 10:30 3:5 22:7744 44:23232
```

The [ladder](https://codility.com/programmers/task/ladder/) is a counting exercise which is just implementation of Fibonacci numbers. Pretty much the only complexity is to understand that the number of paths leading to `n`-th rung is `fib(n)`.

```bash
declare -a FIB=(1 1 2)
declare FIB_I=2

function fib {
  local n=$1 f
  while (( FIB_I < n )) ; do
    (( FIB_I++ ))
    (( f = FIB[FIB_I - 2] + FIB[FIB_I - 1] ))
    FIB+=($f)
  done
  echo "${FIB[$n]}"
}

function lad {
  declare -a r
  local a b i f d
  for i in "$@" ; do
    r=(${i//:/ }) ; a="${r[0]}" ; b="${r[1]}"
    f=$( fib "$a" )
    d=$( echo "2^$b" | bc )
    echo $(( f % d ))
  done
}

lad 4:3 4:2 5:4 5:3 1:1
```

The [Fibonacci frog](https://codility.com/programmers/task/fib_frog/) is actually dynamic programming exercise, which I was too lazy to implement just yet. I am traversing all possible paths instead, which is very bad. When I reach the same point `c` on two different paths, I am computing the value function at point `c` twice. Nobody should ever do that. Learnings are that writing recursive functions in bash is pain, probably even more pain than writing iterative versions with manual stack handling.

```bash
declare -a FIB=(1 2)
declare FIB_I=1

# generate fibonacci numbers up to n
function fib {
  local n=$1 f
  while (( FIB[FIB_I] <= n )) ; do
    (( FIB_I++ ))
    (( f = FIB[FIB_I - 2] + FIB[FIB_I - 1] ))
    FIB+=($f)
  done
}

function jmp {
  # c - current position
  # d - next position
  # f - iterator over fibonacci numbers
  # m - length of min path, -1 if no path from current point
  # v - the length of sub path starting from d = c + f
  local c=$1 d f m v
  shift
  # n is the length of the padded array
  local n=$#
  declare -a as=("$@")

  # check if we reached the end
  if (( c == n - 1 )) ; then
    echo "0"
  else
    m=-1
    for f in "${FIB[@]}" ; do
      # check if jumping over f is a valid point
      (( d = c + f ))
      (( d < n )) || continue
      (( as[d] == 1 )) || continue
      # find the optimal path length starting from d
      v=$( jmp $d "$@" )
      # update minimum
      if (( v >= 0 )) ; then
        if (( m == -1 )) || (( v + 1 < m )) ; then
          (( m = v + 1 ))
        fi
      fi
    done
    echo "$m"
  fi
}

function frog {
  # pre-compute fibonacci numbers
  fib "$#"
  # pad the array with terminal 1 to denote terminal position
  jmp -1 "$@" 1
}

frog 0 0 0 1 1 0 1 0 0 0 0
```

Enough.
