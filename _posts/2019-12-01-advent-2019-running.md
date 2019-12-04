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

[Day 1](#day-1) [Day 2](#day-2) [Day 3](#day-3)

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
