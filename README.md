<img src="http://i.imgur.com/0h9dFhl.png" width="300px"/>

[![Build Status](https://travis-ci.org/qfpl/waargonaut.svg?branch=master)](https://travis-ci.org/qfpl/waargonaut)

# Waargonaut

_NB:_ **BETA Release**

Flexible, precise, and efficient JSON decoding/encoding library. This package
provides a plethora of tools for decoding, encoding, and manipulating JSON data.

## Features

* Fully RFC compliant, with property based testing used to ensure the desired
  invariants are preserved.

* Encoders and Decoders are values, they are not tied to a typeclass and as such
  you are not tied to a single interpretation of how a particular type
  "_should_" be handled.
  
* No information is discarded on parsing. Trailing whitespace, and any
  formatting whitespace (carriage returns etc) are all preserved. 

* A history keeping zipper is used for Decoding, providing precise control of
  how _you_ decode _your_ JSON data. With informative error messages if things
  don't go according to plan.

* Flexible and expressive Decoder & Encoder functions let you parse and build
  the JSON structures _you_ require, with no surprises.

* BYO parsing library, the parser built into Waargonaut does not tie you to a
  particular parsing library. With the caveat that your parsing library must
  have an instance of `CharParser` from the [parsers](https://hackage.haskell.org/package/parsers) package.

* Generic functions are provided to make the creation of Encoders and Decoders
  are bit easier. However these _are_ tied to typeclasses, so they do come with
  some assumptions.

* Lenses, Prisms, and Traversals are provided to allow you to investigate and
  manipulate the JSON data structures to your hearts content, without breaking
  the invariants.


* _Soon_ there will be a companion package for this library that leverages the awesome
  work on succinct data structures by John Ky and [Haskell Works](https://github.com/haskell-works/). 

  Providing the same zipper capabilities and property based guarantees, but with
  all the speed, efficiency, and streaming capabilities that succinct data
  structures have to offer. That's the idea, anyway.
  
  **NB:** It exists for the keen and the brave [waargonaut-succinct-ds](https://github.com/qfpl/waargonaut-succinct-ds).

## Example

- Data Structure:
```
data Image = Image
  { _imageWidth    :: Int
  , _imageHeight   :: Int
  , _imageTitle    :: Text
  , _imageAnimated :: Bool
  , _imageIDs      :: [Int]
  }
```

- Encoder:
```
encodeImage :: Applicative f => Encoder f Image
encodeImage = E.mapLikeObj $ \img ->
    E.intAt "Width" (_imageWidth img)
  . E.intAt "Height" (_imageHeight img)
  . E.textAt "Title" (_imageTitle img)
  . E.boolAt "Animated" (_imageAnimated img)
  . E.listAt E.int "IDs" (_imageIDs img)
```

- Decoder:
```
imageDecoder :: Monad f => Decoder f Image
imageDecoder = D.withCursor $ \curs ->
  Image
    <$> D.fromKey "Width" D.int curs
    <*> D.fromKey "Height" D.int curs
    <*> D.fromKey "Title" D.text curs
    <*> D.fromKey "Animated" D.bool curs
    <*> D.fromKey "IDs" (D.list D.int) curs
```

### Zippers

Waargonaut uses zippers for it's decoding which allows for precise control in
how you interrogate your JSON input. Take JSON structures and decode them
precisely as you require:

##### Input:

```JSON
["a","fred",1,2,3,4]
```

##### Data Structure:

```
data Foo = (Char,String,[Int])
```

##### Decoder:

The zipper starts the very root of the JSON input, we tell it to move 'down'
into the first element.
```haskell
fooDecoder :: Monad f => Decoder f Foo
fooDecoder = D.withCursor $ \cursor -> do
  firstElem <- D.down "array" cursor
```
From the first element we can then decode the focus of the zipper using a
specific decoder:
```
  aChar <- D.focus D.unboundedChar fstElem
```
The next thing we want to decode is the second element of the array, so we
move right one step or tooth, and then attempt to decode a string at the
focus.
```
  aString <- D.moveRight1 fstElem >>= D.focus D.string
```
Finally we want to take everything else in the list and combine them into a
single list of Int values. Starting from the first element, we move right
two positions (over the char and the string elements), then we use one of
the provided decoder functions that will repeatedly move in a direction and
combine all of the elements it can until it can no longer move.
```
  aIntList <- D.moveRightN 2 fstElem >>= D.rightwardSnoc [] D.int
```
Lastly, we build the Foo using the decoded values.
```
  pure $ Foo (aChar, aString, aIntList)
```

The zipper stores the history of your movements, so any errors provide
information about the path they took prior to encountering an error. Making
debugging precise and straight-forward.

### Property Driven Development

This library is built to parse and produce JSON in accordance with the [RFC
8259](https://tools.ietf.org/html/rfc8259) standard. The data structures,
parser, and printer are built to comply with the following properties:

```
parse . print = id
```
This indicates that any JSON produced by this library will be parsed back in as
the exact data structure that produced it. This includes whitespace such as
carriage returns and trailing whitespace. There is no loss of information.

```
print . parse . print = print
```
This states that the printed form of the JSON will not change will be identical
after parsing and then re-printing. There is no loss of information.

This provides a solid foundation to build upon.

### TODO(s)

In no particular order...

- [ ] improve/bikeshed encoding object api 
- [ ] gather feedback on tests/benchmarks that matter to people
- [ ] provide testing functions so users can be more confident in their Encoder/Decoder construction
- [ ] documentation in the various modules to explain any weirdness or things that users may consider to be 'missing' or 'wrong'.
- [ ] provide greater rationale behind lack of reliance in typeclasses for encoding/decoding
- [ ] provide functions to add preset whitespace layouts to encoded json.
