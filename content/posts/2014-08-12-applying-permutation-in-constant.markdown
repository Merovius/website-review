---
date: "2014-08-12T11:10:21Z"
tags:
- golang
- programming
aliases:
- "/2014/08/12/applying-permutation-in-constant.html"
omitFeed: true
title: Applying permutation in constant space (and linear time)
summary: A mildly interesting algorithmic problem and my solution to it.
---
I stumbled upon a mildly interesting problem yesterday: Given an Array a and a
permutation p, apply the permutation (in place) to the Array, using only O(1)
extra space.  So, if b is the array after the algorithm, we want that
`a[i] == b[p[i]]`.

Naively, we would solve our problem by doing something like this (I'm using go
here):

```go
func Naive(vals, perm []int) {
	n := len(vals)
	res := make([]int, n)
	for i := range vals {
		res[perm[i]] = vals[i]
	}
	copy(vals, res)
}
```

This solves the problem in O(n) time, but it uses of course O(n) extra space
for the result array. Note also, that it does not really work in place, we have
to copy the result back.

The simplest iteration of this, would be to simply use a sorting-algorithm of
our choice, but use as a sorting key not the value of the elements, but the
position of the corresponding field in the permutation array:

```go
import "sort"

type PermSorter struct {
	vals []int
	perm []int
}

func (p PermSorter) Len() int {
	return len(p.vals)
}

func (p PermSorter) Less(i, j int) bool {
	return p.perm[i] < p.perm[j]
}

func (p PermSorter) Swap(i, j int) {
	p.vals[i], p.vals[j] = p.vals[j], p.vals[i]
	p.perm[i], p.perm[j] = p.perm[j], p.perm[i]
}

func Sort(vals, perm []int) {
	sort.Sort(PermSorter{vals, perm})
}
```

This appears a promising idea at first, but as it turns out, this doesn't
*really* use constant space after all (at least not generally). The go sort
package uses introsort internally, which is a combination of quick- and
heapsort, the latter being chosen if the recursion-depth of quicksort exceeds a
limit in O(log(n)). Thus it uses actually O(log(n)) auxiliary space. Also, the
running time of sorting is O(n log(n)) and while time complexity wasn't part of
the initially posed problem, it would actually nice to have linear running
time, if possible.

Note also another point: The above implementation sorts perm, thus destroying
the permutation array. Also not part of the original problem, this might pose
problems if we want to apply the same permutation to multiple arrays. We can
rectify that in this case by doing the following:

```go
type NDPermSorter struct {
	vals []int
	perm []int
}

func (p NDPermSorter) Len() int {
	return len(p.vals)
}

func (p NDPermSorter) Less(i, j int) bool {
	return p.perm[p.vals[i]] < p.perm[p.vals[j]]
}

func (p NDPermSorter) Swap(i, j int) {
	p.vals[i], p.vals[j] = p.vals[j], p.vals[i]
}

func NDSort(vals, perm []int) {
	sort.Sort(NDPermSorter{vals, perm})
}
```

But note, that this only works, because we want to sort an array of consecutive
integers. In general, we don't want to do that. And I am unaware of a solution
that doesn't have this problem (though I also didn't think about it a lot).

The solution of solving this problem in linear time lies in a simple
observation: If we start at any index and iteratively jump to the *target*
index of the current one, we will trace out a cycle. If any index is not in the
cycle, it will create another cycle and both cycles will be disjoint. For
example the permutation

```text
i    0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19
p[i] 2  13 1  5  3  15 14 12 8  10 4  19 16 11 9  7  18 6  17 0
```

will create the following set of cycles:
<a href="/assets/permutation.svg"><img src="/assets/permutation.svg"></a>

So the idea is to resolve every cycle separately, by iterating over the indices
and moving every element to the place it belongs:

```go
func Cycles(vals, perm []int) {
	for i := 0; i < len(vals); i++ {
		v, j := vals[i], perm[i]
		for j != i {
			vals[j], v = v, vals[j]
			perm[j], j = j, perm[j]
		}
		vals[i], perm[i] = v, i
	}
}
```

This obviously only needs O(1) space. The secret, why it also only uses O(n)
time lies in the fact, that the inner loop will not be entered for elements,
that are already at the correct position. Thus this is (from a complexity
standpoint at least) the optimal solution to the problem, as it is impossible
to use *less* than linear time for applying a permutation.

There is still one small problem with this solution: It also sorts the
permutation array. We need this, to know when a position is already occupied by
it's final element. In our algorithm this is represented by the fact, that the
permutation is equal to it's index at that point. But really, it would be nice
if we could mark the index *without* losing the order of the permutation. But
that is not hard either - because every index is non-negative, we can
simply negate every index we are done with. This will make a negative index out
of it and we can check for that if we encounter it later and skip it in this
case. After we are done, we only need to take care to flip everything back and
all should be fine:

```go
func NDCycles(vals, perm []int) {
	for i := 0; i < len(vals); i++ {
		if perm[i] < 0 {
			// already correct - unmark and go on
			// (note that ^a is the bitwise negation
			perm[i] = ^perm[i]
			continue
		}

		v, j := vals[i], perm[i]
		for j != i {
			vals[j], v = v, vals[j]
			// When we find this element in the future, we must not swap it any
			// further, so we mark it here
			perm[j], j = ^perm[j], perm[j]
		}
		vals[i] = v
	}
}
```

Here we only mark the elements we will again encounter in the *future*. The
current index will always be unmarked, once we are done with the outer loop.

I am aware, that this is technically cheating; This solution relies on the
fact, that the upper-most bit of the permutation elements won't ever be set.
Thus, we actually *do* have O(n) auxiliary space (as in n bit), because these
bits are not necessary for the algorithm. However, since it is pretty unlikely,
that we will find an architecture where this is not possible (and go guarantees
us that it actually is, because len(vals) is *always* signed, so we cant have
arrays that are big enough for the msb being set anyway), I think I am okay
with it ;)

I ran sum Benchmarks on this an these are the figures I came up with:

<table>
	<tr>
		<th>n</th>
		<th>Naive</th>
		<th>NDCycle</th>
		<th>NDSort</th>
	</tr>
	<tr>
		<td>10</td>
		<td>332 ns</td>
		<td>130 ns</td>
		<td>1499 ns</td>
	</tr>
	<tr>
		<td>100</td>
		<td>883 ns</td>
		<td>1019 ns</td>
		<td>27187 ns</td>
	</tr>
	<tr>
		<td>1000</td>
		<td>15046 ns</td>
		<td>17978 ns</td>
		<td>473078 ns</td>
	</tr>
	<tr>
		<td>10000</td>
		<td>81800 ns</td>
		<td>242121 ns</td>
		<td>4659433 ns</td>
	</tr>
</table>

I did not measure space-use. The time of NDCycle for 10000 elements seems
suspicious - while it is not surprising, that in general it takes more time
than the naive approach, due to it's complexity, this jump is unexpected. Maybe
if I have the time I will investigate this and also measure memory use. In the
meantime, I
[uploaded](https://gist.github.com/Merovius/9e31f4dc6a42a78c1942) all the
code used here, so you can try it out yourself. You can run it with `go run
perm.go` and run the benchmarks with `go test -bench Benchmark.*`.
