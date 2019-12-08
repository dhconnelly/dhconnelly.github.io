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

[Day 1](#day-1) [Day 2](#day-2) [Day 3](#day-3) [Day 4](#day-4)
[Day 5](#day-5) [Day 6](#day-6) [Day 7](#day-7) [Day 8](#day-8)

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
still talking about like thirtyish lines of code at a minimum.

Anyway, enough whining, because yesterday's problem showed how Go can make
hard problems easy.

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
