---
author: Richard Wild
layout: post
asset-type: post
title: "The Functional Style - Part 7"
date: 2018-11-26 00:00:00
description: Functional programming explained for the pragmatic programmer. Part 7. Lazy evaluation.
image: 
    src: /assets/custom/img/blog/the-functional-style/title-images/episode-7.jpeg
    attribution:
       text: Free for commercial use (no attribution required)
       href: https://pixabay.com/en/sunset-hammock-relaxation-bali-2058002/

abstract: Functional programming explained for the pragmatic programmer.
tags: 
- functional programming
---

# Lazy evaluation.

<p style="font-style: italic; margin: 0em 3em 1em 3em">To see a world in a grain of sand and heaven in a wild flower<br/>
Hold infinity in the palm of your hand and eternity in an hour
</p>
<p style="margin: 0em 3em 1em 3em">- William Blake</p>

Several years ago, I was attending a training course on C#. I recall having trouble understanding two things in particular. One of them was LINQ, which was partly a matter of not quite being able to wrap my mind round the syntax. I had been steeped for many years in SQL, and this language which was similar but not quite the same confused me. Also, I hadn’t yet learned about the functional style of programming; now that I have, it makes a lot more sense to me.

The other thing was the `yield` keyword. But again, with an understanding of the functional style it makes much more sense, and in fact it’s really very simple. The [.NET documentation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/yield) gives this example usage:

```csharp
public class PowersOf2
{
    static void Main()
    {
        foreach (var i in Power(2, 8))
            Console.Write("{0} ", i);
    }

    public static IEnumerable<int> Power(int number, int exponent)
    {
        var result = 1;
        for (var i = 0; i < exponent; i++)
        {
            result = result * number;
            yield return result;
        }
    }
}
```

When run it prints out: `2 4 8 16 32 64 128 256`

What happens is, the `Power` method returns an IEnumerable instance and the foreach loop calls MoveNext on it repeatedly. However, the Power method does not explicitly create an instance of IEnumerable. By using the `yield return` statement, the method literally becomes an iterator that computes its iterated values on demand. The value returned here is the first iterated element. At this point, control returns to the foreach loop whereupon its body is executed once. Then it calls MoveNext again on the iterator, which causes control to return to the Power method immediately after the yield return statement. The Power method's internal state is preserved from before, therefore its for loop iterates again and it yields the next iterated element. Control thus jumps repeatedly between the foreach loop and the yield return statement, until finally the for loop terminates and Power exits without calling yield return again.

As [the documentation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/yield) explains it, the principal use case for `yield return` is to implement an iterator for a custom collection type in a method, without the need to create a new IEnumerable or IEnumerator implementation, thus avoiding the need to define a new class. It gives another example to illustrate this point.

### But what if the iterator method never exits?

Imagine we removed the terminating condition so that the loop containing the `yield return` statement continues forever:

```csharp
public static IEnumerable<int> Numbers()
{
    var number = 1;
    for (;;)
        yield return number++;
}
```

If we call it like this, we print out `1 2 3 4 5 6 7 8 9 10`:

```csharp
static void Main()
{
    foreach (var i in Numbers())
    {
        Console.Write("{0} ", i);
        if (i == 10)
            break;
    }
}
```

Now we have to break out of the foreach loop instead, because clearly the `Numbers` method will never exit normally. So what would be the point of that? A deep and profound point, in fact. What we have now is an `IEnumerable` that purports to contain all the natural numbers, which is an infinite set.

### Surely that is impossible!

Indeed it is, but it turns out that the promise is much more important than the fact that it cannot be kept.

If you recall in part 4 we had this code that generates a sieve of prime numbers and returns a predicate for testing whether or not a number is prime:

```java
private Predicate<Integer> notInNonPrimesUpTo(int limit) {
    Set<Integer> sieve = IntStream.range(2, (limit / 2))
            .boxed()
            .flatMap(n -> Stream.iterate(n * 2, nonPrime -> nonPrime += n)
                    .takeWhile(nonPrime -> nonPrime <= limit))
            .collect(Collectors.toSet());
    return candidate -> !sieve.contains(candidate);
}
```

I said that the `Stream.iterate` was interesting and that I would come back to it later. The time has come. This is another example of an iterator that purports to iterate an infinite list:

```java
Stream.iterate(n * 2, nonPrime -> nonPrime += n)
```

It generates a stream of integers beginning at `(n * 2)` and increasing by `n` each time. Notice there is no upper bound. The terminating condition - when the value of `limit` is exceeded - is here:

```java
      .takeWhile(nonPrime -> nonPrime <= limit)
```

