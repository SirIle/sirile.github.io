---
layout: post
title:  "New Mac with High Sierra"
commentIssueId: 18
---

This post outlines how to set up a new MacOS High Sierra with sane default dotfiles
and basic applications. Dotfiles are heavily influenced by [Mathias
Bynes](https://github.com/mathiasbynens/dotfiles). My own dotfiles can
be found [here](https://github.com/SirIle/dotfiles.git).

## Install brew

[Homebrew](https://brew.sh/) is a package manager which greatly simplifies
the set up and maintenance of packages.

~~~bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
~~~

## Clone the dotfiles

~~~bash
git clone https://github.com/SirIle/dotfiles.git ~/.dotfiles
~~~

## Install the default packages with a bundle

~~~bash
cd .dotfiles
brew bundle
~~~

## Run RCM

RCM links the files in the .dotfiles folder to the user's home directory. 
This way they can easily be maintained with git in their own folder without
the home folder becoming too complex with .gitignore.

~~~bash
rcup rcrc
rcup -f
~~~

## Make GNU bash default shell

~~~bash
echo "/usr/local/bin/bash" | sudo tee -a /etc/shells
chsh -s /usr/local/bin/bash
~~~

## Install nvm

Node Version Manager doesn't support installation through Brew, so it's better
to install it from the command line.

~~~bash
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.9/install.sh | bash
~~~

For me it gave an error before I created the _~/.nvm_ directory first.

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
