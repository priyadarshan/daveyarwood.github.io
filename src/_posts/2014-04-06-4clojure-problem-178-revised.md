---
layout: post
title: "4Clojure Problem #178 - Revised"
category:
tags: [clojure]

redirect_from: '/2014/04/06/4clojure-problem-178-revised/'
---
{% include JB/setup %}

I got some great [feedback][feedback] on my [last post][last] and decided to re-work my solution a bit. I think this turned out much nicer.

The main things I improved:

* My first solution involved "predicates" (technically they may not have been true predicates, since they returned a keyword or nil, instead of true or false) for each possible hand within one big `let` binding, and then at the end I removed all the nil values and returned the first possible hand in the list. This was fairly readable, but there were two problems:
  1. The code could be more concise if I didn't have to wrap each "predicate" in an `if` statement to either return the keyword or nil. Using `when` instead would help to some extent (since it returns either a value or nil), but a better solution altogether came from the other problem, which was that
  2. Wrapping all of my "predicates" in a `let` binding forces them all to be evaluated, even if we may only need to evaluate a handful of them. Using `or` or `cond` instead allows you to take advantage of short-circuiting and stop evaluating once you reach a predicate that is met, which is better from a performance standpoint. Since the first possibility we check for (i.e. the highest ranking possible hand) is a straight flush, I decided to include the predicates `flush?` and `straight?` in my `let` binding and then refer to them in the main part of my code within a `cond` statement.
* It turns out that my previous definition of the `flush?` predicate, `(= 1 (count (set suits)))`, could be simplified to `(apply = suits)`, thanks to `=` being a variable-arity function in Clojure.
* My first solution repeated this piece of code an awful lot: `(some (fn [[r num]] (>= num n)) (frequencies ranks))`. This was a classic opportunity to apply the DRY principle, so I added a `rank-freqs` symbol to my `let` binding. To make it easier to use, I defined it as `(sort (vals (frequencies ranks)))`, so that the value of `rank-freqs` is _always_ `[2 3]` for a full house, for instance.

[feedback]: http://www.reddit.com/r/Clojure/comments/225uq8/my_solution_to_4clojure_problem_178/
[last]: {% post_url 2014-04-03-4clojure-problem-178 %}

Without further ado, here is my revised solution for [Clojure Problem #178](http://www.4clojure.com/problem/178):

{% highlight clojure %}

(fn best-hand [card-strings]
  (let [card-parser (fn [[s r]]
                      (let [suit ({\S :spade, \H :heart, 
                                   \D :diamond, \C :club} s)
                            rank (if (> (Character/digit r 10) -1)
                                   (- (Character/digit r 10) 2)
                                   ({\T 8, \J 9, 
                                     \Q 10, \K 11, \A 12} r))]
                        {:suit suit, :rank rank}))
        cards (map card-parser card-strings)
        suits (map :suit cards)
        ranks (map :rank cards)
        rank-freqs (sort (vals (frequencies ranks)))
 
        flush? 
        (apply = suits) 
 
        straight? 
        (let [aces-high (sort ranks)
              aces-low (sort (replace {12 -1} ranks))]
          (or
            (= aces-high (take 5 (iterate inc (first aces-high))))
            (= aces-low  (take 5 (iterate inc (first aces-low))))))]
    (cond 
      (and flush? straight?) :straight-flush
      (= rank-freqs [1 4])   :four-of-a-kind
      (= rank-freqs [2 3])   :full-house
      flush?                 :flush
      straight?              :straight
      (some #{3} rank-freqs) :three-of-a-kind
      (= rank-freqs [1 2 2]) :two-pair
      (some #{2} rank-freqs) :pair
      :else                  :high-card)))

{% endhighlight %}