Also, if you recall in the previous article we saw several uses of the `range` function in Clojure. In each case I used it in this manner:

```clojure
(range 1 10)
```

which evaluates to a list of numbers from 1 to 10. But we can call `range` with no arguments as well:

```clojure
user=> (take 5 (range))
(0 1 2 3 4)
```

That's right, `range` with no arguments is another purportedly infinite list of integers. However, if you execute `(range)` in the REPL by itself, it will do nothing but simply block. It is waiting for something to be taken. Just as with the Java `Stream.iterate` and the C# `yield` loop, values are only generated when they are requested. This is why the evaluation is said to be lazy. An ‘eagerly’ evaluated sequence generates all its values immediately on creation, just as `(range 1 10)` does. A ‘lazily’ evaluated sequence only generates its values on demand. It is this deferred execution that enables the pretence of infinite sequences. With a wink and a knowing nod, the pretence can be maintained, provided the program never actually asks for the promise to be fulfilled.

### Loop indices.

As you have probably figured out by now, I like a clever trick as much as the next person, but I like it best if it can be put to some practical use that makes my code better. So it is with lazy evaluation.

Earlier in the series, I asserted that the adopting functional style means you will hardly ever have to write a loop again. Mapping and reducing do indeed cover a great many use cases, but nevertheless you do sometimes need to keep track of your iteration with some kind of counter. Now, were I programming in a language like C# or Java, both of which are imperative at heart, I wouldn't bother going through the contortions necessary to do it in a functional style. I would use a traditional `for` loop instead:

```java
public void IterateWithIndex()
{
    var numbers = new[] {"one", "two", "three", "four", "five"};
    for (var i = 0; i < numbers.Length; i++)
        Console.WriteLine("{0}: {1}", i, numbers[i]);
}
```

I _do_ prefer not to mutate state unnecessarily, but for me simplicity is still a higher virtue, and in some languages this is still the simplest way to tackle the problem. All else being equal, I would choose three lines of simple code that mutates state over half a dozen lines of inscrutable functional code that does the same job.

Some languages give you an easy way out-of-the-box to iterate a sequence in a functional style and also provide you with an iteration counter. Groovy provides an `eachWithIndex` method that allows this to be done without the administration work:

```groovy
[
        [symbol: 'I', value: 1],
        [symbol: 'V', value: 5],
        [symbol: 'X', value: 10],
        [symbol: 'L', value: 50],
        [symbol: 'C', value: 100],
        [symbol: 'D', value: 500],
        [symbol: 'M', value: 1000],
].eachWithIndex { numeral, i -> 
    println("${i}: ${numeral.symbol} ${numeral.value}")
}
```

In that case, this is a good choice. But getting the subject back to lazy evaluation, if you were working with Clojure, the problem could be solved in an interesting way by using `(range)`:

```clojure
(map (fn [i number] (format "%d: %s" i number))
     (range)
     '("one" "two" "three" "four" "five"))
```

Recall that `map` on multiple sequences keeps going until the shortest of the sequences runs out. Clearly the lazily evaluated sequence is not going to run out first, so it keeps counting up until the other list is exhausted. Therefore, the result is:

```
("0: one" "1: two" "2: three" "3: four" "4: five")
```

That said, Clojure also provides the `map-indexed` function which does a similar job to `eachWithIndex` in Groovy:

```clojure
(map-indexed (fn [i number] (format "%d: %s" i number))
             '("one" "two" "three" "four" "five"))
```

### Not convinced. Show me an example where lazy evaluation really helps.

One situation I faced in the real world recently which required a loop index was that I wanted to write parameterised log messages. To do this, I chose a template format similar to that used in C#, i.e.:

```
The {0} has a problem. Cause of failure: {1}. Recommended solution: {2}.
```

That wasn’t one of the actual log messages but you get the idea. The program was in Java and the solution quite straightforward:

```java
{% raw  %}
private String replaceParameters(String template, String... parameters) {
    var out = template;
    for (var i = 0; i < parameters.length; i++)
        out = out.replace(String.format("{%d}", i), parameters[i]);
    return out;
}
{% endraw  %}
```

It is totally imperative and it mutates its internal state shamelessly. If we tried to rewrite it into a more functional style it might look something like this:

```java
{% raw  %}
private String replaceParameters(String template, String... parameters) {
    var i = new AtomicInteger(0);
    return Arrays.stream(parameters)
            .reduce(template, (string, parameter) ->
                    string.replace(
                            String.format("{%d}", i.getAndIncrement()),
                            parameter));
}
{% endraw  %}
```

