+++
date = 2022-07-29
title = "TOML Syntax Highlighting for Zola Front Matter in Vim"
taxonomies.tags = ["static-site-generation", "vim", "webdev", "zola"]
+++

If you use [Zola](https://www.getzola.org) and use Vim to edit Markdown files in your `content` directory, you may notice that the TOML front matter meta-data at the top of the file isn't syntax highlighted.
If you already have [TOML syntax highlighting](https://github.com/toml-lang/toml/wiki#editor-support) set up for Vim, you can fix that with the following configuration:

```vim
unlet b:current_syntax
syntax include @Toml syntax/toml.vim
syntax region tomlFrontMatter start=/\%^+++$/ end=/^+++$/ contains=@Toml
```

Put this in `~/.vim/after/syntax/markdown.vim` and the TOML front matter in your Markdown files should now be syntax highlighted.

The `unlet` line is needed for the file inclusion on the next line to work properly.

The `syntax include` line creates a *cluster* named `@Toml` that contains every syntax rule in all paths ending with `syntax/toml.vim` that Vim can find in its runtime path list.

The `syntax region` line denotes a Vim syntax region that we've chosen to name `tomlFrontMatter`.
The region starts if the very first line of the file matches `+++`, and continues until a `+++` line is found.
The whole region is marked as containing the `@Toml` cluster from the previous line.

As a bonus, you can accomplish the same thing for YAML front matter in Jekyll with a similar configuration:

```vim
unlet b:current_syntax
syntax include @Yaml syntax/yaml.vim
syntax region yamlFrontMatter start=/\%^---$/ end=/^---$/ contains=@Yaml
```

Both of these configurations can co-exist in the same file too, applying either TOML or YAML syntax highlighting depending on whether `+++` or `---` is present at the top of the file.
