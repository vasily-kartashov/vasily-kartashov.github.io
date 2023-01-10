---
layout: post
title: Simple twitter bot in Perl
tags: perl twitter
---

are you looking for a fun project to try out? How about creating a twitter bot that publishes flashcards with Indonesian and English words? And let's make it even more fun by setting it to run once an hour.

First things first, let's gather a list of the most commonly used words in Indonesian, you can find it on sites like [101languages](http://www.101languages.net/indonesian/most-common-indonesian-words/). Once you have the list, it should look something like this:

    itu     it
    akan    will
    dalam   in
    bahwa   that
    dengan  with
    anda    you
    ada     exist

Next step is to register for a twitter app, you can do that by heading over to their [website](https://apps.twitter.com/). The complete code for the bot is just a few lines of Perl. Trust me, there's nothing better than Perl for this kind of quick and easy hacking.

The code looks something like this:

```perl
#!/usr/bin/env perl

use strict;
use warnings;
use Net::Twitter;
use Scalar::Util 'blessed';

srand;
rand($.) < 1 && ($message = $_) while <>;

$message =~ s/\s+/: /;

my $nt = Net::Twitter->new(legacy => 0);
my $nt = Net::Twitter->new(
  traits              => [qw/API::RESTv1_1/],
  consumer_key        => 'VH...',
  consumer_secret     => 'ir...',
  access_token        => '38...',
  access_token_secret => 'Mu...',
);

my $result = $nt->update($message . ' #Indonesian');
```

This is a great project to learn more about programming, Perl and twitter bots. Plus, you'll be learning some new words in Indonesian!
