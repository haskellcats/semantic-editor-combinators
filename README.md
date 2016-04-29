# Semantic Editor Combinators

This is a demo of a few forms of what Conal Elliott on [his blog][1] calls
semantic editor combinators. It is a functional programming technique for
modifying a complex immutable nested data structure. The long name semantic
editor combinator comes from the following three characteristics:

- They are editors because they modify some data type. An editor for type T has
  type T -> T.
- They are combinators just because they easily compose with each other and
  with other things.
- The tag "semantic" comes from their ability to not only modify a data
  structure but to make modifications based on a view of the structure. This
sort of view is what Conal calls the semantic value, or tangible value. By
contrast the actual structure is called the syntactic value.

"Functional references" is an earlier name for this technique.

## List example

To modify an element of a list at some index you have to traverse to that
index, apply the modification, then append the unchanged rest of the list.
I'll write a function to do it and I will choose the name and argument order
in an "interesting" way.

```
index :: Int -> (a -> a) -> [a] -> [a]
index 0 f (x:xs) = f x : xs
index i f (x:xs) = x : index (i-1) f xs
index i f [] = error "index out of bounds"

> index 2 (+100) [1,2,3,4,5]
[1,2,103,4,5]
```

Check the type of index when you leave off some of the arguments.

```
index :: Int -> (a -> a) -> [a] -> [a]
index 0 :: (a -> a) -> [a] -> [a]
index 0 myEdit :: [a] -> [a]
```

It is clear that index 0 takes a modification function and returns an editor.
More than that, you can compose several of these to get an editor that
targets a location in a nested list.

```
index 2 . index 3 :: (b -> b) -> [[b]] -> [[b]]

> (index 2 . index 3) (+100) [[1,2],[3,4],[5,6,7,8,9],[10,11]]
[[1,2],[3,4],[5,6,7,108,9],[10,11]]
```

Here type inference has determined that a = [b]. You can continue tacking
on these list indexes to get deeper into a nest of lists. The path to the
target of the edit is read from left to right.

```
index 2 . index 3 . index 0 :: (c -> c) -> [[[c]]] -> [[[c]]]
```

Deeply nested lists and indexing aren't the greatest ways to program with
functional data, but these examples show the basic pattern that semantic
editor combinators follow. Next we will see that it works on other data
structures, functions, and even "virtual values".

## Record example

Record update syntax is the bane of many Haskell programmers. Luckily editor
combinators are much easier to use and don't require any extra language
features. 

```
data Rank = Bishop | Knight | Rook | King
data Employee = Emp { name :: String, age :: Int, rank :: Rank }

onName f e = e { name = f (name e) }
onAge  f e = e { age  = f (age e)  }
onRank f e = e { rank = f (rank e) }

:t onRank
onRank :: (Rank -> Rank) -> Employee -> Employee
```

Unfortunately you have to have a different editor combinator for each field.
Fortunately these can be automatically generated using template Haskell.
They work the same way as the list editor, in fact they work together:

```
let db = [Emp "ron" 40 Bishop, Emp "peter" 35 Rook, Emp "heather" 50 King]
db :: [Employee]

> (index 1 . onRank) promote db
[Emp "ron" 40 Bishop, Emp "peter" 35 King, Emp "heather" 50 King]

> (index 2 . onName) reverse db
[Emp "ron" 40 Bishop, Emp "peter" 35 Rook, Emp "rehtaeh" 50 King]
```

If you had nested records, then these sort of field editors would let you
reach as deep as you need to to modify a particular deep field. Also if there
are lists or other containers along the way, editors for those will naturally
come in handy.

```
data Job = Jo { title :: String, powers :: [Power] }
data Item = It { name :: Maybe String, weight :: Int }
data Player = Pl { job :: Job, items :: [Item] }

let db =
  [Pl
    (Jo "paladin" [TurnUndead])
    [ It (Just "flute") 11
    , It (Just "sword") 66
    , It Nothing 13]
  ]

> (index 0 . onItems . map . onName . fmap) toUpper db
[Pl
  (Jo "paladin" [TurnUndead])
  [ It (Just "Flute") 11
  , It (Just "Sword") 66
  , It Nothing 13]
]
```

In the last example more than one thing was updated because of the use of map.
Actually you can make editors to target many things at once in an arbitrarily
complex way. Here is a basic example which only modifies every nth item of a
list.

```
onEvery :: Int -> (a -> a) -> [a] -> [a]
onEvery n f xs = go 0 xs where
  go i [] = []
  go i (x:xs) | i `mod` n == 0 = f x : go (i+1) xs
              | otherwise      =   x : go (i+1) xs

> onEvery 3 (const 0) [1,2,3,4,5,6,7,8,9]
[0,2,3,0,5,6,0,8,9]
```

## Function example

You aren't limited to editing just data values. Here is an editor combinator
for modifying the output of a function.

```
ret :: (a -> a) -> (b -> a) -> (b -> a)
ret f g = f . g

data Cat = Cat { sleep :: IO (), meow :: Volume -> IO (), puke :: Volume -> IO Puke }
setColor :: Color -> Puke -> Puke

let db = [snowball, furball, screwball]
let db' = (index 2 . onPuke . ret . fmap) (setColor Blue) db

> puke (db' !! 2) 13
13 blue puke appears
```

Similarly you can edit the argument before it gets to a function.

```
arg :: (a -> a) -> (a -> b) -> (a -> b)
arg f g = g . f
```

## Other simple examples

```
left  :: (a -> a) -> (a,b) -> (a,b)
right :: (b -> b) -> (a,b) -> (a,b)
onValue  :: k -> (a -> a) -> Map k a -> Map k a
onValues :: [k] -> (a -> a) -> Map k a -> Map k a
onFirst :: (a -> a) -> Seq a -> Seq a
onLast  :: (a -> a) -> Seq a -> Seq a
onMatching :: (a -> Bool) -> (a -> a) -> [a] -> [a]
```

## "Virtual path" example

So far the editors only modify the actual content of a data structure. But
they can also dig into regions of the data that are only there in our
imagination. This simple example modifies a character by operating on its
code point.

```
asCode :: (Int -> Int) -> Char -> Char
asCode f = chr . f . ord

> (map . asCode) (+1) "hello world"
"ifmmp!xpsme"
```

We edited the character by editing the numeric value! We can go farther and
edit the bits of a number.

```
asBits :: ([Bool] -> [Bool]) -> Int -> Int
asBits f = pack . f . unpack

> (index 3 . asCode . asBits . index 2) not "hello world"
"helho world"
```

With any editor you have that digs into a virtual value, or "semantic value",
you can then use all the editors you already made for those values and treat
values past that point as real. When the data structure is finally updated
those values will all be gone, but the updates will persist as if they were
really there!

[1]: http://conal.net/blog/posts/semantic-editor-combinators
