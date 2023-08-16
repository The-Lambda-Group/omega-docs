Here’s the setup: There are five houses, each painted a different color. In each house lives a person with a different nationality. These five owners drink a certain type of beverage, smoke a certain brand of cigar, and keep a certain pet. No owners have the same pet, smoke the same cigar, or drink the same beverage.

The question is, **who owns the fish?**

Here are the clues:

- The Brit lives in the red house
- The Swede keeps dogs as pets
- The Dane drinks tea
- The green house is on the left of the white house
- The green house’s owner drinks coffee
- The person who smokes Pall Mall rears birds
- The owner of the yellow house smokes Dunhill
- The man living in the center house drinks milk
- The Norwegian lives in the first house
- The man who smokes blends lives next to the one who keeps cats
- The man who keeps horses lives next to the man who smokes Dunhill
- The owner who smokes BlueMaster drinks beer
- The German smokes Prince
- The Norwegian lives next to the blue house
- The man who smokes blend has a neighbor who drinks water

```clj
(:- (next List A B)
    (nth List IndexA A)
    (nth List IndexB B)
    (or (+ IndexA 1 IndexB)
        (+ IndexB 1 IndexA)))

(:- (einstein Result)
    (length Houses 5)
    (contains Houses ["brit", _, _, _, "red"])
    (contains Houses ["swede", "dogs", _, _, _])
    (contains Houses ["dane", _, _, "tea", _])
    (next Houses [_, _, _, _, "green"] [_, _, _, _, "white"])
    (contains Houses [_, _, _, "coffee", "green"])
    (contains Houses ["birds", _, "pallmall", _, _])
    (contains Houses [_, _, "dunhill", _, "yellow"])
    (nth Houses 2 [_, _, _, "milk", _])
    (nth Houses 0 ["norwegian", _, _, _, _])
    (next Houses
          [_, _, "blends", _, _]
          ["cats", _, _, _, _])
    (next Houses
          ["horses", _, _, _, _]
          [_, _, "dunhill", _, _])
    (contains Houses [_, _, "bluemaster", "beer", _])
    (contains Houses ["german", _, "prince", _, _])
    (next Houses
          ["norwegian", _, _, _, _]
          [_, _, _, _, "blue"])
    (next Houses
          [_, _, "blend", _, _]
          [_, _, _, "water", _])
    (contains Houses Result)
    (= Result ["fish", _, _, _, _]))
```