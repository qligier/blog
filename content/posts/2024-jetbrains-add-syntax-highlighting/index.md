---
title: "Implement syntax highlighting of a new language in JetBrains IDEs"
date: 2024-03-18
draft: false
tags: ['Development', 'FHIR', 'IDE']
description: "The blog post is exploring different ways to implement syntax highlighting for a language in JetBrains IDEs."
---

I regularly spend time working on FHIR ImplementationGuides, and I use the
[Visual Studio Code](https://code.visualstudio.com/) IDE to develop them. It is an IDE I used to
work with, but as of lately, my preference goes to the [Jetbrains](https://www.jetbrains.com/) IDEs.

The FHIR ImplementationGuides are written in a mix of Markdown, HTML, XML, JSON, INI and a specific language called
[FHIR Shorthand](https://build.fhir.org/ig/HL7/fhir-shorthand/index.html) (FSH). While the former
languages are properly supported in many IDEs, the latest is only supported in Visual Studio Code through a
[dedicated extension](https://marketplace.visualstudio.com/items?itemName=MITRE-Health.vscode-language-fsh).

In this article, I will investigate the possibilities to add syntax highlighting for FHIR Shorthand in the
JetBrains IDEs.

## Existing resources

We can investigate how the Visual Studio Code extension implemented the syntax highlighting for FHIR Shorthand. Its
sources are available in a
[GitHub repository](https://github.com/standardhealth/vscode-language-fsh), and we can quickly find
a promising _syntaxes_ folder that contains a single file: _fsh.tmLanguage.json_. This file contains
the language grammar in a [TextMate](https://macromates.com/manual/en/language_grammars) definition
in JSON format, mostly made of keywords and regular expressions (RegExp). That's a good resource in our quest, but
may prove itself limited because not all languages can be properly parsed by RegExps (only Type-3 in the
[Chomsky hierarchy](https://en.wikipedia.org/wiki/Chomsky_hierarchy)).

FSH files are transpiled to FHIR resources by [SUSHI](https://github.com/FHIR/sushi), a Node.js
application. The application uses a parser to read the FSH files, which may be useful to us. The sources are also
available on GitHub, and after a quick search, we can find two useful files: _FSH.g4_ and
_FSHLexer.g4_. Those are ANTLR4 grammar files, and they contain the grammar to
[lex and parse](https://www.cs.umd.edu/class/spring2017/cmsc430/slides/03-lexing-parsing.pdf) the
FSH language. This is a very powerful tools, but it may be overkill for syntax highlighting only.

## Doing it the official way

JetBrains documentation is quite extensive, and after a quick search, we can find a whole documentation section
dedicated to
[Custom Language Support](https://plugins.jetbrains.com/docs/intellij/custom-language-support.html),
including information about syntax highlighting. It indicates that it requires the creation of a plugin, the
definition of a lexing grammar with JFlex, and the definition of a parsing grammar in an extended NBF syntax
(Grammar-Kit).

While JFlex and the Grammar-Kit BNF syntax are not far from the ANTLR4 syntax, it can become quite complex to
convert the existing grammar to the JetBrains format because they do not share the same features. I once did such an
attempt, and it revealed itself quite inconclusive (
[lexer](https://github.com/qligier/jetbrains-plugin-fhir/blob/dev-2023/src/main/java/ch/qligier/jetbrains/plugin/fhir/fsh/lexer/Fsh.flex),
[parser](https://github.com/qligier/jetbrains-plugin-fhir/blob/dev-2023/src/main/java/ch/qligier/jetbrains/plugin/fhir/fsh/parser/Fsh.bnf)).

## Doing it with the ANTLR grammar

The documentation contains a single sentence about using an ANTLR grammar for the parsing, referring to a project
led by ANTLR maintainers: the
[antlr4-intellij-adaptor library](https://github.com/antlr/antlr4-intellij-adaptor). While the repository
is still maintained, there was no new release since 2019, and the documentation is quite scarce. It is not clear if
this parser can properly substitute the Grammar-Kit parser, and if it can be used for syntax highlighting.

The ANTLR4 grammar would also need to be modified for a proper parser generation, because the lexer currently discards
some tokens that are not needed for the parser (the comments and whitespaces), whereas JetBrains IDEs require a continuous
token stream. While it should be a minor modification (replacing `-> skip` by `-> channel(HIDDEN)`),
the advantage to not maintain a separate grammar than the official project is lost.

## Doing it with the TextMate grammar

JetBrains now supports TextMate grammars through the bundled
[TextMate bundles](https://plugins.jetbrains.com/plugin/7221-textmate-bundles) plugin. It does not
require the creation of a plugin, but can be configured in the IDE settings. The immediate issue is that we only
have a `.tmLanguage` file, but a `.tmBundle` is needed. There are a
[few leads](https://stackoverflow.com/questions/29229511/how-do-i-install-the-es6-tmlanguage-into-textmate-2)
to convert the former to the latter, but nothing too simple.

## Doing it with an LSP integration

The last possibility I found to implement syntax highlighting for a custom language in JetBrains IDEs is to use the
[Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) with a
application that implements the protocol for the custom language. I'm not aware of any existing LSP server for FSH,
but the current SUSHI application could be a good candidate for developing such a server. The LSP offers a lot of
features, but only a
[small subset is currently supported](https://plugins.jetbrains.com/docs/intellij/language-server-protocol.html).
It is nonetheless the most future-proof and helpful solution, as the LSP standard is widely adopted and in active
development. Other IDEs (including Visual Studio Code) would profit from the LSP server, implementing the FSH logic
once and providing its results to all the IDEs.

## Personal decision

While I might try to implement the TextMate grammar in the JetBrains IDEs without creating a plugin to profit faster
from syntax highlighting, I will also investigate the ANTLR4-based parser. A proper parser would allow implementing
a lot of other features, such as code completion, code navigation, and refactoring thanks to the impressive APIs
accessible to the plugin developers.

The LSP solution is also very appealing, but it requires a lot of work and most probably a lot of coordination with
the SUSHI developers. It is a long-term solution, and I will keep it in mind for the future.

You can check the progress of the implementation in the
[GitHub repository](https://github.com/qligier/jetbrains-plugin-fhir).