I don’t think this is pragmatic. Rather, I think this is using the functional style just for its own sake, not because it is better. Recall what I said about the sweet spot for functional programming? This isn’t it. For one thing, the clarity of the original has been sacrificed: I've had to break it into more lines to try to aid readability. And it's not really any more functional - it still mutates state anyway, because the `AtomicInteger` is being incremented.

So in Java I think the traditional loop is the best way to go here, but what about in a properly functional language that doesn't give us that choice? In Clojure, the `map-indexed` function would not help here. This is a job for `reduce` instead, and there is no `reduce-indexed` function. What we _do_ have is `zipmap`, sometimes called “zip” in other languages. It is so named because its behaviour is reminiscent of the action of a zipper fastener:

```clojure
user=> (zipmap [1 2 3] ["one" "two" "three"])
{1 "one", 2 "two", 3 "three"}
```

We can use it to write a parameter replacement function using `reduce` like this:

```clojure
{% raw  %}
(defn- replace-parameter [string [i parameter]]
  (clojure.string/replace string (format "{%d}" i) parameter))

(defn replace-parameters [template parameters]
  (reduce replace-parameter template (zipmap (range) parameters)))
{% endraw  %}
```

and it works!

```
user=> (replace-parameters
  #_=>   "The {0} has a problem. Cause of failure: {1}. Recommended solution: {2}."
  #_=>   ["time machine" "out of plutonium" "find lightning bolt"])
"The time machine has a problem. Cause of failure: out of plutonium. Recommended solution: find lightning bolt."
```

### Being lazy ourselves.

We’ve played a bit with lazy sequences out of the box, so let’s do a little exercise that involves building one of our own. You may have heard of Pascal’s Triangle. In mathematical terms, Pascal’s Triangle is a triangular array of the binomial coefficients, but you don’t really need to know what that means because it is very simple to construct. You begin by placing a 1 at the apex and then build a triangle by placing rows of numbers below, offset to the left and right of the numbers above, so that each row has one more number in it than the one above. Each number is equal to the sum of the two numbers above it; for numbers on the edge of the triangle, absent numbers are treated as zero. This diagram should make it clear:

```
     1
    1 1
   1 2 1
  1 3 3 1
 1 4 6 4 1
```

Writing a program to produce this is an interesting problem. It becomes much simpler if you observe that the next row can be generated by duplicating the previous row, shifting one of them to the left or right, and then summing the offset digits, i.e.

```
    1  4  6  4  1  0
+   0  1  4  6  4  1
=   1  5 10 10  5  1
```

In Clojure terms this can be expressed like this:

```clojure
(defn next-row [previous]
  (apply vector (map + (conj previous 0)
                       (cons 0 previous))))
```

A quick test shows that it works:

```
#'user/next-row
user=> (next-row [1])
[1 1]
user=> (next-row [1 1])
[1 2 1]
user=> (next-row [1 2 1])
[1 3 3 1]
```

and then we can build a whole triangle lazily using `iterate` like this:

```clojure
(def triangle (iterate next-row [1]))
```

In Clojure, `iterate` builds a sequence lazily by repeatedly applying the result of the previous iteration to the `next-row` function, starting with the seed value `[1]`, which is a vector containing the number 1.

You can then take as many rows from `triangle` as you wish:

```
user=> (take 7 triangle)
([1] [1 1] [1 2 1] [1 3 3 1] [1 4 6 4 1] [1 5 10 10 5 1] [1 6 15 20 15 6 1])
```

and even request any arbitrary row:

```
user=> (nth triangle 10)
[1 10 45 120 210 252 210 120 45 10 1]
```

Obviously, although `triangle` appears to be a sequence, no actual sequence ever exists in memory, unless you happen to build one yourself out of the results. If you read back to the `yield return` example in C# at the beginning, you will see the same behaviour there. It might seem like a paradox, that unlimited collections require less memory, but this is an important performance tip to bear in mind.

### Do repeat yourself.

To finish our exploration of lazy evaluation, let's have a bit of fun. As someone once drily observed, the kata of FizzBuzz is popular because it avoids the awkwardness of realising no-one in the room can remember how to binary search an array. If you’re one of the handful of people in the world not familiar with the game, it’s dead simple: you count through the numbers while replacing all multiples of three with “Fizz,” all multiples of five with “Buzz,” and all multiples of both with “FizzBuzz.”

