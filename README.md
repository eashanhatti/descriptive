descriptive
=====

Self-describing consumers/parsers

[Haddocks](http://chrisdone.com/descriptive/)

There are a variety of Haskell libraries which are implementable
through a common interface: self-describing parsers:

* A formlet is a self-describing parser.
* A regular old text parser can be self-describing.
* A command-line options parser is a self-describing parser.
* A MUD command set is a self-describing parser.
* A JSON API can be a self-describing parser.

Consumption is done in this data type:

``` haskell
data Consumer s d a
```

To make a consumer, this combinator is used:

``` haskell
consumer :: (s -> (Description d,s))
         -> (s -> (Either (Description d) a,s))
         -> Consumer s d a
```

The first argument generates a description based on some state. The
state is determined by whatever use-case you have. The second argument
parses from the state, which could be a stream of bytes, a list of
strings, a Map, a Vector, etc. You may or may not decide to modify the
state during generation of the description and during parsing.

To use a consumer or describe what it does, these are used:

``` haskell
consume :: Consumer s d a -> s -> (Either (Description d) a,s)
describe :: Consumer s d a -> s -> (Description d,s)
```

A description is like this:

``` haskell
data Description a
  = Unit !a
  | Bounded !Integer !Bound !(Description a)
  | And !(Description a) !(Description a)
  | Sequence [Description a]
  | Wrap a (Description a)
  | None
  deriving (Show)
```

You configure the `a` for your use-case, but the rest is generatable
by the library. Afterwards, you can make your own pretty printing
function, which may be to generate an HTML form, to generate a
commandline `--help` screen, a man page, API docs for your JSON
parser, a text parsing grammar, etc. For example:

``` haskell
describeParser :: Consumer [Char] Text Demo -> Text
describeForm :: Consumer (Map Text Text) (Html ()) Login -> Html ()
describeArgs :: Consumer [Text] CmdArgs MyApp -> CmdArgs
```

See below for some examples of this library.

## Parsing characters

See `Descriptive.Char`.

``` haskell
λ> describe (zeroOrMore (char 'k') <> string "abc") mempty
(And (Bounded 0 UnlimitedBound (Unit "k"))
     (Sequence [Unit "a",Sequence [Unit "b",Sequence [Unit "c",Sequence []]]])
,"")
λ> consume (zeroOrMore (char 'k') <> string "abc") "kkkabc"
(Right "kkkabc","")
λ> consume (zeroOrMore (char 'k') <> string "abc") "kkkab"
(Left (Unit "a character"),"")
```

## Validating forms with named inputs

See `Descriptive.Form`.

``` haskell
λ> describe ((,) <$> input "username" <*> input "password") mempty
(And (Unit (Input "username")) (Unit (Input "password")),fromList [])

λ> consume ((,) <$>
            input "username" <*>
            input "password")
           (M.fromList [("username","chrisdone"),("password","god")])
(Right ("chrisdone","god")
,fromList [("password","god"),("username","chrisdone")])
```

Conditions on two inputs:

``` haskell
login = validate "confirmed password (entered the same twice)"
                 (\(x,y) ->
                    if x == y
                       then Just y
                       else Nothing)
                 ((,) <$>
                  input "password" <*>
                  input "password2")
```

``` haskell
λ> describe login mempty
(Wrap (Constraint "confirmed password (entered the same twice)")
      (And (Unit (Input "password"))
           (Unit (Input "password2")))
,fromList [])

λ> consume login (M.fromList [("password2","gob"),("password","gob")])
(Right "gob",fromList [("password","gob"),("password2","gob")])
```

## Validating forms with auto-generated input indexes

See `Descriptive.Formlet`.

``` haskell
λ> describe ((,) <$> indexed <*> indexed)
            (FormletState mempty 0)
(And (Unit (Index 0))
     (Unit (Index 1))
,FormletState {formletMap = fromList []
              ,formletIndex = 2})
λ> consume ((,) <$> indexed <*> indexed)
           (FormletState (M.fromList [(0,"chrisdone"),(1,"god")]) 0)
(Right ("chrisdone","god")
,FormletState {formletMap =
                 fromList [(0,"chrisdone"),(1,"god")]
              ,formletIndex = 2})
λ> consume ((,) <$> indexed <*> indexed)
           (FormletState (M.fromList [(0,"chrisdone")]) 0)
(Left (Unit (Index 1))
,FormletState {formletMap =
                 fromList [(0,"chrisdone")]
              ,formletIndex = 2})
```

## Parsing command-line options

See `Descriptive.Options`.

``` haskell
server =
  ((,,,) <$>
   constant "start" <*>
   anyString "SERVER_NAME" <*>
   flag "dev" "Enable dev mode?" <*>
   arg "port" "Port to listen on")
```

``` haskell
λ> describe server []
(And (And (And (Unit (Constant "start"))
               (Unit (AnyString "SERVER_NAME")))
          (Unit (Flag "dev" "Enable dev mode?")))
     (Unit (Arg "port" "Port to listen on"))
,[])
λ> consume server ["start","any","--port","1234","-fdev"]
(Right ("start","any",True,"1234"),["--port","1234","-fdev"])
λ> consume server ["start","any","--port","1234"]
(Right ("start","any",False,"1234"),["--port","1234"])
λ> consume server ["start","any"]
(Left (Unit (Arg "port" "Port to listen on")),[])
```

## Self-documenting JSON parser

See `Descriptive.JSON`.

``` haskell
-- | Submit a URL to reddit.
data Submission =
  Submission {submissionToken :: !Integer
             ,submissionTitle :: !Text
             ,submissionComment :: !Text
             ,submissionSubreddit :: !Integer}
  deriving (Show)

submission :: Consumer Value Doc Submission
submission =
  obj "Submission"
      (Submission
        <$> key "token" (integer "Submission token; see the API docs")
        <*> key "title" (string "Submission title")
        <*> key "comment" (string "Submission comment")
        <*> key "subreddit" (integer "The ID of the subreddit"))

sample :: Value
sample =
  toJSON (object
            ["token" .= 123
            ,"title" .= "Some title"
            ,"comment" .= "This is good"
            ,"subreddit" .= 234214])

badsample :: Value
badsample =
  toJSON (object
            ["token" .= 123
            ,"title" .= "Some title"
            ,"comment" .= 123
            ,"subreddit" .= 234214])
```

``` haskell
λ> describe submission (toJSON ())
(Wrap (Struct "Submission")
      (And (And (And (Wrap (Key "token")
                           (Unit (Integer "Submission token; see the API docs")))
                     (Wrap (Key "title")
                           (Unit (Text "Submission title"))))
                (Wrap (Key "comment")
                      (Unit (Text "Submission comment"))))
           (Wrap (Key "subreddit")
                 (Unit (Integer "The ID of the subreddit"))))
,Array (fromList []))

λ> consume submission sample
(Right (Submission {submissionToken = 123
                   ,submissionTitle = "Some title"
                   ,submissionComment = "This is good"
                   ,submissionSubreddit = 234214})
,Object (fromList [("token",Number 123.0)
                  ,("subreddit",Number 234214.0)
                  ,("title",String "Some title")
                  ,("comment",String "This is good")]))

λ> consume submission badsample
(Left (Wrap (Struct "Submission")
            (Wrap (Key "comment")
                  (Unit (Text "Submission comment"))))
,Object (fromList [("token",Number 123.0)
                  ,("subreddit",Number 234214.0)
                  ,("title",String "Some title")
                  ,("comment",Number 123.0)]))
```

The bad sample yields an informative message that:

* The error is in the Submission object.
* The key "comment".
* The type of that key should be a String and it should be a
  Submission comment (or whatever invariants you'd like to mention).

## Parsing Attempto Controlled English for MUD commands

TBA. Will use
[this package](http://chrisdone.com/posts/attempto-controlled-english).

With ACE you can parse into:

``` haskell
parsed complV "<distrans-verb> a <noun> <prep> a <noun>" ==
Right (ComplVDisV (DistransitiveV "<distrans-verb>")
                  (ComplNP (NPCoordUnmarked (UnmarkedNPCoord anoun Nothing)))
                  (ComplPP (PP (Preposition "<prep>")
                               (NPCoordUnmarked (UnmarkedNPCoord anoun Nothing)))))
```

Which I can then further parse with `descriptive` to yield
descriptions like:

> <verb-phrase> [<noun-phrase> ..]

Or similar.

## Producing questions and consuming the answers in Haskell

TBA. Will be a generalization of
[this type](https://github.com/chrisdone/exercise/blob/master/src/Exercise/Types.hs#L20).

It is a library which I am working on in parallel which will ask the
user questions and then validate the answers. Current output is like
this:

``` haskell
λ> describe (greaterThan 4 (integerExpr (parse id expr exercise)))
an integer greater than 4
λ> eval (greaterThan 4 (integerExpr (parse id expr exercise))) $(someHaskell "x = 1")
Left expected an expression, but got a declaration
λ> eval (greaterThan 4 (integerExpr (parse id expr exercise))) $(someHaskell "x")
Left expected an integer, but got an expression
λ> eval (greaterThan 4 (integerExpr (parse id expr exercise))) $(someHaskell "3")
Left expected an integer greater than 4
λ> eval (greaterThan 4 (integerExpr (parse id expr exercise))) $(someHaskell "5")
Right 5
```

This is also couples description with validation, but I will probably
rewrite it with this `descriptive` library.
