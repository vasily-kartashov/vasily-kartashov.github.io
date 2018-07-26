---
layout: post
title: Simple twitter bot in Perl
tags: perl twitter
---

Let's write a simple twitter bot that publishes flash cards containing an Indonesian (because why not, they're neighbours) and an English word. And let's do that once an hour.

First thing first download the list of the most used words, for example from [101languages](http://www.101languages.net/indonesian/most-common-indonesian-words/).
The file should look like following

    itu     it
    akan    will
    dalam   in
    bahwa   that
    dengan  with
    anda    you
    ada     exist

Next register a twitter app, starting [here](https://apps.twitter.com/). The complete code of the bot is just few lines of Perl.

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

There are really few things that can beat Perl for this kind of hacking.
