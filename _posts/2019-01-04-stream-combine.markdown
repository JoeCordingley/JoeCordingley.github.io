---
layout: post
title:  "Struggles with combining infinite streams in scala"
date:   2019-01-04 16:43:58 +0000 <!-- :put =strftime('%Y-%m-%d %H:%M:%S %z') -->
comments: true

categories: scala streams
---
## Combining potentially infinite streams in scala

If you want to combine two Streams in scala such a way as to output all possible pairings, you cannot generally simply iterate over them. That is:
```scala 
for {
  i <- Stream.from(0)
  j <- Stream.from(0)
} yield (i,j)
```
would never yield an element with a first index other than 0.

There are many ways of combining two countably infinite sets to create one countably infinite sets in maths. One such example is to keep taking diagonals. The above example could be fixed to:

```scala
val s = for {
  i <- Stream.from(0)
  j <- 0 to i
} yield (j, i - j)
s take 10 foreach println
//(0,0)
//(0,1)
//(1,0)
//(0,2)
//(1,1)
//(2,0)
//(0,3)
//(1,2)
//(2,1)
//(3,0)
```

This is an easy way to get an infinite set of increasing integer pairs, but how about combining two arbitrary streams in this way? That is `def combine[A,B](s1: Stream[A], s2: Stream[B]): Stream[(A,B)] = ???` ?


It would be folly to map the above stream to take indexes of the two input streams as it would be inefficient to continually take the larger and larger indexes of the stream. Plus one would like to account for the cases in which streams are not actually infinite.

My first steps to tackling this problem were to notice the pattern of the first indexes which can be grouped thus: `(0), (0,1), (0,1,2), ... `. The pattern of the second indexes can be grouped similarly thus: `(0), (1,0), (2,1,0), ...`. This led me to try to create these patterns from an arbitrary stream. 

The first is easy: 
```scala
def forwards[A](s:Stream[A]):Stream[List[A]] = s match {
  case Stream.Empty => Stream.empty
  case a #:: as => List(a) #:: forwards(as).map(a::_)
}
forwards(Stream.from(0)) take 4 foreach println
//List(0)
//List(0, 1)
//List(0, 1, 2)
//List(0, 1, 2, 3)
```

The second was somewhat harder but still quite simple:
```scala
def backwards[A](s:Stream[A]):Stream[List[A]] = s match {
  case Stream.Empty => Stream.empty
  case a #:: as => List(a) #:: (as zip backwards(s)).map{
    case (x,xs) => x :: xs
  }
}
backwards(Stream.from(0)) take 4 foreach println
//List(0)
//List(1, 0)
//List(2, 1, 0)
//List(3, 2, 1, 0)
```

We could then use these two functions and zip together the resulting lists and flatten. But rather than do that, we can implement the ideas of those two functions simultaneously and make our function even terser.

```scala
def combine[A,B](s1: Stream[A], s2: Stream[B]): Stream[(A,B)] = {
  def inner(s1: Stream[A], s2: Stream[B]): Stream[List[(A,B)]] = (s1, s2) match {
    case (Stream.Empty,_) => Stream.empty
    case (_,Stream.Empty) => Stream.empty
    case (a #:: as, b#:: bs) => List((a,b)) #:: as.zip(inner(s1,bs)).map{
      case (a,xs) => (a,b) ::xs
    }
  }
  inner(s1,s2).flatten
}
combine(Stream.from(0), Stream.from(0)) take 10 foreach println
//(0,0)
//(1,0)
//(0,1)
//(2,0)
//(1,1)
//(0,2)
//(3,0)
//(2,1)
//(1,2)
//(0,3)
```

This works well for our infinite streams but what about terminating streams?

