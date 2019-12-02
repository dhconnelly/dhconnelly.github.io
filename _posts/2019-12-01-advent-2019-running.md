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

[Day 1](#day-1) [Day 2](#day-2)

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
