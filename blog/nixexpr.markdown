---
title: "Introducing nixexpr: Nix expressions for JavaScript"
date: 2023-08-11
tags: [nix, javascript, nodejs, npm, cursed]
---

<xeblog-hero photo ai="Nikon D3300, photo by Xe Iaso" file="eifel-tower2" prompt="A picture of the tip of the Eifel Tower facsimilie in Las Vegas with a partially cloudy sky"></xeblog-hero>

As a regular reminder, it is a bad idea to give me ideas. Today's bad idea is
brought to you by managerial nerd sniping, insomnia, and the letter "Q".

At a high level: writing complicated data structures in JavaScript kinda sucks.
Here's an example of the kinds of things that I've been writing as I go down the
[ElasticSearch tour-de-insanite](/blog/elasticsearch):

```javascript
{
  highlight: {
    pre_tags: ['<em>'],
    post_tags: ['</em>'],
    require_field_match: false,
    fields: {
      body_content: {
        fragment_size: 200,
        number_of_fragments: 1,
      },
    },
  },
}
```

This works, this is perfectly valid code. It creates an object that has a few
nested layers of stuff in it, but overall I just don't like how it _looks_. I
think it looks superfluous. What if we could make it look a little bit nicer?
How about something like this?

```nix
{
  highlight = {
    pre_tags = [ "em" ];
    post_tags = [ "</em>" ];
    require_fields_match = false;
    fields.body_content.fragment_size = 200;
    fields.body_content.number_of_fragments = 1;
  };
}
```

This is a Nix expression. It's a data structure that looks like JSON, but you
have the power of a programming language at your fingertips. Note the difference
between these two parts:

```javascript
{
  fields: {
    body_content: {
      fragment_size: 200,
      number_of_fragments: 1,
    },
  },
}
```

```
{
  fields.body_content.fragment_size = 200;
  fields.body_content.number_of_fragments = 1;
}
```

These are semantically equal, but you don't have to use so much indentation and
layering. These settings are all related, so it makes sense that the way that
you use them is on the same level as the way that you define them.

If you want to try out this awesome power for yourself,
[Install Nix](https://github.com/DeterminateSystems/nix-installer) and then add
`@xeserv/nixexpr` to your JavaScript dependencies.

```bash
npm install --save @xeserv/nixexpr
```

Then you can use it like this:

```javascript
import { nix } from "@xeserv/nixexpr";

const someValue = "this is a string";

const myData = nix`{
    hello = "world";
    someValue = ${someValue};
}`;

console.log(myData);
```

I originally wrote this
[in Go](https://github.com/Xe/x/blob/c822591c5a46ad9e8f13d14ac96d2e9d26e7c828/cmd/yeet/main.go#L115-L151)
for my scripting automation tool named
[yeet](https://www.urbandictionary.com/define.php?term=Yeet), but I think it's
generically useful enough to exist in its own right in JavaScript. I think that
there's a lot of things that the JavaScript ecosystem can gain from Nix, and I'm
excited to see what people do with this.

This was made so I could write scripts like this:

```javascript
// snipped for brevity
const url = slug.push("within.website");
const hash = nix.hashURL(url);

const expr = nix.expr`{ stdenv }:

stdenv.mkDerivation {
  name = "within.website";
  src = builtins.fetchurl {
    url = ${url};
    sha256 = ${hash};
  };

  phases = "installPhase";

  installPhase = ''
    tar xf $src
    mkdir -p $out/bin
    cp web $out/bin/withinwebsite
    cp config.ts $out/config.ts
  '';
}
`;
```

And then I'd be able to put that Nix expression into a file. I'll get into more
details about this in a future post.

<xeblog-conv name="Mara" mood="hacker">Something something this isn't best
practice something something this is a hack to make dealing with a legacy
deployment easier something something</xeblog-conv>

## How it works

This is a very cheeky library, and it's all powered by one of the most fun to
abuse Nix functions ever:
[builtins.fromJSON](https://nixos.org/manual/nix/stable/language/builtins.html#builtins-fromJSON).
This function takes a string and turns it into a Nix value at the interpreter
level and it's part of the callpath for turning a string into an integer in Nix.
It's an amazingly powerful function in its own right, but it gets even more fun
when we bring JavaScript into the mix.

Any JavaScript data value (simple objects, strings, numbers, etc) can be
formatted as JSON with the `JSON.stringify` function:

```
> JSON.stringify({"hi": "there"})
'{"hi":"there"}'
```

This includes strings. So if we use `JSON.stringify` to convert it to a JSON
string, then string encode it again, we can inject arbitrary JavaScript code
into Nix expressions:

```js
let formattedValue = `(builtins.fromJSON ${
  JSON.stringify(JSON.stringify(value))
})`;
```

The most horrifying part about this hack is that it works.

## What's next?

If this ends up getting used, I may try and make "fast paths" for strings and
numbers so that they don't have to go through the JSON encoding/decoding
process. But so far this works well enough for my purposes.
