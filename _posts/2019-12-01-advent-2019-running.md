---
layout: post
title: "Advent of Code 2019: Commentary"
date: 2019-12-01
permalink: /advent-of-code-2019-commentary.html
---

Here's my running commentary on [Advent of Code
2019](https://adventofcode.com/2019), for which I'm using Go. The full code is
available on [GitHub](https://github.com/dhconnelly/advent-of-code-2019).

## Table of Contents

[[Day 1]](#day-1) [[Day 2]](#day-2) [[Day 3]](#day-3) [[Day 4]](#day-4)
[[Day 5]](#day-5) [[Day 6]](#day-6) [[Day 7]](#day-7) [[Day 8]](#day-8)
[[intcode refactoring]](#intcode-refactoring) [[Day 9]](#day-9)
[[intcode refactoring round 2]](#intcode-refactoring-round-2)
[[Day 10]](#day-10) [[Day 11]](#day-11) [[Day 12]](#day-12)
[[Day 13]](#day-13) [[Day 14]](#day-14) [[Day 15]](#day-15)
[[Day 16]](#day-16) [[Day 17]](#day-17)

## Day 1

Both parts were straightfoward. Each line of the input is just an integer, so
we iterate over each line and convert it. For part 1 we just apply the
fuelToMass function to each value and sum the total, while for part 2 we
repeatedly apply fuelToMass until it becomes negative.

```
func fuelForMass(mass int) int {
  return (mass / 3) - 2
}

func fuelSum(mass int) int {
  sum := 0
  for {
    fuel := fuelForMass(mass)
    if fuel < 0 {
      break
    }
    sum += fuel
    mass = fuel
  }
  return sum
}
```

I spent more time deciding how to structure the program to do the scanning and
parsing and so on in a reusable manner. Here's a place where generics would be
nice:

```
func scanLines(r io.Reader) <-chan int {
  ch := make(chan int)
  go func() {
    scan := bufio.NewScanner(r)
    for scan.Scan() {
      i, err := strconv.Atoi(scan.Text())
      if err != nil {
        log.Fatal(err)
      }
      ch <- i
    }
    if err := scan.Err(); err != nil {
      log.Fatal(err)
    }
    close(ch)
  }()
  return ch
}
```

(The error handling is fine because this isn't a general-purpose library,
in which case the error should be returned the the caller, and I want the
program to die if it fails to process the input.)

`scanLines` would be reusable for each problem if it could take an argument
of type `func (string) T` to parse each line, where the generic type `T` would
also parameterize the output channel: `<-chan T`. As it is, without generics,
I could either replace `T` with `interface{}`, or just do something else. One
solution that's less bad than using `interface{}` is just passing the entire
line, and letting each problem define a channel transformation function around
a lines channel:

```
func scan(in <-chan string) <-chan int {
  ch := make(chan int)
  go func() {
    for line := range in {
      i, err := strconv.Atoi(line)
      if err != nil {
        log.Fatal(err)
      }
      ch <- i
    }
    close(ch)
  }()
  return ch
}
```

But this is as much code as scanLines itself, nearly all of it the same, which
just ends up inflating the size of the program. In the end I opted to avoid the
abstraction and bundle scanning with parsing together. It's less than twenty
lines of code, and maybe a better abstraction will appear later.

**Edit**: I looked at the [AoC
subreddit](https://www.reddit.com/r/adventofcode/comments/e4axxe/2019_day_1_solutions/)
and everyone writes very short solutions! So I wrote a shorter, less modular
version that uses `fmt.Fscanf` instead of `bufio.Scanner` with `fuelForMass`
and `fuelSum` as defined above:

```
func main() {
  f, err := os.Open(os.Args[1])
  if err != nil {
    log.Fatal(err)
  }
  defer f.Close()
  sum, recSum := 0, 0
  var mass int
  for {
    if _, err := fmt.Fscanf(f, "%d\n", &mass); err == io.EOF {
      break
    } else if err != nil {
      log.Fatal(err)
    }
    sum += fuelForMass(mass)
    recSum += fuelSum(mass)
  }
  fmt.Println(sum)
  fmt.Println(recSum)
}
```

Full code is
[here](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day1/).


## Day 2

Reading the input was a bit different today, since it was comma-delimited
instead of line-delimited, but I suppose one could have done `s/,/\n/g` and used
line-reading logic. I read the entire file into memory and split around
commas, but defining a `bufio.SplitFunc` and using `bufio.Scanner` would be
necessary for inputs that don't fit into memory. After that, you just convert
each token to an integer to get the data:

```
func readData(path string) []int {
  txt, err := ioutil.ReadFile(path)
  if err != nil {
    log.Fatal(err)
  }
  toks := strings.Split(string(txt[:len(txt)-1]), ",")
  data := make([]int, len(toks))
  for i, tok := range toks {
    val, err := strconv.Atoi(tok)
    if err != nil {
      log.Fatal(err)
    }
    data[i] = val
  }
  return data
}
```

Running the computer is just a for-loop with a switch statement. Make sure
you're double-indexing into the data, though, since the arguments to the
opcode specify *addresses* and not the argument values themselves:

```
func execute(data []int) {
  for i := 0; i < len(data); i += 4 {
    switch data[i] {
    case 1:
      data[data[i+3]] = data[data[i+1]] + data[data[i+2]]
    case 2:
      data[data[i+3]] = data[data[i+1]] * data[data[i+2]]
    case 99:
      return
    default:
      log.Fatalf("undefined opcode at position %d: %d", i, data[i])
    }
  }
}
```

This is enough to solve part 1. Note that, as mentioned in the problem text,
for future problems we'll want to modify the loop to use an opcode-specific
increment.

For part 2 I wrapped execution into a function to create a local copy of the
data to avoid reusing memory from previous attempts, as required:

```
func executeWith(data []int, noun, verb int) int {
  local := make([]int, len(data))
  copy(local, data)
  local[1], local[2] = noun, verb
  execute(local)
  return local[0]
}
```

To find the answer I just brute-force searched for the noun and verb. An
algorithmically cleverer solution is not obvious to me, but since the search
is embarrassingly parallel, dividing the search space and launching some
goroutines might speed it up. Quitting early when one of the goroutines found
the answer could be implemented with a second `done` channel. I might come
back later and implement that, but for now here's the rest of the code:

```
func main() {
  data := readData(os.Args[1])

  // part 1
  fmt.Println(executeWith(data, 12, 2))

  // part 2
  for noun := 0; noun <= 99; noun++ {
    for verb := 0; verb <= 99; verb++ {
      result := executeWith(data, noun, verb)
      if result == 19690720 {
        fmt.Println(100*noun + verb)
        return
      }
    }
  }
}
```

Full code and inputs and outputs are
[here](https://github.com/dhconnelly/advent-of-code-2019/tree/master/day2).

**Edit**: Some folks on Reddit
([1](https://www.reddit.com/r/adventofcode/comments/e4u0rw/2019_day_2_solutions/f9fm5tv/)
[2](https://www.reddit.com/r/adventofcode/comments/e4u0rw/2019_day_2_solutions/f9fljyn/)
among others) came up with clever solutions to part 2 that don't use brute
force! One used a contraint solver, which I've not seen since college and
don't remember much about, and another used symbolic algebra. Shout out to
Eric, the creator, for not requiring these approaches but making them
possible! (He did a talk about creating Advent of Code that's worth watching.
Link is [here](https://www.youtube.com/watch?v=bS9882S0ZHs))


## Day 3

The input is essentially two lists of vectors (magnitude and direction). I
wasted a bunch of time switching between parsing with `ioutil.ReadFile` and
`strings.Split` vs. `bufio.Scanner` and `bufio.Reader`, but the former used
fewer lines so I kept it. I'll omit it here because it's uninteresting, but
the full code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day3/day3.go).

Once we have the input, we need to convert it into a list of coordinates along
the paths. We'll then find intersections along the two paths. The
vector-path-to-coord-path conversion is pretty verbose; without gofmt I'd just
collapse all the switch case branches into single lines.

```
type vec struct {
  dir  rune
  dist int
}

type coord struct {
  x, y int
}

func toPath(vecPath []vec) []coord {
  var path []coord
  var cur coord
  for _, v := range vecPath {
    var dim *int
    d := 0
    switch v.dir {
    case 'U':
      dim = &cur.y
      d = +1
    case 'D':
      dim = &cur.y
      d = -1
    case 'R':
      dim = &cur.x
      d = +1
    case 'L':
      dim = &cur.x
      d = -1
    default:
      log.Fatalf("bad direction: %s", v.dir)
    }
    for n := v.dist; n > 0; n-- {
      *dim += d
      path = append(path, cur)
    }
  }
  return path
}

```

**Edit**: Somebody on the
[subreddit](https://www.reddit.com/r/adventofcode/comments/e5bz2w/2019_day_3_solutions/f9iz68s/?utm_source=share&utm_medium=ios_app&utm_name=iossmf)
used a map of direction character to x and y diffs, which
reduces the lines of code by quite a bit.

Finding intersecting points and choosing the one that's closest to the
starting point (i.e. closest to 0,0) isn't bad. I initially wasted time and
lines of code on handling more than two wires, since I wasn't sure what would
come in part 2. I cleaned it up a bit after finding the answer, since handling
just two paths requires fewer loops and avoids some slices. I should have done
that from the beginning, actually, since it's not hard to refactor from two to
N parameters, and "maybe I'll need it later" is a terrible reason to add
abstraction. I woke up at 5 to do this in bed in the dark, though, and I just
didn't think clearly enough about it.

To intersect the two coordinate paths, we dump the first one into a
`map[coord]bool` representing a set, and then look for all the coordinates in
the second path that are in that set.

```
func findIntersects(coords1, coords2 []coord) []coord {
  coords := make(map[coord]bool)
  for _, c := range coords1 {
    coords[c] = true
  }
  var intersections []coord
  for _, c := range coords2 {
    if coords[c] {
      intersections = append(intersections, c)
    }
  }
  return intersections
}
```

Finding the closest intersection is just finding the one with the smallest
magnitude.

```
func dist(c coord) int {
  return int(math.Abs(float64(c.x)) + math.Abs(float64(c.y)))
}

func closestIntersect(intersects []coord) int {
  var closestDist int
  for _, coord := range intersects {
    if d := dist(coord); closestDist == 0 || d < closestDist {
      closestDist = d
    }
  }
  return closestDist
}
```

For part 2, we need to find how many steps it took each path to reach each
intersection point. Since the path coordinates are ordered according to the
order in which they were visited, we can just iterate to find the index of the
intersection point. Then we iterate over each intersection and add those step
counts for the two paths:

```
func stepsTo(to coord, path []coord) int {
  for i, cur := range path {
    if cur == to {
      return i + 1
    }
  }
  return 0
}

func fastestIntersect(coords []coord, path1, path2 []coord) int {
  var speed int
  for _, c := range coords {
    sum := stepsTo(c, path1) + stepsTo(c, path2)
    if speed == 0 || sum < speed {
      speed = sum
    }
  }
  return speed
}
```

That's it! Full code is
[here](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day3/day3.go).

**Edit**: I went back tonight and extracted libraries for [2d
geometry](https://github.com/dhconnelly/advent-of-code-2019/blob/master/geom/point.go),
[integer
math](https://github.com/dhconnelly/advent-of-code-2019/blob/master/ints/ints.go),
and the [intcode
machine](https://github.com/dhconnelly/advent-of-code-2019/blob/master/intcode/intcode.go).
Hopefully this will come in handy later :)


## Day 4

This was much more straightforward than the last two days. Loop over every
number in the given range and test whether it's valid:

```
func countValidPasswords(from, to int) (int, int) {
  numValid1, numValid2 := 0, 0
  for i := from; i < to; i++ {
    p := toPassword(i)
    valid1, valid2 := valid(p)
    if valid1 {
      numValid1++
    }
    if valid2 {
      numValid2++
    }
  }
  return numValid1, numValid2
}
```

We need access to the individual digits in order to check validity:

```
func toPassword(x int) [6]byte {
  var p [6]byte
  for i := 5; i >= 0; i-- {
    p[i] = byte(x % 10)
    x /= 10
  }
  return p
}
```

For checking part 1 validity, we only need to look at one digit at a time and
its neighbor to the right. We can stop immediately if we find a decreasing
digit, and all we have to track is whether we've already seen a repeat.

```
func valid(p [6]byte) bool {
  twoAdjacentSame := false
  for i := 0; i < len(p)-1; i++ {
    if p[i] > p[i+1] {
      return false, false
    }
    if p[i] == p[i+1] {
      twoAdjacentSame = true
    }
  }
  return twoAdjacentSame
}
```

Adding the part 2 condition that there's a repeating group of length *only*
two means we need to keep track of the length of the current group in addition
to tracking whether the condition has already been satisfied. Since we can
only know if the repeating group is length 2 after exiting it, we have to
check it in the loop as well as after exiting:

```
func valid(p [6]byte) (bool, bool) {
  twoAdjacentSame, onlyTwoAdjacentSame := false, false
  matchLen := 1
  for i := 0; i < len(p)-1; i++ {
    if p[i] > p[i+1] {
      return false, false
    }
    if p[i] == p[i+1] {
      twoAdjacentSame = true
      matchLen++
    } else if matchLen == 2 {
      onlyTwoAdjacentSame = true
    } else {
      matchLen = 1
    }
  }
  return twoAdjacentSame, onlyTwoAdjacentSame || matchLen == 2
}
```

Full code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day3/day4.go).

Lots of people on
[Reddit](https://www.reddit.com/r/adventofcode/comments/e5u5fv/2019_day_4_solutions/)
used either a regular expression or sorting+sets for validation. My validation
algorithm is O(n) in the length of the password, whereas sorting a string and
setifying it for each candidate is O(n log n). For small inputs this doesn't
matter, of course, and doing something clever produces much shorter code than
mine: mine takes 18 lines to validate. The regex idea is interesting but
produces nontrivial regexes with lots of captures, and I personally avoid
writing nontrivial regexes since they can be pretty opaque.

I have to say, looking at all the Python solutions makes me jealous of Python
for this kind of problem-solving. I find it hard to imagine a readable Go
implementation of any of these so far that's less than ~70 lines or so,
whereas the Python implementations I see are pretty consistently ~10 lines or
less. On the other hand, my Go solutions appear (to me, of course) very
explicit and well-factored. This could be a result of the language forcing
such a style, or it could be my lack of experience with code golfing and
the impact of (too much? :) professional software development.


## Day 5

This is deeply satisfying:

```
day5 $ time ./day5 input.txt 1 5
9025675
11981754

real    0m0.006s
user    0m0.000s
sys     0m0.006s
```

Today's problem involved extending the intcode computer from [Day 2](#day-2),
but I just decided to write again from scratch. That code made certain
assumptions that were violated by today's problem (particularly read modes and
input/output), and I had a clear enough idea about how to proceed that I felt
it would be faster to write from scratch than refactor. I'm keeping it as a
post-AoC TODO to come back and unify all of the implementations.

Reading the input is just a string split on commas and mapping `strconv.Atoi`,
so I'll omit that and jump to instruction parsing. First, how to represent
modes (unneccessary actually, could just store a byte) and instructions:

```
type mode int

const (
  POS mode = iota
  IMM
)

type instr struct {
  opcode int
  params int
  modes  []mode
}
```

To parse an `instr`, we split the instruction integer into the opcode and
modes parts using `i%100` and `i/100`, then pull off individual mode bits one
at a time. This resembles the password validation from [Day 4](#day-4):

```
func parseInstr(i int) instr {
  var in instr
  in.opcode = i % 100
  in.params = opcodeToParam[in.opcode]
  for i /= 100; len(in.modes) < in.params; i /= 10 {
    in.modes = append(in.modes, mode(i%10))
  }
  return in
}
```

Notice how we handle the leading zeroes: since repeated division of zero is
zero, which is indeed the desired mode for a dropped leading zero, we just
keep pulling off leading zeroes for as many parameters are expected for the
given opcode. We keep the number of parameters expected for each opcode in a
map:

```
var opcodeToParam = map[int]int{
  1: 3, 2: 3, 3: 1, 4: 1, 5: 2, 6: 2, 7: 3, 8: 3, 99: 0,
}
```

So now we can parse an instruction from an integer in the data. Before
proceeding to execution, let's talk about retrieving opcode arguments now that
we have two parameter modes (immediate and position). I extracted lookup into
a function that handles this so it doesn't further clutter the execution
logic:

```
func get(data []int, i int, m mode) int {
  v := data[i]
  switch m {
  case POS:
    return data[v]
  case IMM:
    return v
  }
  log.Fatalf("unknown mode: %d", m)
  return 0
}
```

Pretty straightforward. This is used during program execution, implemented
again (as in day 2) using a loop and a switch over opcodes. Getting pretty
long now; if we keep adding opcodes, it it would be worth generalizing a bit.

Let's look at the code first, and then I'll explain a bit more.

```
func run(data []int, in <-chan int, out chan<- int) {
  for i := 0; i < len(data); {
    instr := parseInstr(data[i])
    switch instr.opcode {
    case 1:
      l := get(data, i+1, instr.modes[0])
      r := get(data, i+2, instr.modes[1])
      s := data[i+3]
      data[s] = l + r
      i += instr.params + 1
    case 2:
      l := get(data, i+1, instr.modes[0])
      r := get(data, i+2, instr.modes[1])
      s := data[i+3]
      data[s] = l * r
      i += instr.params + 1
    case 3:
      s := data[i+1]
      data[s] = <-in
      i += instr.params + 1
    case 4:
      v := get(data, i+1, instr.modes[0])
      out <- v
      i += instr.params + 1
    case 5:
      l := get(data, i+1, instr.modes[0])
      r := get(data, i+2, instr.modes[1])
      if l != 0 {
        i = r
      } else {
        i += instr.params + 1
      }
    case 6:
      l := get(data, i+1, instr.modes[0])
      r := get(data, i+2, instr.modes[1])
      if l == 0 {
        i = r
      } else {
        i += instr.params + 1
      }
    case 7:
      l := get(data, i+1, instr.modes[0])
      r := get(data, i+2, instr.modes[1])
      s := data[i+3]
      if l < r {
        data[s] = 1
      } else {
        data[s] = 0
      }
      i += instr.params + 1
    case 8:
      l := get(data, i+1, instr.modes[0])
      r := get(data, i+2, instr.modes[1])
      s := data[i+3]
      if l == r {
        data[s] = 1
      } else {
        data[s] = 0
      }
      i += instr.params + 1
    case 99:
      close(out)
      return
    }
  }
}
```

The logic for opcodes 1 and 2 is the same as in day 2. For input/output in
opcodes 3 and 4 I used channels: the machine takes a read-only channel for
getting input and a write-only channel for emitting output. This completely
abstracts the implementation details from the program execution logic. 

For part 2, the opcode 5 and 6 logic for jumping is relatively straightforward
(retrieve the arguments, compare, then update the program counter
appropriately), and the opcode 7 and 8 compare-and-store logic is similar to
opcodes 1 and 2.

Running the machine now requires copying the input data, setting up the
channels, and starting execution in a goroutine. We go ahead and send the
single input value on a buffered channel so it doesn't block, then read and
discard all output values until the last one, which is returned as the final
result.

```
func execute(data []int, input int) int {
  data = copied(data)
  in, out := make(chan int, 1), make(chan int)
  in <- input
  go run(data, in, out)
  var o int
  for o = range out {
  }
  close(in)
  return o
}
```

During debugging it was useful to print the outputs, and a nicer
implementation of the machine would support this with a flag, but I removed
the logging when I had the answers to both parts.

Full code is
[here](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day5/day5.go).

Now that we have branching and jumping, the machine should now be [Turing
complete](https://en.wikipedia.org/wiki/Turing_completeness), i.e. it should
be able to do anything that any given programming language can do. In
particular, one could write a C compiler (or Go, or Haskell, etc) that emits
this intcode to be executed by our machine. I'm sure someone on Reddit will do
this in the coming days and weeks :)

**Edit**: It took less than one day:
[Intscript](https://www.reddit.com/r/adventofcode/comments/e6q9f6/2019_day_5_intscript_a_tiny_language_that/)


# Day 6

This was my quickest puzzle since day 1, about 40 minutes to both answers. So
far it's taken me between an hour and 1:20 or so to finish each day,
regardless of difficulty, of which I think a large part is reading data and
building the right data structures. This required only one data structure,
though, and parsing it was simple -- not even any `strconv.Atoi` today.

```
type orbit struct {
  orbiter, orbited string
}
```

Both problems needed a graph representation to store links between orbiter and
orbited. Using a real graph might have had some nice properties, but I used a
map since it's so easy to look up and iterate, whereas finding or writing a
real graph structure would have required either learning its API or building
one. This is one nice property of using a standard library / built-in language
features over dependencies: usage patterns are the ones you already know.

```
func orbitMap(orbits []orbit) map[string]string {
  m := make(map[string]string)
  for _, o := range orbits {
    m[o.orbiter] = o.orbited
  }
  return m
}
```

To count orbits for any object `k`, we need to count orbits for the object it
is orbiting `v`, and so on recursively until reaching an object that orbits
nothing. I used iteration here instead of explicit recursion, again because
it's so straightforward with a for loop over the map:

```
func chain(k string, m map[string]string) []string {
  var chain []string
  for v, ok := m[k]; ok; v, ok = m[v] {
    chain = append(chain, v)
  }
  return chain
}

func countOrbits(orbits map[string]string) int {
  n := 0
  for k, _ := range orbits {
    n += len(chain(k, orbits))
  }
  return n
}
```

(Note that we could cache the transitive chains as we build them, instead of
visiting the entire chain for each object. See: [dynamic
programming](https://en.wikipedia.org/wiki/Dynamic_programming#Computer_programming))

So we look at each orbit, find the chain of all indirectly orbited objects,
and sum up the chain lengths.

For part 2, we want to find the orbit chains for `YOU` and `SAN`, find the
object `o` that is closest to both of them, and return the sum of the distance
from each of `YOU` and `SAN` to `o`:

```
func closestAncestor(chain1, chain2 []string) (string, int, int) {
  m := make(map[string]int)
  for i, k := range chain1 {
    m[k] = i
  }
  for i, o := range chain2 {
    if j, ok := m[o]; ok {
      return o, i, j
    }
  }
  return "", 0, 0
}
```

First we dump all of the first chain in a map to keep track of the distances
for each object in it, then look at the second chain and find the first object
that's in the map. That's the closest ancestor, and we return its index in the
chain as the second distance and its value in the map as the first distance.

This works because the chains are sorted by distance from the starting object
(see the function `chain` above), so the first object that we visit in the
second chain that's in the first one will always be closest than any other
ancestor: any closer ancestor for the second chain would have been visited
earlier in the loop over the second chain.

The number of required transfers is just the sum of these two distances:

```
func transfers(orbits map[string]string, from, to string) int {
  fromChain, toChain := chain(from, orbits), chain(to, orbits)
  _, dist1, dist2 := closestAncestor(fromChain, toChain)
  return dist1 + dist2
}
```

That's it! Code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day6/day6.go).


## Day 7

Well, today I wasted half an hour on how to generate permutations. I misread
and started by generating with replacement (i.e. 44444 is valid), and then,
since this produced signals that didn't match the examples, I spent a while
trying to debug the execution logic. Anyway, after adapting the code to
properly generate without replacement, I couldn't figure out how to detect
that all permutations had been generated and the output channel could be
closed. Eventually I gave up and closed it manually after n! values had been
produced. This could surely be much cleaner. I look forward to reading the
soultions where they produce all of these in a single line of Python...

```
func fact(n int) int {
  if n < 1 {
    return 1
  }
  return n * fact(n-1)
}

func genSeq(s *seq, i int, avail map[int]bool, out chan<- seq) {
  if i >= 5 {
    out <- *s
    return
  }
  for phase, free := range avail {
    if free {
      avail[phase] = false
      s[i] = phase
      genSeq(s, i+1, avail, out)
      avail[phase] = true
    }
  }
}

func genSeqs(phases []int) chan seq {
  out := make(chan seq)
  var s seq
  go func() {
    ch := make(chan seq)
    avail := availMap(phases)
    go genSeq(&s, 0, avail, ch)
    for i := 0; i < fact(len(s)); i++ {
      s := <-ch
      out <- s
    }
    close(out)
  }()
  return out
}
```

Essentially, to generate sequences of length n, we choose a number that's not
already been used (tracked in a `map[int]bool`), generate sequences of length
n-1, and use recursion. We return the values in a channel because I wanted to
be able to do `for seq := range genSeqs()`. I think this would be much cleaner
if I were to just generate one giant slice of all the permutations that could
be modified recursively and returned. Anyway, lots of channel usage in this
problem, which was new for me -- I've not really used them much until now.

For machine execution, I adapted the input/output so that each execution
takes and returns a channel. Then we can send and receive an arbitrary number
of signals, which helps in part 2. The only trickiness is appending the phase
setting to the front of the input channel.

```
func execute(data []int, phase int, signals <-chan int) chan int {
  data = copied(data)
  in, out := make(chan int), make(chan int)
  go func() {
    in <- phase
    for signal := range signals {
      in <- signal
    }
    close(in)
  }()
  go run(data, in, out)
  return out
}
```

Now we can execute with a given phase sequence by kicking off the five
amplifiers by using the output value from one amplifier as the input to the
next amplifier. The output from the final amplifier's output channel is the
generated signal.

```
func executeSeq(data []int, s seq) int {
  out := 0
  for _, phase := range s {
    in := make(chan int, 1)
    in <- out
    close(in)
    out = <-execute(data, phase, in)
  }
  return out
}
```

To find the max signal, we iterate over all sequences using `genSeqs` and
track the largest signal produced so far. This implementation takes the
execution function as a parameter, which we need for part 2.

```
func maxSignal(data []int, exec func([]int, seq) int, nums []int) int {
  max := 0
  for seq := range genSeqs(nums) {
    out := exec(data, seq)
    if out > max {
      max = out
    }
  }
  return max
}
```

Okay, so part 2 requires looping the inputs and outputs. This is pretty
straightforward using our channel mechanism: instead of piping the single
output value produced by each amplifier into the next one, we send the entire
output channel from one amplifier as the input channel to the next one. Then,
instead of returning the first value produced by the final amplifier, we
save a copy of it and pipe it back into the first amplifier until the channel
is closed (which happens when the final amplifier halts). The last output
value we saw is the signal.

```
func executeWithFeedback(data []int, s seq) int {
  // send the initial input
  in := make(chan int, 1)
  in <- 0

  // pipe the amplifiers together
  out := in
  for _, phase := range s {
    out = execute(data, phase, out)
  }

  // pipe the output back into the input, but keep track of it
  var o int
  for o = range out {
    in <- o
  }

  // last output is the output signal
  return o
}
```

This function is used for a second call to `maxSignal` as defined above, since
the looping over sequences and tracking max signal logic is identical.

```
func main() {
  data := read(os.Args[1])
  fmt.Println(maxSignal(data, executeSeq, []int{0, 1, 2, 3, 4}))
  fmt.Println(maxSignal(data, executeWithFeedback, []int{5, 6, 7, 8, 9}))
}
```

This one was pretty difficult, although I think if I had more experience with
channels (and hadn't misread the sequence requirements) it would have been
much more straightforward, because using channels for the input and output
really makes it easy to hook the machines together and run them concurrently.

All of the actual intcode machine logic is identical to [Day 5](#day-5). Full
code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day7/day7.go).

**Edit**: I came back and replaced the terrible, complex channel-based
generation algorithm with a simple recursive one that builds a straightforward
slice of the sequences. Here it is:

{% raw %}
```
func genSeqsRec(nums []int, used map[int]bool, until int) []seq {
  if until == 0 {
    return []seq{{}}
  }
  var seqs []seq
  for _, num := range nums {
    if !used[num] {
      used[num] = true
      for _, recSeq := range genSeqsRec(nums, used, until-1) {
        recSeq[until-1] = num
        seqs = append(seqs, recSeq)
      }
      used[num] = false
    }
  }
  return seqs
}

func genSeqs(nums []int) []seq {
  return genSeqsRec(nums, make(map[int]bool), len(nums))
}
```
{% endraw %}

Essentially, we choose a phase setting and mark it unavailable, then generate
all phases sequences of length n-1 without the phase we removed, then add the
one we removed to the end of all the recursively-generated sequences. We do
this for each phase setting. Should have started there, but got a bit
channel-ambitious :)


## Day 8

A nice and easy one today, even though I spent way too much time on reading
and parsing the input, just like every day. These problems are great practice
for this, though. One thing I want to do is actually move away from using
`ioutil.ReadFile` and do proper scanning, which today I started to do again
before realizing I was spending a lot of time on it. In retrospect, who
cares? Well, for one, my daughter, who is awake and I'll need to get out of
her crib in fourteen minutes. Maybe I can come back to it later and refactor
to using byte scanning. At some point I'll have to actually spend the time,
otherwise I'll always be more comfortable with manipulating the entire thing
in memory.

Anyway, we first read in the pixel data layer-by-layer into an image
structure:

```
type image struct {
  width, height int
  layers        [][]byte
}
```

To compute the "checksum", we iterate over the layers, find the one that has
the fewest zeroes, and multiply the number of ones and twos together. Here's
a place where Go is just depressingly overboard-verbose.

```
func zeroes(pixels []byte) int {
  n := 0
  for _, p := range pixels {
    if p == 0 {
      n++
    }
  }
  return n
}

func layerChecksum(pixels []byte) int {
  ones, twos := 0, 0
  for _, p := range pixels {
    switch p {
    case 1:
      ones++
    case 2:
      twos++
    }
  }
  return ones * twos
}

func checksum(img image) int {
  fewestLayer := img.layers[0]
  fewestZeroes := zeroes(fewestLayer)
  x := layerChecksum(fewestLayer)
  for _, layer := range img.layers[1:] {
    if z := zeroes(layer); z < fewestZeroes {
      fewestZeroes = z
      fewestLayer = layer
      x = layerChecksum(layer)
    }
  }
  return x
}
```

This is just so much code. Some of it could be reduced by inlining the switch
statement into the checksum for-loop and including the zeroes in it, but we're
still talking about like twenty-ish lines of code at a minimum.

Anyway, enough whining, because yesterday's problem showed how Go can make
hard problems easy: from reading the subreddit it's clear that a lot of
people had trouble with running several copies of the intcode machine and
connecting them together, while for me this was an almost-trivial change
using channels and goroutines.

To decode the image, we can start at the bottom layer and move upwards, always
replacing a pixel if it's non-transpartent:

```
func apply(base, layer []byte) {
  for i := 0; i < len(base); i++ {
    if layerPix := layer[i]; layerPix != 2 {
      base[i] = layerPix
    }
  }
}

func decode(img image) []byte {
  b := make([]byte, img.width*img.height)
  for i := len(img.layers) - 1; i >= 0; i-- {
    apply(b, img.layers[i])
  }
  return b
}
```

Now we just need to print it:

```
func printImage(b []byte, width, height int) {
  for i := 0; i < height; i++ {
    for j := 0; j < width; j++ {
      pix := b[i*width+j]
      if pix == 0 {
        fmt.Printf(" ")
      } else {
        fmt.Printf("%d", b[i*width+j])
      }
    }
    fmt.Println()
  }
}
```

That's it. Code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day8/day8.go)
as usual :)


## Intcode refactoring

On Sunday night I decided to refactor my intcode implementation, which paid
off the next day. Let me go over what I did a bit before getting to the day 9
changes.

To begin with, I extracted machine state into a struct:

```
type machine struct {
  data []int
  in   <-chan int
  out  chan<- int
}

func newMachine(data []int, in <-chan int, out chan<- int) *machine {
  m := &machine{0, make([]data, len(data)), in, out}
  copy(m.data, data)
  return m
}
```

I also extracted get/set/read/write into methods:

```
func (m *machine) get(i int, md mode) int {
  v := m.data[i]
  switch md {
  case pos:
    return m.data[v]
  case imm:
    return v
  case rel:
    return m.data[v+m.relbase]
  }
  log.Fatalf("unknown mode: %d", md)
  return 0
}

func (m *machine) set(i, x int, md mode) {
  switch md {
  case pos:
    m.data[i] = x
  case rel:
    m.data[i+m.relbase] = x
  default:
    log.Fatalf("bad mode for write: %d", md)
  }
}

func (m *machine) read() int {
  return <-m.in
}

func (m *machine) write(x int) {
  m.out <- x
}
```

Then I defined enums for the opcodes:

```
type opcode int

const (
  add    opcode = 1
  mul           = 2
  read          = 3
  print         = 4
  jmpif         = 5
  jmpnot        = 6
  lt            = 7
  eq            = 8
  halt          = 99
)
```

This makes it easier to read the code and to refer to them in the arity map,
for example, and in the new handler map: I also moved opcode implementations
  out of the giant for-switch and into a map of handlers. For example:

```
type handler func(m *machine, pc int, instr instruction) (int, bool)

var handlers = map[opcode]handler{
  add: func(m *machine, pc int, instr instruction) (int, bool) {
    l := m.get(pc+1, instr.modes[0])
    r := m.get(pc+2, instr.modes[1])
    s := m.data[pc+3]
    m.set(s, l+r, instr.modes[2])
    return pc + instr.arity + 1, true
  },
}
```

The handler returns the updated program counter and a boolean indicating
whether the machine should continue running. So HALT is just:

```
halt: func(m *machine, pc int, instr instruction) (int, bool) {
  return 0, false
},
  ```

This simplifies the core execution logic, which now just adjusts the program
counter and invokes opcode handlers.

```
func (m *machine) run() {
  for pc, ok := 0, true; ok; {
    instr := parseInstruction(m.data[pc])
    if h, present := handlers[instr.op]; present {
      pc, ok = h(m, pc, instr)
    } else {
      log.Fatalf("bad instr at pos %d: %v", pc, instr)
    }
    if !ok {
      close(m.out)
    }
  }
}
```

The full code for the extracted intcode machine is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/intcode).
The package's public interface is:

```
$ go doc github.com/dhconnelly/advent-of-code-2019/intcode

package intcode // import "github.com/dhconnelly/advent-of-code-2019/intcode"

func ReadProgram(path string) ([]int, error)
func RunProgram(data []int, in <-chan int) <-chan int
```


## Day 9

Okay, so now we've finished implementing the intcode computer. I suspect we're
not done *using* it, but okay :) Today's problem was not bad for me:

- Support for large numbers just worked with no changes. Go's `int` type is
  machine-defined, though, so this may not work on a 32-bit machine, which
  would require switching to explicit int64 types.

- To support memory beyond the program, I initially started to add slice
  expansion logic into the machine's `get` and `set` methods. It then occurred
  to me that I could just use a `map[int]int` for memory instead of a slice. I
  looked through the usages of the data slice, decided that none of them
  required actual slice semantics (i.e. no appends and no explicit dependency
  on ordering or contiguous data), then simply switched to a map. This worked
  perfectly with no problems.

- Adding relative mode support involved adding a field to the `machine` struct
  and updating `get` and `set` to support that mode.

```
func (m *machine) get(i int, md mode) int {
  v := m.data[i]
  switch md {
  /* snip */
  case rel:
    return m.data[v+m.relbase]
  }
  log.Fatalf("unknown mode: %d", md)
  return 0
}

func (m *machine) set(i, x int, md mode) {
  switch md {
  /* snip */
  case rel:
    m.data[i+m.relbase] = x
  default:
    log.Fatalf("bad mode for write: %d", md)
  }
}

var handlers = map[opcode]handler{
  /* snip */
  adjrel: func(m *machine, pc int, instr instruction) (int, bool) {
    v := m.get(pc+1, instr.modes[0])
    m.relbase += v
    return pc + instr.arity + 1, true
  },
  /* snip */
}
```


This took me about a half an hour and produced correct answers to the examples
as well as both parts of the problem on the first try. I'm glad we didn't have
to implement all of this in a single day, and I'm also glad that it was spread
over several days so that I was able to think about the implementation and
possible improvements over the entire week. Even when it seems straightforward
to write a program, the right structure is usually not clear at the beginning.
It takes rewriting and thinking about it offline and talking about it and so
on before the right structure starts to appear.

Quick stats, since I was surprised that it didn't take my Chromebook longer
to run, given the warning that it may take a few seconds:

```
day9 $ time ./day9 input.txt 1 2
2494485073
44997

real    0m0.110s
user    0m0.103s
sys     0m0.016s
```

**Edit**: I came back and made a couple of small changes: first I switched to
explicit int64 types in the machine, so that big number handling isn't
machine-specific, and then I moved the program counter into the machine state
and moved updating it into the opcode handlers, which makes it easier to
inspect the machine state (in case we need -- or I want -- to do some sort of
single-instruction stepping or debugging.

Full code for the updated intcode machine is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/intcode),
as is the [Day 9-specific
code](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day9/day9.go)


## Intcode refactoring round 2

After talking with a colleague it was clear that the parameter modes for
writes can be tricky. My implementation works but is inconsistent in what
arguments should be provided to `get` and `set`, so I've refactored a bit and
added some comments. This affects
[machine.go](https://github.com/dhconnelly/advent-of-code-2019/blob/master/intcode/machine.go)
as well as its callers in
[opcodes.go](https://github.com/dhconnelly/advent-of-code-2019/blob/master/intcode/opcodes.go):

```
// Retrieves a value according to the specified mode.
//
// * In immediate mode, returns the value stored at the given address.
//
// * In position mode, the value stored at the address is interpreted
//   as a *pointer* to the value that should be returned.
//
// * In relative mode, the machine's current relative base is interpreted
//   as a pointer, and the value stored at the address is interpreted
//   as an offset to that pointer. The value stored at the *resulting*
//   address is returned.
//
func (m *machine) get(addr int64, md mode) int64 {
  v := m.data[addr]
  switch md {
  case pos:
    return m.data[v]
  case imm:
    return v
  case rel:
    return m.data[v+m.relbase]
  }
  log.Fatalf("unknown mode: %d", md)
  return 0
}

// Sets a value according to the specified mode.
//
// * In position mode, the value stored at the given address specifies
//   the address to which the value should be written.
//
// * In relative mode, the value stored at the given address specifies
//   an offset to the relative base, and the sum of the offset and the
//   base specifies the address to which the value should be written.
//
func (m *machine) set(addr, val int64, md mode) {
  v := m.data[addr]
  switch md {
  case pos:
    m.data[v] = val
  case rel:
    m.data[v+m.relbase] = val
  default:
    log.Fatalf("bad mode for write: %d", md)
  }
}
```

I think that's a bit clearer, but it changes the callers of `set` to provide
the simple argument location instead of dereferencing it first. For example:

```
var handlers = map[opcode]handler{
  /* snip */`
  mul: func(m *machine, instr instruction) bool {
    l := m.get(m.pc+1, instr.modes[0])
    r := m.get(m.pc+2, instr.modes[1])
    m.set(m.pc+3, l*r, instr.modes[2])
    m.pc += instr.arity + 1
    return true
  },
  /* snip */`
}
```


## Day 10

This took me three hours and I haven't had time to write up this post, but
I'll go ahead and link to the [code on
GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day10/day10.go).
I also wrote a cleaned-up version that uses angles instead of precomputed
integral diffs, which is
[here](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day10/day10_rf.go).

**Edit**: Okay, off work now, waiting to head over to our annual holiday party
and finally have time to write this up. To summarize: I made two bad choices
in the first ten minutes, struggled along with it for more than an hour and a
half, then ended up having to abandon it in part 2 anyway. Those choices were
(1) using integral (dx,dy) pairs instead of float64 slopes for finding line
intersections, and (2) trying to avoid an O(n^2) algorithm by precomputing
those possible pairs ahead of time. That is, instead of:

```
for each point1 in graph:
  for each point2 in graph:
    a = angle(point1, point2)
    if already visited a point with angle a from point1:
      if point2 is closer than the previous point:
        save point2 for angle a of point1
```

I was doing:

```
diffs = all possible (dx,dy) pairs
for each point in graph:
  for each diff in diffs:
    continue along diff from point until hitting a point
    store that point
```

The second approach looks deceptively simple, but all the complexity is in the
"all possible (dx,dy) pairs" and "continue along diff" lines. Look at all this
complexity just for finding the (dx,dy) pairs:

```
func allStepsFrom(g grid, from geom.Pt2, dx, dy int) []geom.Pt2 {
  var steps []geom.Pt2
  reachable := make(map[geom.Pt2]bool)
  for i := 0; i < g.height; i++ {
    for j := 0; j < g.width; j++ {
      add := false
      d := geom.Pt2{dx * j, dy * i}
      if d == geom.Zero2 {
        continue
      }
      for p := from.Add(d); inBounds(g, p); p = p.Add(d) {
        if !reachable[p] {
          add = true
          reachable[p] = true
        }
      }
      if add {
        steps = append(steps, d)
      }
    }
  }
  return steps
}

func allSteps(g grid) []geom.Pt2 {
  var steps []geom.Pt2
  steps = append(steps, allStepsFrom(g, geom.Pt2{0, g.height}, 1, -1)...)
  steps = append(steps, allStepsFrom(g, geom.Pt2{0, 0}, 1, 1)...)
  steps = append(steps, allStepsFrom(g, geom.Pt2{g.width, g.height}, -1, -1)...)
  steps = append(steps, allStepsFrom(g, geom.Pt2{g.width, 0}, -1, 1)...)
  return steps
}
```

So here we're starting at each of the four corners of the map and finding the
(dx,dy) pairs that result in being able to visit every point from each corner.

Needless to say, this also burned a ton of time in debugging. Now, once we
have all those steps computed, we visit each point and proceed along the step
lines:

```
func visit(visible map[geom.Pt2]int, g grid, p geom.Pt2, steps []geom.Pt2) {
  for _, d := range steps {
    var cur geom.Pt2
    for cur = p.Add(d); inBounds(g, cur) && !g.points[cur]; cur = cur.Add(d) {
    }
    if g.points[cur] {
      visible[p]++
    }
  }
}

func countVisible(g grid, steps []geom.Pt2) map[geom.Pt2]int {
  counts := make(map[geom.Pt2]int)
  for p, _ := range g.points {
    visit(counts, g, p, steps)
  }
  return counts
}
```

To find the "best" point for placing the space station, we just find the point
with the highest visible count:

```
func bestPoint(g grid, counts map[geom.Pt2]int) (geom.Pt2, int) {
  var best geom.Pt2
  count := 0
  for p, _ := range g.points {
    if c := counts[p]; c > count {
      best = p
      count = c
    }
  }
  return best, count
}
```

Part 2 required switching to angles anyway. For each vaporize loop around the
chosen space station position, we find all the points in the grid that are
visible using the steps that we already computed, and do this in a loop until
we've eliminated every asteroid position from the grid:

```
func vaporizeAll(g grid, from geom.Pt2, steps []geom.Pt2) []geom.Pt2 {
  ordered := make([]geom.Pt2, len(g.points))
  for vaporized := 0; vaporized < len(g.points)-1; {
    toVaporize := reachableFrom(g, from, steps)
    for _, p := range toVaporize {
      g.points[p] = false
      ordered[vaporized] = p
      vaporized++
    }
  }
  return ordered
}
```

We need the reachable points in increasing angle order starting from vertical,
though, and my angle calculation doesn't produce an angle of zero for points
directly above the starting point, so we have to (1st) find all reachable
points, (2nd) sort them by angle from the space station, and (3rd) reorder
them starting from the first one that has an angle greater than -pi/2:

```
type byAngleFrom struct {
  p  geom.Pt2
  ps []geom.Pt2
}

func (points byAngleFrom) Len() int {
  return len(points.ps)
}

func angle(p1, p2 geom.Pt2) float64 {
  return math.Atan2(float64(p2.Y-p1.Y), float64(p2.X-p1.X))
}

func (points byAngleFrom) Less(i, j int) bool {
  to1, to2 := points.ps[i], points.ps[j]
  a1 := angle(points.p, to1)
  a2 := angle(points.p, to2)
  return a1 <= a2
}

func (points byAngleFrom) Swap(i, j int) {
  points.ps[j], points.ps[i] = points.ps[i], points.ps[j]
}

func reachableFrom(g grid, from geom.Pt2, steps []geom.Pt2) []geom.Pt2 {
  var to []geom.Pt2
  for _, d := range steps {
    var p geom.Pt2
    for p = from.Add(d); inBounds(g, p) && !g.points[p]; p = p.Add(d) {
    }
    if g.points[p] {
      to = append(to, p)
    }
  }
  sort.Sort(byAngleFrom{from, to})
  var j int
  for j = 0; j < len(to) && angle(from, to[j]) < -math.Pi/2.0; j++ {
  }
  ordered := make([]geom.Pt2, len(to))
  for i := 0; i < len(to); i++ {
    ix := (j + i) % len(to)
    ordered[i] = to[ix]
  }
  return ordered
}
```

This is just too much. When I finally finished and got the right answer I knew
I had to come back and improve it. At work I talked it through with a
colleague, who mentioned normalizing angles using greatest common denominators
-- which addresses a concern I had, namely that storing points in a map by the
slope alone could have issues with floating point equality. Normalizing the
slopes using greatest common denominators addresses this: 3/15 and 5/25
produce the same slope, 1/5, and so turning it into a floating point number
and doing trigonometry on that, even with the loss of precision, will produce
one consistent angle.

So I came back at lunch and improved things drastically. First, avoiding all
the `allSteps` precomputation complexity, reachability from a given point is a
simple matter of iterating over every point, finding the angle from the
starting one, and storing it as reachable as long as that angle has never been
seen or it's closer than the previous one for that angle:

```
func angle(dy, dx int) float64 {
  g := ints.Abs(ints.Gcd(dx, dy))
  if g == 0 {
    return math.NaN()
  }
  return math.Atan2(float64(dy/g), float64(dx/g))
}

func reachable(g grid, from geom.Pt2) map[float64]geom.Pt2 {
  ps := make(map[float64]geom.Pt2)
  for p1, ok1 := range g.points {
    if !ok1 {
      continue
    }
    if p1 == from {
      continue
    }
    s := angle(p1.Y-from.Y, p1.X-from.X)
    if p2, ok2 := ps[s]; !ok2 || from.Dist(p1) < from.Dist(p2) {
      ps[s] = p1
    }
  }
  return ps
}
```

Finding the best location for the space station now involves picking the point
that has the most reachable points (we just ignore the angles for this):

```
func maxReachable(g grid) (geom.Pt2, int) {
  maxPt, max := geom.Zero2, 0
  for p, ok := range g.points {
    if !ok {
      continue
    }
    to := reachable(g, p)
    if l := len(to); l > max {
      max, maxPt = l, p
    }
  }
  return maxPt, max
}
```

Sorting by angle can avoid the sort.Interface implementation and just sort an
angle slice directly, since we have a map from angles to points, and then we
can easily retrieve the points in angle-sorted order as before:

```
func sortedByAngle(ps map[float64]geom.Pt2) []geom.Pt2 {
  sorted := make([]float64, 0, len(ps))
  for a := range ps {
    sorted = append(sorted, a)
  }
  sort.Float64s(sorted)
  var j int
  for j = 0; j < len(sorted) && sorted[j] < -math.Pi/2.0; j++ {
  }
  byAngle := make([]geom.Pt2, len(ps))
  for i := 0; i < len(sorted); i++ {
    a := sorted[(i+j)%len(sorted)]
    byAngle[i] = ps[a]
  }
  return byAngle
}
```

Now we can vaporize, as before, by repeatedly finding reachable asteroid
points from the space station and removing them from the grid in angle-order:

```
func vaporize(g grid, from geom.Pt2) []geom.Pt2 {
  vaporized := make([]geom.Pt2, len(g.points)-1)
  for i := 0; i < len(g.points)-1; {
    ps := reachable(g, from)
    sorted := sortedByAngle(ps)
    for _, p := range sorted {
      vaporized[i] = p
      g.points[p] = false
      i++
    }
  }
  return vaporized
}
```

This is a lot simpler: compare
[before](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day10/day10.go)
with
[after](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day10/day10_rf.go).

I think today was revenge for finishing day 9 in just a half an hour :)


## Day 11

Using all three of my extracted libraries today (intcode, ints, and geom) to
paint the tiles using the programmable robot. I started with some enums (not
really necessary and it inflates the code quite a bit, but who cares :) I'll
omit those here because they're clear below.

The core of the solution is an I/O loop, providing the intcode machine with
inputs according to the color of the tile it's on and reading the (color,
direction) outputs until the channel is closed. I store the current colors in
a `map[geom.Pt2]color` and the current position and orientation of the robot.

```
func run(data []int64, initial color) grid {
  in := make(chan int64)
  out := intcode.RunProgram(data, in)
  g := grid(make(map[geom.Pt2]color))
  p := geom.Zero2
  g[p] = initial
  o := UP
loop:
  for {
    select {
    case c, ok := <-out:
      if !ok {
        break loop
      }
      g[p] = color(c)
      dir := direction(<-out)
      o = turn(o, dir)
      p = move(p, o)
    case in <- int64(g[p]):
    }
  }
  return g
}
```

Channels are fun :)

Okay, so for part 1 we just need the number of tiles that were painted:

```
func main() {
  data, err := intcode.ReadProgram(os.Args[1])
  if err != nil {
    log.Fatal(err)
  }
  g := run(data, BLACK)
  fmt.Println(len(g))
}
```

Using the map makes this easy because it only contains values that were
explicitly written. Note that we kick off the machine with the color that
should be stored at position (0,0), where is the position we use for the
robot's initial tile. For part 1 we use `BLACK`.

For part 2 we kick it off with `WHITE` instead and print the resulting grid.
This requires finding the bounds of the space explored by the robot, after
which we can iterate over every position within those bounds and retrieve its
color from the map we created above. Since Go maps return a zero value when a
key isn't present, and the zero value for a color is `BLACK`, this is easy.

```
func printGrid(g grid) {
  minX, minY := math.MaxInt64, math.MaxInt64
  maxX, maxY := math.MinInt64, math.MinInt64
  for p, _ := range g {
    minX, maxX = ints.Min(minX, p.X), ints.Max(maxX, p.X)
    minY, maxY = ints.Min(minY, p.Y), ints.Max(maxY, p.Y)
  }
  for row := maxY; row >= minY; row-- {
    for col := minX; col <= maxX; col++ {
      p := geom.Pt2{col, row}
      switch g[p] {
      case BLACK:
        fmt.Print(" ")
      case WHITE:
        fmt.Print("X")
      }
    }
    fmt.Println()
  }
}
```

That's it :) Code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day11/day11.go).


## Day 12

Another revenge day for yesterday's easy one. I think it took me five hours.
The first part only took about a half hour, after which I gave part 2 a shot
without changing anything -- which did not work, of course. Let me walk
through part 1 before getting to part 2 and the mistakes I made and the
problems I ran into there.

Okay, so for part 1, each moon has position and velocity:

```
type moon struct {
  p, v geom.Pt3
}
```

We simulate the system by repeatedly stepping through the gravity and velocity
updates:

```
func step(ms []moon) {
  applyGravity(ms)
  applyVelocity(ms)
}

func simulate(ms []moon, n int) []moon {
  ms2 := make([]moon, len(ms))
  copy(ms2, ms)
  for i := 0; i < n; i++ {
    step(ms2)
  }
  return ms2
}
```

`applyGravity` is a bit verbose:

```
func applyGravity(ms []moon) {
  for i := 0; i < len(ms)-1; i++ {
    for j := i + 1; j < len(ms); j++ {
      applyGravityPair(&ms[i], &ms[j])
    }
  }
}

func applyGravityPair(m1, m2 *moon) {
  if m1.p.X < m2.p.X {
    m1.v.X, m2.v.X = m1.v.X+1, m2.v.X-1
  } else if m1.p.X > m2.p.X {
    m1.v.X, m2.v.X = m1.v.X-1, m2.v.X+1
  }
  if m1.p.Y < m2.p.Y {
    m1.v.Y, m2.v.Y = m1.v.Y+1, m2.v.Y-1
  } else if m1.p.Y > m2.p.Y {
    m1.v.Y, m2.v.Y = m1.v.Y-1, m2.v.Y+1
  }
  if m1.p.Z < m2.p.Z {
    m1.v.Z, m2.v.Z = m1.v.Z+1, m2.v.Z-1
  } else if m1.p.Z > m2.p.Z {
    m1.v.Z, m2.v.Z = m1.v.Z-1, m2.v.Z+1
  }
}
```

But velocity is simple:

```
func applyVelocity(ms []moon) {
  for i := range ms {
    ms[i].p.TranslateBy(ms[i].v)
  }
}
```

Finding the total system energy is just a loop, vector norms, and arithmetic:

```
func moonEnergy(m moon) int {
  return m.p.ManhattanNorm() * m.v.ManhattanNorm()
}

func energy(ms []moon) int {
  total := 0
  for _, m := range ms {
    total += moonEnergy(m)
  }
  return total
}
```

That's it for part 1.

```
func main() {
  ms := readPoints(os.Args[1])
  fmt.Println(energy(simulate(ms, 1000)))
}
```

Let me start part 2 by saying that my approach for part 2 was to track every
state that we've seen so far, in case the cycle we eventually find starts from
some non-initial state; that is, I thought that it could a while for all the
moons to settle into a rhythm, and the cycle would be some subset (s_m, s_m+1,
... s_n) of states from the entire chain (s_1, s2, ..., s_n). More on that
later.

When I tried to use this for part 2, it ran out of memory. Not surprising,
considering we were warned in the problem description that we'd need to come
up with a more efficient simulation. I was keeping each previous total system
state (a `[4]moon`) in a `map[[4]moon]bool`, but each moon is modeled as 6
`int64`s (three for each of position and velocity), which is 48 bytes per
moon, which is 192 bytes per state, and once we're tracking 4 billion states
(per test case 2), this is already about a terrabyte of data. Not feasible for
a single hashmap on a Chromebook.

My next idea was to try to write out the update equations and see
if there was some sort of trick. I noticed, for example, that

    pos(moon, n) = pos(moon, 0) + sum(vel(moon, k) for k = 1..n)
    vel(moon, n) = sum(diffs(moon, moons, k) for k = 1..n)

but this didn't help. Ideally this would lead to some sort of linear equation,
something to be solved with matrices, but the fact that the diffs depend on
`sign(pos(moon1) - pos(moon2))` makes this hard, I think. I don't think that
can be modeled with a linear equation.

After this I spent a bit of time trying to reduce the size of the state. If
every moon could fit into a single `int64`, for example, then the entire state
would only be 32 bytes, but this is still dominated by the need to potentially
store 4+ billion states: still >100 GB, still not feasible for my Chromebook.

I talked with a colleague of mine, Andrew, who is also doing Advent of Code.
He argued that we don't actually need to track every previous state, because a
cycle must necessarily return to the initial state. This wasn't clear to me,
but he was pretty certain about it and solved the problem with it, so I went
ahead and modified my code to assume this, hoping to patch it up later if
necessary. But now, instead of running out of memory, the program just sat and
churned. Adding some logging showed that there was no way it would solve even
the second test case in a reasonable amount of time.

It was at this point that Andrew pointed out that we can simulate each (x,y,z)
dimension independently. This is nice because it means we can use goroutines
and simulate on three cores instead of one -- and because each goroutine only
needs to find a cycle in *that dimension*, not all three, making the cycle
lengths much shorter. After making these changes, my program was able to find
the three dimensional cycle lengths in under 50ms -- a drastic difference!
As implemented, the per-dimension simulation never copies data that doesn't
fit into single machine words (as far as I can tell), as opposed to doing math
on `geom.Pt3` values that each copy three `int64`s when methods are called on
them for basic arithmetic. Additionally, finding the per-dimension cycles
simply requires drastically fewer steps -- in my case, by 10 or so
orders of magnitude fewer steps. This makes the problem tractable.

On to the code :)

First we flatten the state so that we can treat each coordinate's position and
velocity separately:

```
type state struct {
  px, py, pz, vx, vy, vz [4]int64
}
```

Each simulation step is pretty straightforward now:

```
func applyGravity(px, vx *[4]int64) {
  for i := 0; i < len(px)-1; i++ {
    for j := i + 1; j < len(px); j++ {
      if px[i] < px[j] {
        vx[i] += 1
        vx[j] -= 1
      } else if px[i] > px[j] {
        vx[i] -= 1
        vx[j] += 1
      }
    }
  }
}

func applyVelocity(px, vx *[4]int64) {
  for i := range px {
    px[i] += vx[i]
  }
}

func step(px, vx *[4]int64) {
  applyGravity(px, vx)
  applyVelocity(px, vx)
}
```

To find a loop for a given coordinate, we step until we see a position and
velocity vector that matches the initial one, then send the iteration count
out on a channel. We do this for each coordinate, then pull the per-dimension
iteration counts out of the channel and compute the least common multiple --
the smallest number that is a multiple of all three, which should be the
number of steps it takes to cycle all three dimensions.

```
func findLoopCoord(px, vx [4]int64, ch chan<- int64) {
  pi, vi := px, vx
  for i := int64(1); ; i++ {
    step(&px, &vx)
    if px == pi && vx == vi {
      ch <- i
      return
    }
  }
}

func findLoop(s state) int64 {
  ch := make(chan int64)
  defer close(ch)
  go findLoopCoord(s.px, s.vx, ch)
  go findLoopCoord(s.py, s.vy, ch)
  go findLoopCoord(s.pz, s.vz, ch)
  return lcm(<-ch, <-ch, <-ch)
}
```

That finally does it.

I should mention that, after this code appeared to be working (i.e. it solved
test case 1 from the problem description), it still seemed to have problems.
The answer it gave was wrong, and it turned out that it wasn't giving the
right answer for test case 2, either. I spent about an hour debugging this,
mostly fiddling with the least common multiple implementation and adding and
removing printf statements. Well, it turns out that when I was trying my
bit-packing approach (mentioned above), I switched the type of each coordinate
to `int8`. Suddenly, while staring at debugging output, I noticed that many of
the values were bigger than I'd expected, outside of the range [-128,127], and
it dawned on me that I had integer overflow. s/int8/int64/ fixed the problem.
I can assure you that this mistake is now forever burned into my brain.

Okay, so now after all, it occurred to me on my commute home why the system
will return to its initial state and we don't need to worry about some
intermediate, non-initial state forming the beginning of the cycle. Here's
why:

Suppose there's a chain of states (s_0, ..., s_m-1, s_m, s_m+1, ..., s_n,
s_m), so that s_m has two distinct parent states: s_m-1, resulting from s_0,
and s_n.

But as mentioned above,

    state(n) = {pos(n), vel(n)}
    pos(n) = pos(n-1) + vel(n)
    vel(n) = vel(n-1) + diffs(n-1)
    diffs = /* something only involving pos(n-1) */

So a state(n-1) is always uniquely determined by state(n). We can't have two
distinct parent states of any beginning of a cycle.

That was a bit convoluted; it's been a long time since I had to write a proof
:)


## Day 13

This was the coolest thing yet. For rendering the screen and getting key
events I used the library [tcell](https://github.com/gdamore/tcell), which has
a super-simple API and worked with zero issues. It took me maybe ten minutes from
adding the import statement to drawing the entire screen properly. Handling
key events properly took a bit longer, particularly since it seems that most
[curses-style](https://en.wikipedia.org/wiki/Curses_(programming_library))
libraries don't support keyup vs. keydown events, just "keypress." Anyway,
on to the code, before I talk about how I implemented beating the games.

For tiles and joystick position I defined some enums, as usual:

```
type TileId int

const (
  EMPTY  TileId = 0
  WALL   TileId = 1
  BLOCK  TileId = 2
  PADDLE TileId = 3
  BALL   TileId = 4
)

type JoystickPos int

const (
  NEUTRAL JoystickPos = 0
  LEFT    JoystickPos = -1
  RIGHT   JoystickPos = 1
)
```

For overall game state and communication with the wrapper main I added a
`GameState` struct with the score and the raw tiles in a map (since for part 1
we need how many tiles were written; probably not necessary but I'd added it
in part 1 and didn't remove it):

```
type ScreenTiles map[geom.Pt2]TileId

type GameState struct {
  Joystick JoystickPos
  Score    int
  Tiles    ScreenTiles
}
```

Before we go into the main loop, we set up the channels: input and output as
usual for the intcode machine, and then a channel for reading key events from
the screen -- implemented with a goroutine that does a blocking read and emits
key events on a channel -- and finally a `time.Tick` channel for controlling
the i/o loop speed:

```
func readEvents(screen tcell.Screen) chan *tcell.EventKey {
  ch := make(chan *tcell.EventKey)
  go func() {
    for {
      event := screen.PollEvent()
      switch e := event.(type) {
      case *tcell.EventKey:
        ch <- e
      }
    }
  }()
  return ch
}

func Play(
  data []int64,
  screen tcell.Screen,
  frameDelay time.Duration,
  joystickInit JoystickPos,
) (GameState, error) {
  in := make(chan int64)
  defer close(in)
  out := intcode.RunProgram(data, in)
  var events chan *tcell.EventKey
  if screen != nil {
    screen.Clear()
    events = readEvents(screen)
  }
  state := GameState{
    Tiles:    ScreenTiles(make(map[geom.Pt2]TileId)),
    Joystick: joystickInit,
  }
  tick := time.Tick(frameDelay)
  /* snip */
}
```

After that we go into the main loop, which wraps a select over the events
channel (to record the current joystick state), the tick channel (to send the
current joystick channel if the machine is trying to read it, otherwise
continue without blocking), and the output channel (to read the screen updates
and detect halting):

```
  /* snip */
loop:
  for {
    select {
    case e := <-events:
      switch e.Key() {
      case tcell.KeyCtrlC:
        break loop
      case tcell.KeyLeft:
        state.Joystick = LEFT
      case tcell.KeyRight:
        state.Joystick = RIGHT
      }

    case <-tick:
      select {
      case in <- int64(state.Joystick):
        state.Joystick = joystickInit
      default:
        continue
      }

    case x, ok := <-out:
      if !ok {
        break loop
      }
      y, z := <-out, <-out
      if x == -1 && y == 0 {
        if z > 0 {
          state.Score = int(z)
        }
      } else {
        tile := TileId(z)
        state.Tiles[geom.Pt2{int(x), int(y)}] = tile
        if screen != nil {
          draw(screen, int(x), int(y), tile)
        }
      }
    }
  }

  return state, nil
}
```

That's the core of the game logic. Rendering the various tiles uses some maps
with predetermined characters:

```
func draw(screen tcell.Screen, x, y int, tile TileId) {
  screen.SetContent(x, y, tileToRune[tile], nil, 0)
  screen.Show()
}

var tileToRune = map[TileId]rune{
  EMPTY:  ' ',
  WALL:   '@',
  BLOCK:  'X',
  PADDLE: '-',
  BALL:   'o',
}
```

The full code is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/breakout/breakout.go).
The wrapper main function is pretty boring, but can be found there too, both
[headless](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day13/day13.go)
to solve parts 1 and 2 and
[interactive](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day13/play.go)
to play the game.

Okay, so how to beat the game without having to do it manually (because I'm
terrible at it)?

I had three ideas:

1. Write an AI for the paddle
2. Disassemble the input program and modify it so the ball always bounces
3. Modify the machine so that the program *thinks* the ball should bounce

The AI idea sounded fun, but the other two sounded like more fun, considering
I've never done anything like them before. I didn't want to write a
disassembler, and even though I know some people already wrote them for
intcode and posted them on Reddit, I wanted a self-contained solution. So I
settled on the third idea, to fool the program.

To do this I added logging to the machine so that it prints each instruction,
including locations of memory reads and writes, at each step. I piped the
logging to a file and played the game once (breaking a few bricks before
losing), then opened the log. Looking through the logs, I saw that immediately
before output instructions for redrawing the paddle and the ball the instructions looked very
similar, with just a couple of addresses different between the two.

I looked for the instructions that precede the (x,y,z) output instructions,
specifically the ones that precede updates for z=3 (PADDLE) and z=4 (BALL).
Compare these lines (with minor changes), preceding each write of (x,y,3):

```
[573] adjrel imm(-4)
[575] jmpif imm(1) rel(0)
[138] add pos(392) pos(384) pos(392)
[142] mul imm(1) pos(392) rel(1)
[146] add imm(0) imm(21) rel(2)
[150] mul imm(3) imm(1) rel(3)
[154] add imm(0) imm(161) rel(0)
[158] jmpif imm(1) imm(549)
[549] adjrel imm(4)
[551] mul rel(-2) imm(42) pos(566)
[555] add rel(-3) pos(566) pos(566)
[559] add imm(639) pos(566) pos(566)
[563] add imm(0) rel(-1) pos(1538)
[567] print rel(-3)
wrote value: 17
[569] print rel(-2)
wrote value: 21
[571] print rel(-1)
wrote value: 3
```

With these lines, preceding (with minor changes) each write of (x,y,4):

```
[573] adjrel imm(-4)
[575] jmpif imm(1) rel(0)
[338] add pos(388) pos(390) pos(388)
[342] add pos(389) pos(391) pos(389)
[346] mul imm(1) pos(388) rel(1)
[350] mul pos(389) imm(1) rel(2)
[354] mul imm(4) imm(1) rel(3)
[358] add imm(0) imm(365) rel(0)
[362] jmpif imm(1) imm(549)
[549] adjrel imm(4)
[551] mul rel(-2) imm(42) pos(566)
[555] add rel(-3) pos(566) pos(566)
[559] add imm(639) pos(566) pos(566)
[563] add imm(0) rel(-1) pos(1586)
[567] print rel(-3)
wrote value: 23
[569] print rel(-2)
wrote value: 22
[571] print rel(-1)
wrote value: 4
```

Since we know the output order is (x,y,z), we can trace back from the print
instruction for the x-coordinate (it's the first one of the three) and see
that in both cases it's writing a value from rel(-3), which, after the `adjrel
imm(4)` statement in both traces, should point to the value that was loaded
previously into rel(1) from position 392 (for the paddle) or position 388 (for
the ball). Okay, so it seems like the ball's x-coordinate is stored at address 388.
Why don't we just always return that value when retrieving the paddle's
x-coordinate, i.e. redirect reads of address 392 to address 388?

Well, I did that in my [Intcode
VM](https://github.com/dhconnelly/advent-of-code-2019/blob/master/intcode/machine.go),
and it looks like this:

```
diff --git a/intcode/machine.go b/intcode/machine.go
index 9b2cc59..a40c516 100644
--- a/intcode/machine.go
+++ b/intcode/machine.go
@@ -40,6 +40,9 @@ func (m *machine) get(addr int64, md mode) int64 {
  v := m.data[addr]
  switch md {
  case pos:
+   if v == 392 {
+     v = 388
+   }
    return m.data[v]
  case imm:
    return v
```

This worked! Here's a video:

<iframe width="560" height="315"
src="https://www.youtube.com/embed/VIEGPBUezWo" frameborder="0"
allow="accelerometer; autoplay; encrypted-media; gyroscope;
picture-in-picture" allowfullscreen></iframe>

The paddle drawing seems a bit wonky now, it never erases after it moves to a
position, but who cares :)


## Day 14

Revenge again for the intcode problems. This took me all day, on and off. Now
granted, I spent all day with my daughter, who is teething (molars) and was
incredibly clingy and crabby and was driving me crazy -- but I still probably
spent four hours on it without even solving part 1. I gave up after her
naptime, and then read through some Reddit threads after her bedtime. I was,
in fact, on the totally wrong track.

Okay, what was I doing wrong?

My initial thought was of something about linear programming, which I learned
about back in university and haven't used since, and so I decided to avoid
something that would require a bunch of research.

I latched on early to the idea of iteratively expanding and reducing the
required chemicals to produce a FUEL until no more reductions were possible
using the given reactions alone, then trying to work out a strategy for how
best to produce unnecessary chemicals. My first idea was to do it essentially
randomly, i.e. in Go's map iteration order. This produced inconsistent
results, so my next idea was to recursively build excess for *each* remaining
required chemical, recording the resulting total ore cost and then using the
branch that minimized that cost. This didn't halt: also not a good strategy.
Doing this recursively, particularly for the real input (which had something
like ten unsatisfied quantities after non-wastefully applying rules alone),
creates a combinatorial explosion. I spent a long time re-writing and staring
at the above and trying to find a better way to select excess chemicals to
build.

The easier approach, apparently, is to just go ahead and build the chemicals
needed at each step and keep track of the excess, which can be used later to
reduce the amount of chemicals to produce for some other reaction.

Types:

```
type quant struct {
  amt  int
  chem string
}

type reaction struct {
  out quant
  ins []quant
}
```

To find the amount of ore we need to build a given amount of a chemical, we
first use whatever excess we have, then build however much more we need to
satisfy the appropriate reaction and store any excess, then recursively find
the amount of ore required to satisfy that reaction.

```
func oreNeeded(
  chem string, amt int,
  reacts map[string]reaction,
  waste map[string]int,
) int {
  if chem == "ORE" {
    return amt
  }

  // reuse excess production before building more
  if avail := waste[chem]; avail > 0 {
    reclaimed := ints.Min(amt, avail)
    waste[chem] -= reclaimed
    amt -= reclaimed
  }
  if amt == 0 {
    return 0
  }

  // build as much as necessary and store the excess
  react := reacts[chem]
  k := 1
  if amt > react.out.amt {
    k = divceil(amt, react.out.amt)
  }
  waste[chem] += k*react.out.amt - amt

  // recursively find the ore needed for the ingredients
  ore := 0
  for _, in := range react.ins {
    ore += oreNeeded(in.chem, k*in.amt, reacts, waste)
  }
  return ore
}
```

This is the first day that I had no idea where I was really going. [Day
12](#day-12), the moon simulation, I understood, and I solved part 1 pretty
easily and had a few ideas in mind for part 2 before talking it over with my
colleague Andrew. Today, though, I had no idea if I was on the right track,
no new ideas to try, and I was frustrated all day.

The lesson I'll draw from this is that, if the problem asks "find the ore
required to build this chemical", then the code should do that: `func
oreRequired(chem string, amt int) int`. The trick of keeping track of excess
chemicals, though -- I don't know that I'd have come up with that even if I'd
started off with a more straightforward approach. Well, it's good to get the
practice.

Code for this solution is on
[GitHub](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day14/day14.go).


## Day 15

Okay, something fun and straightforward again! I didn't get up early today, so
I just had an hour or so at naptime and then again after bedtime, but as
opposed to yesterday, the amount of time it took was productive instead of
frustrating and seemingly boundless. I immediately recognized this as a graph
traversal problem, but couldn't decide whether to do breadth-first or
depth-first initially and ended up implementing part of both before starting
over in the evening.

My problem earlier in the day was in trying to both map out the space and find
the shortest path at the same time, instead of first generating the map using
DFS and then finding shortest paths using BFS. The problem with trying to do
both at the same time is that it adds a ton of complexity. If you try do both
move the droid and keep track of the shortest path to the oxygen with DFS, you
end up having a problem when you find a shorter path to a previously-visited
node: do you then recursively update the distances for everything you already
visited from that node, or do you just update the predecessor of that node and
compute the length later -- and then, why not just do BFS anyway? If you try
to do both at the same time with BFS, then you have to constantly move the
droid back to the starting position each time.

After it occurred to me to first build the map and then find the path, the
problem simplified drastically -- as did the code.

As usual, we start with some enums:

```
type status int

const (
  WALL status = 0
  OK   status = 1
  OXGN status = 2
)

type direction int

const (
  NORTH direction = 1
  SOUTH direction = 2
  WEST  direction = 3
  EAST  direction = 4
)
```

I also abstracted away the intcode I/O into a `droid` struct that can move
about:

```
type droid struct {
  in  chan<- int64
  out <-chan int64
}

func (d *droid) step(dir direction) status {
  d.in <- int64(dir)
  return status(<-d.out)
}
```

Okay, so to build a map of the area, we run a DFS starting from (0, 0). We
assume we're already there, and then we move to each neighbor,
marking the status of that move on the map, recursively visit it, and then
we step back from that neighbor and move to the next one:

```
func (d *droid) visit(p geom.Pt2, m map[geom.Pt2]status) {
  // try to move to each unvisted neighbor, recurse, then return
  for dir, dp := range directions {
    next := p.Add(dp)
    if _, ok := m[next]; ok {
      continue
    }
    s := d.step(dir)
    if m[next] = s; s == WALL {
      continue
    }
    d.visit(next, m)
    d.step(opposite(dir))
  }
}

func explore(prog []int64) map[geom.Pt2]status {
  in := make(chan int64)
  out := intcode.RunProgram(prog, in)
  d := droid{in, out}
  m := map[geom.Pt2]status{geom.Zero2: OK}
  d.visit(geom.Zero2, m)
  return m
}
```

The function `opposite` just returns a direction's opposite (since we need to
move back to our original spot from a neighbor), and `neighbors` is a map of
each direction to the necessary vectors:

```
func opposite(dir direction) direction {
  switch dir {
  case NORTH:
    return SOUTH
  case SOUTH:
    return NORTH
  case WEST:
    return EAST
  case EAST:
    return WEST
  }
  log.Fatal("bad direction:", dir)
  return 0
}

var directions = map[direction]geom.Pt2{
  NORTH: geom.Pt2{0, 1},
  SOUTH: geom.Pt2{0, -1},
  WEST:  geom.Pt2{-1, 0},
  EAST:  geom.Pt2{1, 0},
}
```

When `explore` returns, we have a map of point -> status for each reachable
point in the area. Now we need to find the shortest path from (0, 0) to the
location of the oxygen system. To do this we use BFS over the coordinates,
with neighbors of a given node are the non-wall nodes within one Manhattan
distance step away on the grid. With BFS we always visit nodes at distance n
from the starting position before visiting nodes at distance n+1. We do this
using a queue, and we add neighbors of a given node to the end of the queue.
The code to find all shortest paths is simpler than finding a single one, so
here's the entire thing:

```
type node struct {
  p geom.Pt2
  n int
}

func shortestPaths(from geom.Pt2, m map[geom.Pt2]status) map[geom.Pt2]int {
  // track which nodes we've visited and how far away they are
  visited := make(map[geom.Pt2]bool)
  dist := make(map[geom.Pt2]int)

  // keep a queue of the next nodes to visit, in sorted order, with closer
  // nodes always before further ones
  q := []node{{from, 0}}
  var nd node

  // continue as long as the queue is empty
  for len(q) > 0 {
    // pop the head off the queue and record its distance
    nd, q = q[0], q[1:]
    dist[nd.p] = nd.n

    // add each unvisited neighbor to the end of the visit queue, with
    // distance one greater than the distance of the current node
    for _, dp := range directions {
      nbr := nd.p.Add(dp)
      if visited[nbr] {
        continue
      }
      visited[nbr] = true
      if m[nbr] != WALL {
        q = append(q, node{nbr, nd.n + 1})
      }
    }
  }
  return dist
}
```

Then, to find the length of the shortest path to the oxygen system, we just
return the distance of the oxygen system's node:

```
func findOxygen(m map[geom.Pt2]status) geom.Pt2 {
  for p, s := range m {
    if s == OXGN {
      return p
    }
  }
  log.Fatal("oxygen not found")
  return geom.Zero2
}

func shortestPath(from, to geom.Pt2, m map[geom.Pt2]status) int {
  return shortestPaths(from, m)[to]
}
```

Hooking it all together, we have:

```
func main() {
  data, err := intcode.ReadProgram(os.Args[1])
  if err != nil {
    log.Fatal(err)
  }
  m := explore(data)
  p := findOxygen(m)
  fmt.Println(shortestPath(geom.Zero2, p, m))
}
```

For part 2, we want to find the distance of the *furthest* node, since the
time it takes to reach it is the time it takes for the entire area to fill
with oxygen. Since we know all the shortest path distances already, we just
find the longest one of those:

```
func longestPath(from geom.Pt2, m map[geom.Pt2]status) int {
  max := 0
  for _, n := range shortestPaths(from, m) {
    max = ints.Max(max, n)
  }
  return max
}
```

This was a blast. I'd like to come back to this day after the year is over and
create an animated GIF of the droid exploration and oxygen propagation. I'd
like to do that for several of the problems, actually -- there's a lot of nice
visualizations on Reddit for any given day, and my limited experience with
Go's image library makes it seem like this would be straightforward, if only I
took the time to learn how :)

Full code for today is
[here](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day15/day15.go).


## Day 16

This is the longest I've spent on any day so far, but I did eventually get it
on my own! The first part was straightforward -- define the signal pattern and
apply it to the input signal 100 times:

```
func coef(row, col int) int {
  switch (col / row) % 4 {
  case 0: return 0
  case 1: return 1
  case 2: return 0
  default: return -1
  }
}

func fft(signal []int, phases int) []int {
  signal = copied(signal)
  scratch := make([]int, len(signal))
  for ; phases > 0; phases-- {
    for i := 0; i < len(signal); i++ {
      sum := 0
      for j := 0; j < len(signal); j++ {
        sum += coef(i+1, j+1) * signal[j]
      }
      scratch[i] = ints.Abs(sum) % 10
    }
    signal, scratch = scratch, signal
  }
  return signal
}
```

But the second part took me like five hours. I first tried to find some math
trick that would make it easy, since reading 650 bytes (my input size) and
repeating it 10,000 times requires at least 6.5 MB, assuming you somehow avoid
expanding each integer into a multi-byte int representation, and then trying
to precompute the enormous coefficient matrix would take up something like
that much memory squared (or at least half that, considering the matrix is
triangular) -- this is like 36 terrabytes of data! So I ended up on a
Wikipedia exploration, re-learning about Fast Fourier Transforms, diagonal
matrices, determinants and so on, as well as revisiting basic matrix algebra
using inverses and matrix decompositions and so on, before eventually
abandoning a math-heavy approach as (1) unbounded, when I wanted to definitely
solve the problem today, and (2) unlikely to be necessary, since the
backgrounds of Advent of Code participants vary so widely. I returned to the
naive approach and tried to make it work.

The first problem was running out of memory: the slices were simply too large,
as mentioned. At some point, though, while dumping stuff to the console, I
noticed that the offset specified by the first seven digits was very high,
like, most of the way through the repeated signal. That meant that keeping
everything in memory was more possible and there were many fewer elements to
compute, since to find the last k elements `m[n-k], m[n-k+1], ... m[n]`
elements of a vector `m`, where `A s = m` and `A` is the coefficient matrix
and `s` is the input signal, we only need to consider the last k rows of the
matrix `A` -- and since `A` is triangular, we only need to consider half the
elements of each row of `A` (i.e. only need to precompute those coefficients).

So far the code looked like this:

```
func extractMessage(signal []int, reps, phases, offset, digits int) []int {
  // allocate the [offset, end) slice for computing the message
  msg := sliceSignal(signal, offset, len(signal)*reps)
  n := len(msg)

  // pre-compute the coefficients, skipping the leading zeroes
  coefs := make([][]int, n)
  for i := 0; i < n; i++ {
    coefs[i] = make([]int, n-i)
    for j := i; j < n; j++ {
      coefs[i][j-i] = coef(offset+i+1, offset+j+1)
    }
  }

  // repeatedly apply the coefficient matrix rows to the message vector
  scratch := make([]int, n)
  for ; phases > 0; phases-- {
    for i := 0; i < n; i++ {
      sum := 0
      for j := i; j < n; j++ {
        sum += (coefs[i][j-i] * msg[j])
      }
      scratch[i] = ints.Abs(sum) % 10
    }
    msg, scratch = scratch, msg
  }

  return msg[:digits]
}
```

This worked well, and it solved the part 2 examples in a reasonable
amount of time, but still out-of-memoried on the real input during coefficient
precomputation. I removed the precomputation, just relying on the `coef`
function as defined above to compute each coefficient as needed, but it was
too slow: even computing a single phase took more than several minutes before
I stopped it.

At some point here I noticed another thing in the debug output I was dumping
to the console: it seemed like the coefficients were all ones! While taking a
break, something occurred to me that I noticed during my matrix math
diversion: rows in the lower half of the matrix were all ones, regardless of
what size matrix I wrote out for doing manual products (to look for a general
formula). And then it was obvious: the first n-1 elements of the nth row are
zero, and then the next n elements are one, and so together this means that if
we're looking at a row more than half way down the matrix, the zeroes and ones
make up the entire row!

This means that we can forget about the coefficients entirely and just sum up
the vector elements -- and if we do it starting from the last element, which
is just itself, we don't even need to start the sum over at each previous
element, since the  sum for element `a[n-k]` is just `sum(a[n-k+1], ...
a[n])`.

This was a pretty trivial change to the code above, and it works -- and
computes the answer very quickly! This makes sense: the only allocation
now is for the repeated signal from the offset (something like 500k integers,
which should be about 4 MB at 8 bytes per integer).

```
func extractMessage(signal []int, reps, phases, offset, digits int) []int {
  msg := sliceSignal(signal, offset, len(signal)*reps)
  n := len(msg)
  for ; phases > 0; phases-- {
    sum := 0
    for i := n - 1; i >= 0; i-- {
      sum += msg[i]
      msg[i] = ints.Abs(sum) % 10
    }
  }
  return msg[:digits]
}
```

Even though this took me forever I'm proud of it! This felt like a
typical problem solving process for something nontrivial: explore the
problem space a bit with some simpler examples/implementation, read the theory
and try to apply it a bit, produce a naive implementation, use some heuristics
based on understanding the input data, make space/time tradeoffs to get
something usably efficient, then eventually find another heuristic that makes
the problem tractable based on data exploration and debugging and writing
things out by hand. That kind of realization really requires having spent
enough time with the problem and data, in my experience!

Looking at the [global
leaderboard](https://adventofcode.com/2019/leaderboard/day/16) for this
problem, it seems like this was the hardest problem yet. So I'm very happy
that I solved it myself! Day 14 remains the only one that I couldn't figure
out at all.

Full code for today is
[here](https://github.com/dhconnelly/advent-of-code-2019/blob/master/day16/day16.go).


## Day 17

Another straightforward part 1 and very long part 2 that required staring at
the data until devising something based on the properties of the specific case
:)

For part 1, finding intersections, we start by reading the grid from the Intcode machine.

```
type grid struct {
  height, width int
  g             map[geom.Pt2]rune
}

func readGridFrom(out <-chan int64) (grid, bool) {
  g := grid{g: make(map[geom.Pt2]rune)}
  var width int
  for ch := range out {
    if ch == '\n' {
      if width > 0 {
        g.height++
        g.width = width
        width = 0
        continue
      } else {
        return g, true
      }
    }
    g.g[geom.Pt2{width, g.height}] = rune(ch)
    width++
  }
  return grid{}, false
}
```

Then we construct an adjacency list, where two points are adjacent if their
Manhattan distance is 1 and neither is empty space (ASCII '.').

```
func (g grid) neighbors(p geom.Pt2) []geom.Pt2 {
  var nbrs []geom.Pt2
  for _, nbr := range p.ManhattanNeighbors() {
    if c, ok := g.g[nbr]; ok && c != '.' {
      nbrs = append(nbrs, nbr)
    }
  }
  return nbrs
}

func readGraph(g grid) map[geom.Pt2][]geom.Pt2 {
  m := make(map[geom.Pt2][]geom.Pt2)
  for i := 0; i < g.height; i++ {
    for j := 0; j < g.width; j++ {
      p := geom.Pt2{j, i}
      if c := g.g[p]; c == '.' {
        continue
      }
      var edges []geom.Pt2
      for _, nbr := range g.neighbors(p) {
        edges = append(edges, nbr)
      }
      m[p] = edges
    }
  }
  return m
}
```

Now we can find intersections by simply finding all points that have more than
two neighbors, and the alignment sum is computed over those points:

```
func intersections(m map[geom.Pt2][]geom.Pt2) []geom.Pt2 {
  var ps []geom.Pt2
  for p, edges := range m {
    if len(edges) > 2 {
      ps = append(ps, p)
    }
  }
  return ps
}

func alignmentSum(g grid) int {
  m := readGraph(g)
  ps := intersections(m)
  sum := 0
  for _, p := range ps {
    sum += p.X * p.Y
  }
  return sum
}
```

For part 2, let me start with the framework for providing programs to the
robot and reading its responses. All I/O is line-based, so we need to be able
to read and write entire strings at a time, terminated by `'\n'`:

```
func writeLine(ch chan<- int64, line string) {
  for _, c := range line {
    ch <- int64(c)
  }
  ch <- int64('\n')
}

func readLine(ch <-chan int64) string {
  var s []rune
  for {
    c := <-ch
    if c == '\n' {
      return string(s)
    }
    s = append(s, rune(c))
  }
}
```

To run the program headless we have to read the grid once at the beginning,
read the input prompt lines before each input line, and then ignore all output
but the last digit. This took a bit of trial-and-error to figure out.

```
func computeDust(data []int64, prog [4]string) int64 {
  data = ints.Copied64(data)
  data[0] = 2
  in := make(chan int64)
  out := intcode.RunProgram(data, in)
  readGridFrom(out)
  for _, line := range prog {
    readLine(out)
    writeLine(in, line)
  }
  readLine(out)
  writeLine(in, "n")
  var answer int64
  for c := range out {
    answer = c
  }
  return answer
}
```

Okay! So for part 2, in the end I had to figure out the program by hand. Let
me walk through how that happened.

Initially I misread the problem and thought the program had to navigate the
robot back to the starting position. So after writing a DFS traversal of the
scaffolding that always walks back from successor nodes, to make sure we get
back to the beginning, the path was very long. So at this point I assumed that
I definitely had to figure out something clever, because there seemed to be
simply too many different path components. So I read about different
compression techniques for a while, read the problem again, and then noticed
that the robot goes back to its initial position on its own.

Then I noticed by staring at the grid that actually it's possible to visit
every point without doing anything clever at all:

```
..............#########..........................
..............#.......#..........................
..............#.......#..........................
..............#.......#..........................
..............#.......#..........................
..............#.......#..........................
..............#.......#..........................
..............#.......#..........................
..............#...#####.#######.....#############
..............#...#.....#.....#.....#...........#
..............#...#.....#.....#.....#...........#
..............#...#.....#.....#.....#...........#
..............#######...#...#############.......#
..................#.#...#...#.#.....#...#.......#
..................#.#...#############...#.......#
..................#.#.......#.#.........#.......#
..................#.#.......#.#.........#########
..................#.#.......#.#..................
..............#############.#.#...#######........
..............#...#.#.....#.#.#...#.....#........
############^.#...#############...#.....#........
#.............#.....#.....#.#.....#.....#........
#.............#.....#.....#.#.....#.....#........
#.............#.....#.....#.#.....#.....#........
#.............#######.....#.#############........
#.........................#.......#..............
#.....#########...........#.......#..............
#.....#.......#...........#.......#..............
#.....#.......#...........#.......#..............
#.....#.......#...........#.......#..............
#.....#.......#############.......#######........
#.....#.................................#........
#######.................................#........
........................................#........
........................................#........
........................................#........
........................................#........
........................................#........
........................................#........
........................................#........
........................................#........
........................................#........
................................#########........
```

You can visit every point by simply always going forward until hitting a wall
and taking the only available turn. I modified the path generation to do this,
which is much simpler than DFS. The resulting path has a lot of common
subsequences (rewritten here in the form that I eventually was able to reduce,
with three repeated sequences):

```
L,12,L,12,L,6,L,6,
R,8,R,4,L,12,
L,12,L,12,L,6,L,6,
L,12,L,6,R,12,R,8,
R,8,R,4,L,12,
L,12,L,12,L,6,L,6,
L,12,L,6,R,12,R,8,
R,8,R,4,L,12,
L,12,L,12,L,6,L,6,
L,12,L,6,R,12,R,8,
```

This is small enough to do by hand. I started by finding the longest element I could see
immediately, "L,12,L,6", extracted it to a "variable", and proceeded from there. It didn't
take long to get an answer:

```
A,B,A,C,B,A,C,B,A,C
A: L,12,L,12,L,6,L,6
B: R,8,R,4,L,12
C: L,12,L,6,R,12,R,8
```

This worked. I don't believe it's a coincidence that the problems today and
yesterday required exploiting specific properties of the data rather than
solving some hard, general problem. It's a good lesson for problem solving in
general and also reflects the real world :)
