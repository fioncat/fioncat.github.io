[Fioncat Blog](https://fioncat.github.io)
================================

Install ruby and jekyll in ArchLinux:

```bash
sudo pacman -S ruby
gem install jekyll
```

Add this to profile:

```bash
export PATH=$PATH:$HOME/.local/share/gem/ruby/3.0.0/bin
```

Run server locally:

```bash
jekyll serve --watch
```
