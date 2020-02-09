---
title: "Four of a Kind"
date: 2020-02-08T18:40:13Z
categories:
    - probability
    - mathematics
    - puzzles
tags:
    - hypergeometric
    - symmetry
    - random variable
    - expectation
    - nerd snipe
---

I stumbled across a blog from a really interesting pioneer of computing, [David S Maynard](https://www.software-artist.com/). [This post about a probability puzzle](https://www.software-artist.com/joy-of-coding-observable/) nerd-sniped me. The problem statement is:

```
Imagine we have a deck of 52 standard playing cards.

First question:  How many cards do we have to draw until we can be sure
that we have four of a kind?  In other words:  What is the minimum
number of cards we must draw until we can be certain that four of those
cards are the same kind?

Second question:  What if we started from a tall stack made of TWO decks
of 52 standard playing cards.  How many cards do we have to draw from
this stack until we can be sure that we have four of a kind, where it's
OK to have two of a kind from the same suit?

Third question:  On average, with a standard deck of cards, how many
cards do we have to draw to get four of a kind?
```

The post goes into details about [how to use simulation in Observable](https://observablehq.com/@dmaynard/tickler-puzzle-four-of-a-kind) to solve the third question. A [closed-form solution](https://en.wikipedia.org/wiki/Closed-form_expression) isn't given and I thought it looked like a good way to brush away some probability cobwebs.

## First and Second Questions

David S Maynard says the first two questions are easy and he's right. If you draw 40 or more cards from the deck and try to arrange them in 13 "buckets" by rank, then at least one bucket will have at least four cards. If you draw 39 cards, then the buckets can be filled with three cards each (three-of-a-kinds) without resulting in a four-of-a-kind. Therefore,
to guarantee that you get a four-of-a-kind, draw 40 cards.

Note that the solution is independent of the construction of the deck of cards (so long as it has at least 40 cards). In particular, the solution applies to drawing from a single deck or a double deck.

## Third Question

Here's my attempt at a closed-form solution. Let's remind ourselves of the probability basics (I need probability formality to keep things sane in my head: probability is counter-intuitive, at least to me).

![probability space for standard deck of cards](/images/fourok/prob-space.svg#center)

To model counting the number of draws, I introduce a [random variable](https://en.wikipedia.org/wiki/Random_variable) and calculate its [expectation](https://en.wikipedia.org/wiki/Expected_value).

![solution](/images/fourok/soln.svg#center)

![symmetry](/images/fourok/symmetry.svg#center)

![hypergeometric](/images/fourok/hypergeometric.svg#center)

I'll show below that the answer is 68719476736/2748462675 (25.002877921927755).

## Implementation in Golang

The [following golang program](https://play.golang.org/p/ZeYQwya_VLr) evaluates the close form solution (exactly as a fraction) and approximates the solution with a Monte Carlo experiment.

```golang
package main

import (
	"fmt"
	"math"
	"math/big"
	"math/rand"
	"runtime"
	"time"
)

func mul(a, b *big.Int) *big.Int {
	return (&big.Int{}).Mul(a, b)
}

func binomial(n, m int64) *big.Int {
	if m > n || n < 0 || m < 0 {
		return big.NewInt(0)
	}

	if n == m {
		return big.NewInt(1)
	}
	return (&big.Int{}).Binomial(n, m)
}

func choices(i int64) *big.Int {
	return binomial(13, i)
}

func add(a, b *big.Int) *big.Int {
	return (&big.Int{}).Add(a, b)
}

func fraction(a, b *big.Int) *big.Rat {
	return (&big.Rat{}).SetFrac(a, b)
}

func sign(i int64) *big.Int {
	return big.NewInt(-1 + 2*(i%2))
}

func estimate(samples, channels int) float64 {

	type card struct {
		suit, rank int
	}

	histogram := make([]int, 52)

	draw := func(pack []card) int {
		set := map[int]int{}
		for n, card := range pack {
			set[card.rank] += 1
			if set[card.rank] == 4 {
				return n
			}
		}
		return len(pack)
	}

	results := make(chan int)

	rand.Seed(time.Now().UnixNano())
	for i := 0; i < channels; i++ {
		go func(i int) {
			r := rand.New(rand.NewSource(int64(i) + time.Now().UnixNano()))
			pack := make([]card, 52)

			for i := 0; i < len(pack); i++ {
				pack[i] = card{
					suit: i % 4,
					rank: (i / 4) % 13,
				}
			}
			for i := 0; i < samples; i++ {
				r.Shuffle(len(pack), func(i, j int) { pack[i], pack[j] = pack[j], pack[i] })
				results <- draw(pack)
			}
		}(i)
	}

	for i := 0; i < channels; i++ {
		for j := 0; j < samples; j++ {
			n := <-results
			histogram[n] += 1
		}
	}

	sum := 0
	for k, v := range histogram {
		sum += (k + 1) * v
	}
	return float64(sum) / (float64(samples) * float64(channels))
}

func main() {
	const (
		maxDraws          int64 = 40
		samplesPerChannel       = 100000
	)

	channels := runtime.NumCPU()

	expectation := big.NewRat(maxDraws, 1)

	for n := int64(4); n < maxDraws; n++ {
		sum := big.NewInt(0)
		for m := int64(1); m <= 13; m++ {
			tmp := mul(mul(sign(m), choices(m)), binomial(52-4*m, n-4*m))
			sum = add(sum, tmp)
		}
		expectation.Sub(expectation, fraction(sum, binomial(52, n)))
	}

	expectationFloat, _ := expectation.Float64()

	fmt.Printf("Expectation = %v (%v)\n", expectation, expectationFloat)
	approximation := estimate(samplesPerChannel, channels)
	fmt.Printf("Approximation for %d samples = %v (error = %v)\n", samplesPerChannel*channels, approximation, math.Abs(expectationFloat-approximation))
}
```

An example run:

```
Expectation = 68719476736/2748462675 (25.002877921927755)
Approximation for 100000 samples = 25.02365 (error = 0.020772078072244682)
```

Some things to note:

* it's necessary to use big.Rat and big.Int as these numbers will overflow built in types
* the inner sum of the expectation calculation runs from 4 to 40 (a minor optimisation) because
  * you can't have a four-of-a-kind with less than four cards
  * [P(X <= n) = 1 for n >= 40](#first-and-second-questions)
* the sum is subtracted from 40 (see [above for why](#first-and-second-questions))

## Wolfram Alpha is Pretty Amazing

![Wolfram Alpha Solution](/images/fourok/wolfram.png#center)