<p style="margin: 0em 3em 1em 3em; font-style: italic">N.B. It is usually said to have originated as a drinking game, and indeed I did play a drinking game like this at university, although the Warwick Rules were slightly different. As we played it, buzz was four not five, and there was an additional rule: when the decimal number contained a digit 3 then you also had to say “fizz” and when it contained 4 you had to say “buzz.” Therefore, 12 was “fizzbuzz,” 13 was “fizz,” 14 was “buzz” and 15 was “fizz” again. This made it a bit more complicated and therefore we got more drunk.<br/><br/>

We once tried during a band social playing it with binary numbers, but the baritone sax player objected on the grounds that he was a law student not a scientist. So we agreed to do it in roman numerals instead, thus killing two katas in one.</p>

If there is a canonical implementation, in Java it probably looks something like this:

```java
public class FizzBuzz {

    public static void main(String[] args) {
        for (var i = 0; i < 100; i++)
            System.out.println(fizzBuzz(i));
    }

    private static String fizzBuzz(int i) {
        if (i % 3 == 0 && i % 5 == 0)
            return "FizzBuzz";
        else if (i % 3 == 0)
            return "Fizz";
        else if (i % 5 == 0)
            return "Buzz";
        else
            return String.valueOf(i);
    }
}
```

<p>(but definitely not <a href="https://github.com/EnterpriseQualityCoding/FizzBuzzEnterpriseEdition">like this</a>).</p>

So that solution treats it as an arithmetical problem, but maybe it doesn't have to be. If we analyse it, the whole cycle repeats continuously with a period of fifteen, like this:

_number, number, Fizz, number, Buzz, Fizz, number, number, Fizz, Buzz, number, Fizz, number, number, FizzBuzz_

and perhaps we can treat it that way in our program. Clojure has a function called `cycle` which lazily repeats a supplied pattern endlessly:

```
user=> (take 10 (cycle [nil nil "Fizz"]))
(nil nil "Fizz" nil nil "Fizz" nil nil "Fizz" nil)
```

If we had a function that mapped over these three lazy sequences we could produce a lazily evaluated FizzBuzz implementation out of them:

```clojure
(def numbers (map inc (range)))               ; goes 1 2 3 4 5 6 7 8 9 10 etc.
(def fizzes (cycle [nil nil "Fizz"]))         ; goes nil nil fizz nil nil fizz etc.
(def buzzes (cycle [nil nil nil nil "Buzz"])) ; goes nil nil nil nil buzz nil nil nil nil buzz etc.
```

The `map inc` is necessary because `range` starts from 0 and we need it to start from 1, so we increment every value in the sequence by 1. Now let’s have a stab at writing the function:

```clojure
(defn fizzbuzz [n fizz buzz]
  (if (or fizz buzz)
      (str fizz buzz)
      (str n)))
```

That’s pretty simple; if either `fizz` or `buzz` are truthy (i.e. not nil) then concatenate them both together - when concatenating strings, nil is treated like an empty string - otherwise return `n` as a string. When we map this over all three lazy sequences, we get FizzBuzz!

```
user=> (take 30 (map fizzbuzz numbers fizzes buzzes))
("1" "2" "Fizz" "4" "Buzz" "Fizz" "7" "8" "Fizz" "Buzz" "11" "Fizz" "13" "14" "FizzBuzz" "16" "17" "Fizz" "19" "Buzz" "Fizz" "22" "23" "Fizz" "Buzz" "26" "Fizz" "28" "29" "FizzBuzz")
```

We can also use `nth` to obtain any value from the sequence (taking into account that nth counts from zero, not one):

```
user=> (nth (map fizzbuzz numbers fizzes buzzes) 1004)
"FizzBuzz"
```

Isn’t that cool?

### Next time.

In the next article, I will conclude our examination of the practical aspects of functional programming. I will take a look at persistent data structures, which is how functional languages are able to implement immutable data structures that give the impression of being mutable, while at the same time making the most efficient use of memory by avoiding unnecessary duplication of data.

<hr/>

## The whole series:

1. [Introduction](/2018/08/09/the-functional-style-part-1/)
1. [First Steps](/2018/08/17/the-functional-style-part-2/)
1. [First-Class Functions I: Lambda Functions & Map](/2018/09/04/the-functional-style-part-3/)
1. [First-Class Functions II: Filter, Reduce & More](/2018/09/19/the-functional-style-part-4/)
1. [Higher-Order Functions I: Function Composition and Monads](/2018/10/17/the-functional-style-part-5/)
1. [Higher-Order Functions II: Currying](/2018/11/02/the-functional-style-part-6/)
1. Lazy Evaluation
1. [Persistent Data Structures](/2018/12/04/the-functional-style-part-8/)
1. [Pragmatism](/2018/12/12/the-functional-style-part-9/)