```scala
combine(Stream(0,1),Stream(0,1,2)) take 6 foreach println
//(0,0)
//(1,0)
//(0,1)
combine(Stream(0,1,2),Stream(0,1)) take 6 foreach println
//(0,0)
//(1,0)
//(0,1)

combine(Stream(0,1,2),Stream(0,1,2)) take 9 foreach println
//(0,0)
//(1,0)
//(0,1)
//(2,0)
//(1,1)
//(0,2)
```

There are missing pairs in all cases. This is as a consequence of the zip function cutting off the additional elements when zipping two uneven streams. Luckily the Stream API has a `zipAll` function that will zip to the length of the longest stream given some default elements to pad with.

```scala
def combine[A,B](s1: Stream[A], s2: Stream[B]): Stream[(A,B)] = {
  def inner(s1: Stream[A], s2: Stream[B]): Stream[List[(A,B)]] = (s1, s2) match {
    case (Stream.Empty,_) => Stream.empty
    case (_,Stream.Empty) => Stream.empty
    case (a #:: as, b#:: bs) => List((a,b)) #:: as.map(Some(_)).zipAll(inner(s1,bs),None,Nil).map{
      case (Some(x),xs) => (x,b) ::xs
      case (None,xs) => xs
    }
  }
  inner(s1,s2).flatten
}

combine(Stream(0,1),Stream(0,1,2)) take 6 foreach println
//(0,0)
//(1,0)
//(0,1)
//(1,1)
//(0,2)
//(1,2)
```

This seems to work for our terminating streams but what about infinite streams? In fact it blows up due to a stack overflow error. It turns out that zipAll is inherited from `IterableLike` rather than `Stream` and is therefore not lazy: But worse than that, even when I implement zipAll lazily, i.e.: 

```scala
def zipAllFixed[A,B](s1: Stream[A], s2: Stream[B], thisElem: A, thatElem: B): Stream[(A,B)] = (s1, s2) match {
  case (Stream.Empty, _) => s2.map(b => (thisElem,b))
  case (_, Stream.Empty) => s1.map(a => (a,thatElem))
  case (a#:: as, b#:: bs) => (a,b) #:: zipAllFixed(as,bs,thisElem,thatElem)
}
```

so I can use it in my function like this:

```scala
def combine[A,B](s1: Stream[A], s2: Stream[B]): Stream[(A,B)] = {
  def inner(s1: Stream[A], s2: Stream[B]): Stream[List[(A,B)]] = (s1, s2) match {
    case (Stream.Empty,_) => Stream.empty
    case (_,Stream.Empty) => Stream.empty
    case (a #:: as, b#:: bs) => List((a,b)) #:: zipAllFixed(as.map(Some(_)),inner(s1,bs),None,Nil).map{
      case (Some(x),xs) => (x,b) ::xs
      case (None,xs) => xs
    }
  }
  inner(s1,s2).flatten
}
```

It still blows up. And I know it's not zipAllFixed, because I tested it. The only thing I can reason about it is that Stream.Cons is not behaving correctly, as that's the only recursive part and recursion is showing up in the stack trace.

Implementing identically in haskell: (Note I am not a haskell developer, so this may not be idiomatic haskell)

```haskell
zipAll :: [a] -> [b] -> a -> b -> [(a,b)]
zipAll [] [] _ _ = []
zipAll (a:as) [] da db = (a,db) : zipAll as [] da db
zipAll [] (b:bs) da db = (da,b) : zipAll [] bs da db
zipAll (a:as) (b:bs) da db = (a,b) : zipAll as bs da db

combine :: [a] -> [b] -> [(a,b)]
combine as bs = concat (go as bs)
  where
    go [] _ = []
    go _ [] = []
    go (a:as) (b:bs) = [(a,b)] : rest 
      where
        rest = map appendAB zipped
        zipped = zipAll justAs dropB Nothing []
        justAs = map Just as
        dropB = go (a:as) bs
        appendAB (Just a, abs) = (a,b) : abs
        appendAB (Nothing, abs) = abs 
```

This works fine in the infinite and the terminating cases. So what gives, Scala?
