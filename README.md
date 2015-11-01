# Minify [![Build Status](https://travis-ci.org/tdewolff/minify.svg?branch=master)](https://travis-ci.org/tdewolff/minify) [![GoDoc](http://godoc.org/github.com/tdewolff/minify?status.svg)](http://godoc.org/github.com/tdewolff/minify) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify)](http://gocover.io/github.com/tdewolff/minify)

[![Join the chat at https://gitter.im/tdewolff/minify](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/tdewolff/minify?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

**WARNING: an API change is happening soon, be aware or continue using the old API in tag v1.0.0**

Minify is a minifier package written in [Go][1]. It has build-in HTML5, CSS3, JS, JSON, SVG and XML minifiers and provides an interface to implement any minifier. Minification is the process of removing bytes from a file (such as whitespace) without changing its output and therefore speeding up transmission over the internet. The implemented minifiers are high performance and streaming (which implies O(n)).

It associates minification functions with mime types, allowing embedded resources (like CSS or JS in HTML files) to be minified too. The user can add any mime-based implementation. Users can also implement a mime type using an external command (like the ClosureCompiler, UglifyCSS, ...). It is possible to pass parameters through the mimetype to specify the charset for example.

Bottleneck for minification is mainly io and can be significantly sped up by having the file loaded into memory and providing a `Bytes() []byte` function like `bytes.Buffer` does.

**Table of Contents**

[Online live demo](http://pi.tacodewolff.nl:8080/minify) running on a Raspberry Pi 2.

[Command-line-interface](https://github.com/tdewolff/minify/tree/master/cmd/minify) executable `minify` provided for tooling.

- [Minify](#minify--)
	- [Prologue](#prologue)
	- [Comparison](#comparison)
		- [Alternatives](#alternatives)
	- [HTML](#html--)
		- [Beware](#beware)
	- [CSS](#css--)
	- [JS](#js--)
	- [JSON](#json--)
	- [SVG](#svg--)
	- [XML](#xml--)
	- [Installation](#installation)
	- [Usage](#usage)
		- [New](#new)
		- [From reader](#from-reader)
		- [From bytes](#from-bytes)
		- [From string](#from-string)
		- [Custom minifier](#custom-minifier)
		- [Mediatypes](#mediatypes)
	- [Examples](#examples)
		- [Common minifiers](#common-minifiers)
		- [Custom minifier](#custom-minifier-1)
		- [ResponseWriter](#responsewriter)
	- [License](#license)

**Roadmap**

* [x] HTML parser and minifier
* [x] CSS parser and minifier
* [x] Command line tool
* [x] JSON parser and minifier
* [x] JS lexer and basic minifier
* [x] Improve CSS parser to implement the same technique as HTML/JSON does (ie. a lightweight parser)
* [x] XML parser and minifier according to the specs
* [x] Optimize and test JSON minification
* [x] Optimize and test XML minification
* [x] Optimize and test CSS minification
* [x] SVG minifier using the XML parser
* [ ] Expand SVG minifier using https://github.com/svg/svgo techniques
* [x] Test with https://github.com/dvyukov/go-fuzz, *found >10 bugs*
* [x] Make parsers zero-copy
* [ ] ~~JS lightweight parser~~
* [x] Use ECMAScript 6 for JS lexer instead of 5.1
* [ ] JS minifier with local variable renaming and better semicolon and newline omission
* [ ] Optimize the CSS parser to use the same parsing style as the JS parser
* [ ] Options feature to disable techniques
* [ ] HTML templates minification, e.g. Go HTML templates or doT.js templates etc.

## Prologue
Minifiers or bindings to minifiers exist in almost all programming languages. Some implementations are merely using several regular-expressions to trim whitespace and comments (even though regex for parsing HTML/XML is ill-advised, for a good read see [Regular Expressions: Now You Have Two Problems](http://blog.codinghorror.com/regular-expressions-now-you-have-two-problems/)). Some implementations are much more profound, such as the [YUI Compressor](http://yui.github.io/yuicompressor/), [Google Closure Compiler](https://github.com/google/closure-compiler) for JS and the [HTML Compressor](https://code.google.com/p/htmlcompressor/).

These industry-grade minifiers are written in Java and are generally relatively slow. Futhermore, these tools provide a large number of configurations which is often confusing or not required. Regular-expression based minifiers are slow anyways because they use multiple regular-expressions, each of which parses the complete document. While regular-expressions are overkill (or ill-advised) for parsing HTML/CSS/JS documents, parsing it a number of times is certainly not speeding things up. Other implementations are mostly written in uncompiled languages such as JS, which is great for bindings with [Grunt](http://gruntjs.com/) for example, but catastrophic for the minification speed of large files or projects with many files.

Additionally, many of these minifier either do not follow the specifications or drag a lot of legacy code around. When you are still trying to support IE6 I don't suppose you are squeezing out every bit of performance from your web applications. Supporting old mistakes or work-arounds is not a fairly long-term vision and seldomly justified.

However, implementing an HTML minifier is the bare minimum. HTML documents can contain embedded resources such as CSS, JS and SVG file formats. Thus for increased minification of HTML, other file format minifiers must be present too. A minifier should really handle a number of mediatypes to be successful.

This minifier proves to be that fast and encompassing minifier which stream-minifies files and can minify them concurrently.

## Comparison
HTML (with JS and CSS) minification typically runs at about 35MB/s ~= 120GB/h, depending on the composition of the file.

Website | Original | Minified | Ratio | Time<sup>&#42;</sup>
------- | -------- | -------- | ----- | -----------------------
[Amazon](http://www.amazon.com/) | 463kB | **414kB** | 89% | 13ms
[BBC](http://www.bbc.com/) | 113kB | **96kB** | 85% | 4ms
[StackOverflow](http://stackoverflow.com/) | 201kB | **182kB** | 91% | 6ms
[Wikipedia](http://en.wikipedia.org/wiki/President_of_the_United_States) | 435kB | **410kB** | 94%<sup>&#42;&#42;</sup> | 12ms

<sup>&#42;</sup>These times are measured on my home computer which is an average development computer. The duration varies a lot but it's important to see it's in the 10ms range! The benchmark uses all the minifiers and excludes reading from and writing to the file from the measurement.

<sup>&#42;&#42;</sup>Is already somewhat minified, so this doesn't reflect the full potential of this minifier.

### Alternatives
[HTML Compressor](https://code.google.com/p/htmlcompressor/) performs worse in output size (for HTML and CSS) and speed; it is a magnitude slower. Its whitespace removal is not precise or the user must provide the tags around which can be trimmed.

An alternative library written in Go is [https://github.com/dchest/htmlmin](https://github.com/dchest/htmlmin). It is simpler but slower. Also [https://github.com/omeid/jsmin](https://github.com/omeid/jsmin) contains a port of JSMin, just like this JS minifier, but is slower.

Other alternatives are bindings to existing minifiers written in other languages. These are inevitably more robust and tested but will often be slower. For example, Java-based minifiers incur overhead of starting up the JVM.

## HTML [![GoDoc](http://godoc.org/github.com/tdewolff/minify/html?status.svg)](http://godoc.org/github.com/tdewolff/minify/html) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify/html)](http://gocover.io/github.com/tdewolff/minify/html)

The HTML5 minifier uses these minifications:

- strip unnecessary whitespace and otherwise collapse it to one space
- strip superfluous quotes, or uses single/double quotes whichever requires fewer escapes
- strip default attribute values and attribute boolean values
- strip some empty attributes
- strip unrequired tags (`html`, `head`, `body`, ...)
- strip unrequired end tags (`tr`, `td`, `li`, ... and often `p`)
- strip default protocols (`http:`, `https:` and `javascript:`)
- strip comments (except conditional comments)
- shorten `doctype` and `meta` charset
- lowercase tags, attributes and some values to enhance gzip compression

After recent benchmarking and profiling it became really fast and minifies pages in the 10ms range, making it viable for on-the-fly minification.

However, be careful when doing on-the-fly minification. Minification typically trims off 10% and does this at worst around about 20MB/s. This means users have to download slower than 2MB/s to make on-the-fly minification worthwhile. This may or may not apply in your situation. Rather use caching!

### Whitespace removal
The whitespace removal mechanism collapses all sequences of whitespace (spaces, newlines, tabs) to a single space. It trims all text parts (in between tags) depending on whether it was preceded by a space from a previous piece of text and whether it is followed up by a block element or an inline element. In the former case we can omit spaces while for inline elements whitespace has significance.

Make sure your HTML doesn't depend on whitespace between `block` elements that have been changed to `inline` or `inline-block` elements using CSS. Your layout *should not* depend on those whitespaces as the minifier will remove them. An example is a menu consisting of multiple `<li>` that have `display:inline-block` applied and have whitespace in between them. It is bad practise to rely on whitespace for element positioning anyways!

## CSS [![GoDoc](http://godoc.org/github.com/tdewolff/minify/css?status.svg)](http://godoc.org/github.com/tdewolff/minify/css) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify/css)](http://gocover.io/github.com/tdewolff/minify/css)

Minification typically runs at about 20MB/s ~= 70GB/h.

Library | Original | Minified | Ratio | Time<sup>&#42;</sup>
------- | -------- | -------- | ----- | -----------------------
[Bootstrap](http://getbootstrap.com/) | 134kB | **111kB** | 83% | 5ms
[Gumby](http://gumbyframework.com/) | 182kB | **167kB** | 90% | 8ms

<sup>&#42;</sup>The benchmark excludes the time reading from and writing to a file from the measurement.

The CSS minifier will only use safe minifications:

- remove comments and unnecessary whitespace
- remove trailing semicolons
- optimize `margin`, `padding` and `border-width` number of sides
- shorten numbers by removing unnecessary `+` and zeros and rewriting with/without exponent
- remove dimension and percentage for zero values
- remove quotes for URLs
- remove quotes for font families and make lowercase
- rewrite hex colors to/from color names, or to 3 digit hex
- rewrite `rgb(`, `rgba(`, `hsl(` and `hsla(` colors to hex or name
- replace `normal` and `bold` by numbers for `font-weight` and `font`
- replace `none` &#8594; `0` for `border`, `background` and `outline`
- lowercase all identifiers except classes, IDs and URLs to enhance gzip compression
- shorten MS alpha function
- rewrite data URIs with base64 or ASCII whichever is shorter
- calls minifier for data URI mediatypes, thus you can compress embedded SVG files if you have that minifier attached

It does purposely not use the following techniques:

- (partially) merge rulesets
- (partially) split rulesets
- collapse multiple declarations when main declaration is defined within a ruleset (don't put `font-weight` within an already existing `font`, too complex)
- remove overwritten properties in ruleset (this not always overwrites it, for example with `!important`)
- rewrite properties into one ruleset if possible (like `margin-top`, `margin-right`, `margin-bottom` and `margin-left` &#8594; `margin`)
- put nested ID selector at the front (`body > div#elem p` &#8594; `#elem p`)
- rewrite attribute selectors for IDs and classes (`div[id=a]` &#8594; `div#a`)
- put space after pseudo-selectors (IE6 is old, move on!)

It's great that so many other tools make comparison tables: [CSS Minifier Comparison](http://www.codenothing.com/benchmarks/css-compressor-3.0/full.html), [CSS minifiers comparison](http://www.phpied.com/css-minifiers-comparison/) and [CleanCSS tests](http://goalsmashers.github.io/css-minification-benchmark/). From the last link, this CSS minifier is almost without doubt the fastest and has near-perfect minification rates. It falls short with the purposely not implemented and often unsafe techniques, so that's fine.

## JS [![GoDoc](http://godoc.org/github.com/tdewolff/minify/js?status.svg)](http://godoc.org/github.com/tdewolff/minify/js) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify/js)](http://gocover.io/github.com/tdewolff/minify/js)

The JS minifier is pretty basic. It removes comments, whitespace and line breaks whenever it can. It employs all the rules that [JSMin](http://www.crockford.com/javascript/jsmin.html) does too, but has additional improvements. For example the prefix-postfix bug is fixed.

Minification typically runs at about 40MB/s ~= 150GB/h. Common speeds of PHP and JS implementations are about 100-300kB/s (see [Uglify2](http://lisperator.net/uglifyjs/), [Adventures in PHP web asset minimization](https://www.happyassassin.net/2014/12/29/adventures-in-php-web-asset-minimization/)).

Library | Original | Minified | Ratio | Time<sup>&#42;</sup>
------- | -------- | -------- | ----- | -----------------------
[ACE](https://github.com/ajaxorg/ace-builds) | 630kB | **442kB** | 70% | 16ms
[jQuery](http://jquery.com/download/) | 242kB | **130kB** | 54% | 6ms
[jQuery UI](http://jqueryui.com/download/) | 459kB | **300kB** | 65% | 12ms
[Moment](http://momentjs.com/) | 97kB | **51kB** | 52% | 2ms

<sup>&#42;</sup>The benchmark excludes the time reading from and writing to a file from the measurement.

TODO:
- shorten local variables / function parameters names
- precise semicolon and newline omission

## JSON [![GoDoc](http://godoc.org/github.com/tdewolff/minify/json?status.svg)](http://godoc.org/github.com/tdewolff/minify/json) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify/json)](http://gocover.io/github.com/tdewolff/minify/json)

Minification typically runs at about 75MB/s ~= 270GB/h. It shaves off about 15% of filesize for common indented JSON such as generated by [JSON Generator](http://www.json-generator.com/).

The JSON minifier only removes whitespace, which is the only thing that can be left out.

## SVG [![GoDoc](http://godoc.org/github.com/tdewolff/minify/svg?status.svg)](http://godoc.org/github.com/tdewolff/minify/svg) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify/svg)](http://gocover.io/github.com/tdewolff/minify/svg)

The SVG minifier uses these minifications:

- trim and collapse whitespace between all tags
- strip comments, `doctype`, XML prelude, `metadata`
- strip SVG version
- strip CDATA sections wherever possible
- collapse tags with no content to a void tag
- collapse empty container tags (`g`, `svg`, ...)
- minify style tag and attributes with the CSS minifier
- minify colors
- shorten lengths and numbers and remove default `px` unit
- shorten the `path` data `m` attribute

TODO:
- convert `rect`, `line`, `polygon`, `polyline` to `path`
- convert attributes to style attribute whenever shorter
- use relative instead of absolute positions for path data (need bytes2float)
- merge path data? (same style and no intersection -- the latter is difficult)

## XML [![GoDoc](http://godoc.org/github.com/tdewolff/minify/xml?status.svg)](http://godoc.org/github.com/tdewolff/minify/xml) [![GoCover](http://gocover.io/_badge/github.com/tdewolff/minify/xml)](http://gocover.io/github.com/tdewolff/minify/xml)

Minification typically runs at about 50MB/s ~= 180GB/h.

The XML minifier uses these minifications:

- strip unnecessary whitespace and otherwise collapse it to one space
- strip comments
- collapse tags with no content to a void tag
- strip CDATA sections wherever possible

## Installation
Run the following command

	go get github.com/tdewolff/minify

or add the following imports and run the project with `go get`
``` go
import (
	"github.com/tdewolff/minify"
	"github.com/tdewolff/minify/css"
	"github.com/tdewolff/minify/html"
	"github.com/tdewolff/minify/js"
	"github.com/tdewolff/minify/json"
	"github.com/tdewolff/minify/svg"
	"github.com/tdewolff/minify/xml"
)
```

## Usage
### New
Retrieve a minifier struct which holds a map of mediatype &#8594; minifier functions.
``` go
m := minify.New()
```

The following loads all provided minifiers.
``` go
m := minify.New()
m.AddFunc("text/css", css.Minify)
m.AddFunc("text/html", html.Minify)
m.AddFunc("text/javascript", js.Minify)
m.AddFunc("image/svg+xml", svg.Minify)
m.AddFuncRegexp(regexp.MustCompile("[/+]json$"), json.Minify)
m.AddFuncRegexp(regexp.MustCompile("[/+]xml$"), xml.Minify)
```

### From reader
Minify from an `io.Reader` to an `io.Writer` for a specific mediatype.
``` go
if err := m.Minify(mediatype, w, r); err != nil {
	panic(err)
}
```

Minify HTML, CSS or JS directly from an `io.Reader` to an `io.Writer`. The passed mediatype is not required for these functions, but are filled out for clarity.
``` go
if err := css.Minify(m, "text/css", w, r); err != nil {
	panic(err)
}

if err := html.Minify(m, "text/html", w, r); err != nil {
	panic(err)
}

if err := js.Minify(m, "text/javascript", w, r); err != nil {
	panic(err)
}

if err := json.Minify(m, "application/json", w, r); err != nil {
	panic(err)
}

if err := svg.Minify(m, "image/svg+xml", w, r); err != nil {
	panic(err)
}

if err := xml.Minify(m, "text/xml", w, r); err != nil {
	panic(err)
}
```

### From bytes
Minify from and to a `[]byte` for a specific mediatype.
``` go
b, err = minify.Bytes(m, mediatype, b)
if err != nil {
	panic(err)
}
```

### From string
Minify from and to a `string` for a specific mediatype.
``` go
s, err = minify.String(m, mediatype, s)
if err != nil {
	panic(err)
}
```

### Custom minifier
Add a function for a specific mediatype.
``` go
m.AddFunc(mediatype, func(m minify.Minifier, mediatype string, w io.Writer, r io.Reader) error {
	// ...
	return nil
})
```

Add a command `cmd` with arguments `args` for a specific mediatype.
``` go
m.AddCmd(mediatype, exec.Command(cmd, args...))
```

### Mediatypes
Mediatypes can contain parameters (`type/subtype; key1=val2; key2=val2`). Minifiers can also be added using a regular expression. For example a minifier with `image/.*` will match any image mime.

Mediatypes such as `text/plain; charset=UTF-8` will be processed by `text/plain` or any regexp it matches. The mediatype string is passed to the minifier function which can retrieve the parameters using the standard library `mime.ParseMediaType`.

## Examples
### Common minifiers
Basic example that minifies from stdin to stdout and loads the default HTML, CSS and JS minifiers. Optionally, one can enable `java -jar build/compiler.jar` to run for JS (for example the [ClosureCompiler](https://code.google.com/p/closure-compiler/)). Note that reading the file into a buffer first and writing to a pre-allocated buffer would be faster (but would disable streaming).
``` go
package main

import (
	"log"
	"os"
	"os/exec"

	"github.com/tdewolff/minify"
	"github.com/tdewolff/minify/css"
	"github.com/tdewolff/minify/html"
	"github.com/tdewolff/minify/js"
	"github.com/tdewolff/minify/json"
	"github.com/tdewolff/minify/svg"
	"github.com/tdewolff/minify/xml"
)

func main() {
	m := minify.New()
	m.AddFunc("text/css", css.Minify)
	m.AddFunc("text/html", html.Minify)
	m.AddFunc("text/javascript", js.Minify)
	m.AddFunc("image/svg+xml", svg.Minify)
	m.AddFuncRegexp(regexp.MustCompile("[/+]json$"), json.Minify)
	m.AddFuncRegexp(regexp.MustCompile("[/+]xml$"), xml.Minify)

	// Or use the following for better minification of JS but lower speed:
	// m.AddCmd("text/javascript", exec.Command("java", "-jar", "build/compiler.jar"))

	if err := m.Minify("text/html", os.Stdout, os.Stdin); err != nil {
		panic(err)
	}
}
```

### Custom minifier
Custom minifier showing an example that implements the minifier function interface. Within a custom minifier, it is possible to call any minifier function (through `m minify.Minifier`) recursively when dealing with embedded resources.
``` go
package main

import (
	"bufio"
	"fmt"
	"io"
	"log"
	"strings"

	"github.com/tdewolff/minify"
)

func main() {
	m := minify.New()

	// remove newline and space bytes
	m.AddFunc("text/plain", func(m minify.Minifier, mediatype string, w io.Writer, r io.Reader) error {
		rb := bufio.NewReader(r)
		for {
			line, err := rb.ReadString('\n')
			if err != nil && err != io.EOF {
				return err
			}
			if _, errws := io.WriteString(w, strings.Replace(line, " ", "", -1)); errws != nil {
				return errws
			}
			if err == io.EOF {
				break
			}
		}
		return nil
	})

	out, err := minify.String(m, "text/plain", "Because my coffee was too cold, I heated it in the microwave.")
	if err != nil {
		panic(err)
	}
	fmt.Println(out)
	// Output: Becausemycoffeewastoocold,Iheateditinthemicrowave.
}
```

### ResponseWriter
ResponseWriter example which returns a ResponseWriter that minifies the content and then writes to the original ResponseWriter. Any write after applying this filter will be minified.
``` go
type MinifyResponseWriter struct {
	http.ResponseWriter
	io.Writer
}

func (m MinifyResponseWriter) Write(b []byte) (int, error) {
	return m.Writer.Write(b)
}

func MinifyFilter(res http.ResponseWriter, mimetype string) http.ResponseWriter {
	m := minify.New()
	// add other minfiers

	pr, pw := io.Pipe()
	go func(w io.Writer) {
		if err := m.Minify(mimetype, w, pr); err != nil {
			panic(err)
		}
	}(res)
	return MinifyResponseWriter{res, pw}
}
```

## License
Released under the [MIT license](LICENSE.md).

[1]: http://golang.org/ "Go Language"
