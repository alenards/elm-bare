## How do you use `recursive`?
The trick to understanding the `recursive` codec is: pretend you are already done.
When the function you pass to `recursive` is called the argument is the finished `Codec`.

An example may be worth a thousand words:

```elm
import Codec.Bare as Codec exposing (Codec)

type Peano
    = Peano (Maybe Peano)

peanoCodec : Codec Peano
peanoCodec =
    Codec.recursive
        (\finishedCodec ->
            Codec.maybe finishedCodec
                |> Codec.map Peano (\(Peano p) -> p)
        )
```

## Why does `map` take two opposite functions?
One is used for the encoder, the other for the decoder

## How do I build `Codec`s for custom types?
You start building with `taggedUnion` which needs the pattern matcher for your type as an argument.

The pattern matcher is just the most generic `case ... of` possible for your type.

You then chain `variantX` calls for every alternative (in the same order as the pattern matcher).

You end with a call to `buildTaggedUnion`.

An example:

```elm
import Codec.Bare as Codec exposing (Codec)

type Semaphore
    = Red Int String Bool
    | Yellow Float
    | Green

semaphoreCodec : Codec Semaphore
semaphoreCodec =
    Codec.taggedUnion
        (\redEncoder yellowEncoder greenEncoder value ->
            case value of
                Red i s b ->
                    redEncoder i s b

                Yellow f ->
                    yellowEncoder f

                Green ->
                    greenEncoder
        )
        |> Codec.variant3 0 Red Codec.int Codec.string Codec.bool
        |> Codec.variant1 1 Yellow Codec.f64
        |> Codec.variant0 2 Green
        |> Codec.buildTaggedUnion
```

A second example combining `recursive` and `taggedUnion`:

```elm
import Codec.Bare as Codec exposing (Codec)

type Tree a
    = Node (List (Tree a))
    | Leaf a

treeCodec : Codec a -> Codec (Tree a)
treeCodec leafCodec =
    Codec.recursive
        (\finishedCodec ->
            let
                match nodeEncoder leafEncoder tree =
                    case tree of
                        Node cs ->
                            nodeEncoder cs

                        Leaf x ->
                            leafEncoder x
            in
            Codec.taggedUnion match
                |> Codec.variant1 0 Node (Codec.list finishedCodec)
                |> Codec.variant1 1 Leaf leafCodec
                |> Codec.buildTaggedUnion
        )
```

## If there's no `oneOf` or `optionalField`, how is versioning done?
If you want your `Codec`s to evolve over time and still be able to decode old 
encoded data, the recommended approach is to treat the different versions as a custom type.
Then you can write a `taggedUnion` Codec for those possible versions and use `map` to convert it to the data structure used within your app.

An example:
```elm
import Codec.Bare as Codec exposing (Codec)

{-| The gps coordinate we use internally in our application
-}
type alias GpsCoordinate =
    ( Float, Float )

type GpsVersions
    = GpsV1 String -- Old naive way of storing GPS coordinates
    | GpsV2 ( Float, Float ) -- New better way

gpsV1Codec =
    Codec.string

gpsV2Codec =
    Codec.tuple Codec.float64 Codec.float64

gpsCodec : Codec GpsCoordinate
gpsCodec =
    Codec.taggedUnion
        (\gpsV1Encoder gpsV2Encoder value ->
            case value of
                GpsV1 text ->
                    gpsV1Encoder text

                GpsV2 tuple ->
                    gpsV2Encoder tuple
        )
        |> Codec.variant1 1 GpsV1 gpsV1Codec
        |> Codec.variant1 2 GpsV2 gpsV2Codec
        |> Codec.buildTaggedUnion
        |> Codec.map
            (\value ->
                case value of
                    GpsV1 text ->
                        convertGpsV1ToGpsCoordinate text

                    GpsV2 tuple ->
                        tuple -- No conversion needed here
            )
            (\value -> GpsV2 value)
```

