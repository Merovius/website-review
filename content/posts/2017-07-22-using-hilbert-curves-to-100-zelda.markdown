---
date: "2017-07-22T23:56:00Z"
tags:
- games
- math
- programming
- zelda
aliases:
- "/2017/07/22/using-hilbert-curves-to-100-zelda.html"
omitFeed: true
title: Using Hilbert Curves to 100% Zelda
summary: Using math, I made it a lot easier to find the last undiscovered parts of my
  Zelda - Breath Of The Wild map.
---

**tl;dr: I used Hilbert Curves to make it quicker to walk through a list of locations on a map, so I could could fully complete a video game.**

As you probably know recently the question of what [the best Zelda
Game](https://s-media-cache-ak0.pinimg.com/originals/eb/31/e5/eb31e5c0a14d4a68ab8d492e848de608.jpg)
is was finally settled by Breath Of The Wild. Like most people I know I ended
up playing. And to keep me engaged I early on decided that I would get as close
as possible to 100% of the game before finishing it. That is I wanted to finish
all shrines, find all Korok Seeds, max out all armor and do all quests before
killing Ganon. I recently finished that and finally killed Ganon. Predictably,
I was in for a disappointment:

![98.59%](/assets/botw_9859.jpg)

98.59 percent! I did expect that though. The reason is that only certain things
count into the percentage as displayed; Korok Seeds are one of them, Shrines
are another. But it also counts landmarks and locations as shown on the map.
Each contributes 1/12% to the total.

So I started on the onerous task of finding the last 17 locations. I'm not
above using help for that so I carefully scrolled through [an online
map](https://www.zeldadungeon.net/breath-of-the-wild-interactive-map) of the
BotW universe, maticulously comparing the locations on it with the ones already
on my in-game map. Anything I haven't visited was marked and visited. But that
only put me to 99.58%; I was still missing 5 locations. apparently I didn't
compare carefully enough.

I needed a more systematic approach. I started to instead go through an
[alphabetical list of
locations](http://www.ign.com/wikis/the-legend-of-zelda-breath-of-the-wild/Locations_by_Region),
looking up each on the map and see if I already had it mapped. But that got old
*really* quickly. Alphabetical just wasn't a great way to organize these; I
wanted a list that I could systematically check. But I didn't want it
alphabetically but geographically. I didn't want to have to jump around the map
to try and find the next one. Which is when I realized that this would be the
perfect application for a [Hilbert curve](https://en.wikipedia.org/wiki/Hilbert_curve).

If you don't know (though you should really just read the Wikipedia Article),
the Hilbert curve is a space filling fractal curve, that is a continuous
bijective map from the real number line to the plane. It is iteratively
defined as the limit of finite curves that get denser and denser. One of the
most interesting properties of the curve and its finite approximations is that
points that are close on the real number line get mapped to points that are
close in the plane. So if we could extract all locations from the online map,
figure out for each what real number gets mapped to that point and order the
locations by those numbers, we'd get a list of locations where neighbors in the
list are close to each other on the map. Presumably, that
would make for easy checking of the list: The next location should be pretty
much neighbouring the previous one and if I can't find a location nearby,
chances are that I didn't visit it yet (and I can then look it up specifically).

**\[edit\]** Commentors on
[reddit](https://www.reddit.com/r/programming/comments/6oxra8/using_hilbert_curves_to_100_zelda/dklhina/)
and [Hacker news](https://news.ycombinator.com/item?id=14830691) have pointed
out correctly, that all curves satisfy the property that near point on the line
map to near points on the plane. What makes the Hilbert Curve special, is that
we work on finite approximations and with the Hilbert Curve, we don't have to
worry about the "correct" level of discrete approximation.

To see what that means, we can look at a zig-zag curve. Say, we split our map
into a 100000x100000 grid and move in a zig-zag, left-to-right, top-to-bottom.
Given how sparse our point-set is, this would mean that most of the rows are
empty and some of them would have only one point on them. So we wozuld have to
constantly move along the entire width of the map. On the other hand, if we
split it into a 2x2 grid, it wouldn't be very helpful; a lot of points would
end up in the same quadrant, which would be very large, so we wouldn't have won
anything. So there would have to be some fineness of the grid that's "just
right" somewhere in the middle, which we'd need to find.

On the other hand with Hilbert Curves, this isn't a problem. That's because the
*limit* of the finite approximations is continuous (which isn't the case with
the limit of zig-zag curves). What that means, in essence, is that where a
point falls on the curve won't jump around a lot when we make our grid finer,
it will "home in" to its final location on the continuous curve. A first order
Hilbert Curve is just a zig-zag curve, so it has the same problem as the
2x2-grid zig-zag line. But as we increase it's order, the points will just
become more and more local, instead of requiring scanning empty space. That is
the interesting consequence of the Hilbert curve being space-filling.

Really, [this video](https://www.youtube.com/watch?v=3s7h2MHQtxc) explains it
much better than I ever could (even though I find the example given there
slightly ridiculous). In the end, I mostly agree with the commentors; it
probably wouldn't have been too hard to find a good approximation that would
make a zig-zag curve work well. But I had Hilbert Curves ready anyway and
appreciated the opportunity to usue them.**\[/edit\]**

The first step for this was to get a list of locations and their corresponding
positions. I was pretty sure that the online map should have that available
somehow, as it uses some Google Maps framework to draw the map. So I
looked at the network tab of the Chrome developer tools, found the URL that
loaded the landmark data, copied the request as curl and saved the output for
further massaging.

![Chrome developer tools - copy as cURL](/assets/botw_curl.jpg)

The returned file turns out to not actually be JSON (that'd be too easy, I
guess) but some kind of javascript-code which is then probably eventually
eval'd to get the data (**edit**: It has been pointed out, that this is just
[JSONP](https://en.wikipedia.org/wiki/JSONP). I was aware that this is probably
the case, but didn't feel comfortable using the term, as I don't know enough
about it. I also didn't consider it very important) :

```
/**/jQuery31106443585752152035_1500757689075(/* json-data */)
```

I just removed everything but the actual JSON with my editor and ran it through
a pretty-printer, to get at it's actual structure. I spare you the details; it
turns out the list of locations isn't even simply contained in that it's
embedded as another string, with HTML tags, as a property (twice!).

So I quickly hacked together some go code to dissect the data and voilà: Got a
list of location names with the corresponding positions:

```go
func main() {
	var data struct {
		Parse struct {
			Properties []struct {
				Name    string `json:"name"`
				Content string `json:"*"`
			} `json:"properties"`
		} `json:"parse"`
	}

	if err := json.NewDecoder(os.Stdin).Decode(&data); err != nil {
		panic(err)
	}

	var content string

	for _, p := range data.Parse.Properties {
		if p.Name == "description" {
			content = p.Content
		}
	}

	if content == "" {
		panic("no content")
	}

	var landmarks []struct {
		Type     string
		Geometry struct {
			Type        string
			Coordinates []float64
		}
		Properties struct {
			Type string
			Id   string
			Name string
			Link string
			Src  string
		}
	}

	if err := json.NewDecoder(arrayReader(content)).Decode(&landmarks); err != nil {
		panic(err)
	}

	for _, m := range landmarks {
		fmt.Printf("%s: %v\n", m.Properties.Name, m.Geometry.Coordinates)
	}
}

func arrayReader(s string) io.Reader {
	s = strings.TrimSuffix(strings.TrimSpace(s), ",")
	return io.MultiReader(strings.NewReader("["), strings.NewReader(s), strings.NewReader("]"))
}
```

This bode well. Now all I needed to do was to calculate the Hilbert Curve
coordinate for each of them and I'd have what I need. The Wikipedia Article
helpfully contains an
[implementation](https://en.wikipedia.org/wiki/Hilbert_curve#Applications_and_mapping_algorithms)
of the corresponding algorithm in C. `xy2d` assumes a discrete grid of n² cells
and returns an integer preimage of the given coordinates. The coordinates we have
are all floating point numbers between 0 and 2 (ish) with 5 significant digits.
I figured that 65536 should be able to represent the granularity of points well
enough, so I chose that as an n, ported the code to go, sorted the locations
accordingly and it *actually worked*!

```go
func main() {
	// Same stuff as before

	sort.Slice(landmarks, func(i, j int) bool {
		xi := f2d(landmarks[i].Geometry.Coordinates[0])
		yi := f2d(landmarks[i].Geometry.Coordinates[1])
		xj := f2d(landmarks[j].Geometry.Coordinates[0])
		yj := f2d(landmarks[j].Geometry.Coordinates[1])
		di := xy2d(1<<16, xi, yi)
		dj := xy2d(1<<16, xj, yj)
		return di < dj
	})

	for _, m := range landmarks {
		fmt.Printf("%s: %v\n", m.Properties.Name, m.Geometry.Coordinates)
	}
}

func xy2d(n, x, y int) int {
	var d int
	for s := n / 2; s > 0; s = s / 2 {
		var rx, ry int
		if (x & s) > 0 {
			rx = 1
		}
		if (y & s) > 0 {
			ry = 1
		}
		d += s * s * ((3 * rx) ^ ry)
		x, y = rot(s, x, y, rx, ry)
	}
	return d
}

func rot(n, x, y, rx, ry int) (int, int) {
	if ry == 0 {
		if ry == 1 {
			x = n - 1 - x
			y = n - 1 - y
		}
		x, y = y, x
	}
	return x, y
}

func f2d(f float64) int {
	return int((1 << 15) * f)
}
```

In the end, there was still a surprising amount of jumping around involved. I
don't know whether that's accidental (i.e. due to my code being wrong) or
inherent (that is the Hilbert curve just can't map this perfectly well). I
assume it's a bit of both. The list also contains the same landmark multiple
times. This is because things like big lakes or plains where marked multiple
times. It would be trivial to filter duplicates but I actually found them
reasonably helpfull when having to jump around.

There might also be better approaches than Hilbert Curves. For example, we
could view it as an instance of the [Traveling Salesman
Problem](https://en.wikipedia.org/wiki/Travelling_salesman_problem) with a
couple of hundred points; it should be possible to have a good heuristic
solution for that. On the other hand, a TSP solution doesn't necessarily only
have short jumps, so it *might* not be that good?

In any case, this approach was definitely good enough for me and it's probably
the nerdiest thing I ever did :)

![100%](/assets/botw_1000.jpg)
