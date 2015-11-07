---
layout: post
title: Simple twitter bot in Perl
---

I like learning foreign languages, and tought that it would be a good idea to write a simple twitter bot that would regularly post a foreign word with an English translation.
As much as I like the hot tech (not sure what's hot anymore though, is it still Rust or did we move on already?), there's nothing that can beat	the raw power of good old perl.

First thing first download the list of top words for your language of interest, for example from here: http://www.101languages.net/indonesian/most-common-indonesian-words/ (@todo add exact link)
The file should look like following

    itu     it
    akan    will
    dalam   in
    bahwa   that
    dengan  with
    anda    you
    ada     exist

Next register a twitter app (@TODO write how to do it with a couple of screenshots)

{% highlight perl %}
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
{% endhighlight %}

...