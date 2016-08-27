---
layout: post
title:  "Setting up a new Mac"
---

This post outlines how to set up a new OS X Yosemite with sane default dotfiles
and basic applications. Dotfiles are heavily influenced by [Mathias
Bynes](https://github.com/mathiasbynens/dotfiles).

## Install brew

~~~bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
~~~

## Install brewdler

Updated on 9.3.2015

~~~bash
brew tap Homebrew/brewdler
~~~

## Clone dotfiles

~~~bash
git clone https://github.com/SirIle/dotfiles.git ~/.dotfiles
~~~

## Run brewdler

~~~bash
cd .dotfiles
brew brewdle
~~~

## Run RCM

~~~bash
rcup rcrc
rcup -f
~~~

## Make GNU bash default shell

~~~bash
echo "/usr/local/bin/bash" | sudo tee -a /etc/shells
chsh -s /usr/local/bin/bash
~~~

## Install NPM as non-root

~~~bash
echo prefix=~/.node >> ~/.npmrc
curl -L https://www.npmjs.org/install.sh | sh
~~~

## Add a ~/.extra to contain non-github stuff

~~~bash
# Git credentials
# Not in the repository, to prevent people from accidentally committing under my name
GIT_AUTHOR_NAME="Ilkka Anttonen"
GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME"
git config --global user.name "$GIT_AUTHOR_NAME"
GIT_AUTHOR_EMAIL="ilkka.anttonen@accenture.com"
GIT_COMMITTER_EMAIL="$GIT_AUTHOR_EMAIL"
git config --global user.email "$GIT_AUTHOR_EMAIL"
git config --global credential.helper osxkeychain
~~~
