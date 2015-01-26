---
layout: post
title:  "Setting up a new Mac"
---
This post outlines how to set up a new OS X Yosemite with sane default dotfiles and basic applications. Dotfiles are heavily influenced by [Mathias Bynes](https://github.com/mathiasbynens/dotfiles).

## Install brew

{% highlight bash %}
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
{% endhighlight %}

## Install brewdler

{% highlight bash %}
sudo gem install brewdler
{% endhighlight %}

## Clone dotfiles

{% highlight bash %}
git clone https://github.com/SirIle/dotfiles.git ~/.dotfiles
{% endhighlight %}

## Run brewdler

{% highlight bash %}
cd .dotfiles
brewdle install
{% endhighlight %}

## Run RCM

{% highlight bash %}
rcup rcrc
rcup -f
{% endhighlight %}

## Make GNU bash default shell

{% highlight bash %}
echo "/usr/local/bin/bash" | sudo tee -a /etc/shells
chsh -s /usr/local/bin/bash
{% endhighlight %}

## Install NPM as non-root

{% highlight bash %}
echo prefix=~/.node >> ~/.npmrc
curl -L https://www.npmjs.org/install.sh | sh
{% endhighlight %}

## Add a ~/.extra to contain non-github stuff

{% highlight bash %}
# Git credentials
# Not in the repository, to prevent people from accidentally committing under my name
GIT_AUTHOR_NAME="Ilkka Anttonen"
GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME"
git config --global user.name "$GIT_AUTHOR_NAME"
GIT_AUTHOR_EMAIL="ilkka.anttonen@accenture.com"
GIT_COMMITTER_EMAIL="$GIT_AUTHOR_EMAIL"
git config --global user.email "$GIT_AUTHOR_EMAIL"
{% endhighlight %}
