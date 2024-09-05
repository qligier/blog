---
title: "How to implement highlighted inline code blocks in Hugo"
date: 2024-09-05
draft: false
tags: ['Development', 'Hugo']
description: "The blog post describes how I implemented the syntax used by InlineHilite for highlighted inline code 
blocks in Hugo."
---

I have written multiple documentation websites using [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/),
and got used to the highlighted inline code blocks.
It allows adding code snippets inlined with the text, while keeping the syntax highlighted (for known languages).
The syntax chosen by the plugin that introduced this feature,
[InlineHilite](https://facelessuser.github.io/pymdown-extensions/extensions/inlinehilite/), uses a shebang to specify
the language: <code>#​!python print("Hello, World!")</code> (surrounded with backticks) would be rendered as 
`#!python print("Hello, World!")`.

This blog is powered by Hugo, a static site generator, and InlineHilite is not available for it.
I wanted to keep the same syntax for the code snippets, as it makes writing and reading the source files more 
pleasant for me.
I searched for a similar feature in Hugo, but did not find any good solution to this question.
After digging into the documentation, and testing multiple ideas, I found a way to implement the shebang syntax for 
my blog.

The first step is to create a shortcode in Hugo.
There are a lot of resources on the internet that explain how to create a shortcode, so the implementation will be fast.
I checked the [Hugo implementation of the `highlight` shortcode](https://github.com/gohugoio/hugo/blob/master/tpl/tplimpl/embedded/templates/shortcodes/highlight.html),
and I used it as a base for my shortcode, but there are many different implementations of this shortcode in the wild.

It is a simple shortcode that takes a language as a parameter, and the code to highlight:
```html {title="/layouts/shortcodes/code_inline.html"}
{{ $lang := .Get 0 }}
{{ $source := .InnerDeindent }}
<span class="chroma">{{ transform.Highlight $source $lang "hl_inline=true,linenos=false" }}</span>
```

I called it _code_inline_, but it is not important, as we won't call it directly.
The shortcode is used in the markdown files as follows:
```markdown
{​{< code_inline "python" >}​}print("Hello, World!"){​{< /code_inline >}​}
```

It is a good start, but still way too verbose compared to the syntax used by the _InlineHilite_ plugin.
The next step is to create a custom markdown renderer that will transform the syntax used by the plugin into the shortcode.
The documentation did not reveal any way to create a custom markdown renderer, but I found a way to do it by looking at
the list of methods.
The one that got my attention is [RenderString](https://gohugo.io/methods/page/renderstring/): it takes a 
markdown string as input, and returns the rendered HTML.
It also supports rendering the shortcodes.

There are no equivalent for other formats supported by Hugo (HTML, AsciiDoc, etc.), so the solution only works for markdown.
That is not a problem for me, as I like writing in markdown.

The trick to expend the shebang syntax into the shortcode call is to use a regular expression on the markdown string, 
before passing it to the `RenderString` method.
This can be done by overriding the _single.html_ layout file of the Hugo theme.
The layout file often looks like the following:

```html {title="/layouts/_default/single.html"}
{{ define "main" }}
  <h1>{{ .Title }}</h1>
  
  {{ $dateMachine := .Date | time.Format "2006-01-02T15:04:05-07:00" }}
  {{ $dateHuman := .Date | time.Format ":date_long" }}
  <time datetime="{{ $dateMachine }}">{{ $dateHuman }}</time>
  
  {{ .Content }}
  {{ partial "terms.html" (dict "taxonomy" "tags" "page" .) }}
{{ end }}
```

The [.Content](https://gohugo.io/methods/page/content/) variable contains the rendered HTML of the original file.
The original content can still be accessed using the [.RawContent](https://gohugo.io/methods/page/rawcontent/) variable;
we can use it to modify the content as we want, before rendering it again.
This can be done in the following way (removing parts that are not relevant to the example):

```html {title="/layouts/_default/single.html"}
{{ define "main" }}
  {​{ $content := .RawContent }​}
  {​{ $content = replaceRE "`#!([a-z]+) ([^`]+)`" "{​{< code_inline $1 >}​}$2{​{< /code_inline >}​}" $content }​}
  {​{ $content | .RenderString }​}
{{ end }}
```

The [replaceRE](https://gohugo.io/functions/strings/replacere/) function takes a regular expression, a replacement 
expression and a string as input.
It searches for inline code blocks starting with a shebang, and replaces them with the shortcode call.
The modified content is then passed to the `RenderString` method, which renders the markdown content, including the
shortcodes.

I like this solution, as it allows me to keep a cleaner, more readable syntax in the source files, while being quite 
simple to implement.
It has some limitations too: the content won't be properly rendered in other places (like the RSS feed), but I don't 
consider this too annoying.

You can check my implementation in the source code of this blog: 
[the shortcode definition](https://github.com/qligier/blog/blob/master/layouts/shortcodes/code_inline.html), and the 
[_single_ layout file](https://github.com/qligier/blog/blob/master/themes/qblog/layouts/_default/single.html).
