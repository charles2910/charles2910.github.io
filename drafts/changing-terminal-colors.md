---
layout: post
title: Changing Terminal Colors
tags: [misc, debian]
author: Carlos Henrique Lima Melara - Charles
---

As soon as Gnome has made the dark theme available I changed to it - after a
quick research: it was released in Gnome 42 on March 2022. Back then I was at
university and spent most of the time in classroom with a pencil and a notebook
(a paper one :-) and only a few ours on my laptop, so dark theme was very nice.

Let's jump to present day where I work full time with my computer and a lot of
my free time goes to Debian. After I leave my $dayjob, my eyes were pretty much
done, they were dry (working in AC all day doesn't help either), it was
difficulty to focus on text and i was very tired in general. I remember reading
somewhere reading text on a dark background was more demanding to our eyes so I
decided to give a try to a light theme (at least on terminal where I spend most
of my day). I have to say things improved quite a lot and I decided to stick to
it.

But there was a problem still, in weechat and vim there were some colors that
were invisible against the light background (see the picture below for an
example). The worst thing is all pre-defined color schemes had this problem and
I was almost giving up. Then I decided to search the web and JACKPOT! Found
some custom color schemes for terminals, more specifically
[Gogh](http://gogh-co.github.io/Gogh/).

I really enjoyed Catppucin Latte color scheme, but the install instructions
were `bash -c  "$(wget -qO- https://git.io/vQgMr)"` and there is no way on
earth I'm bindly executing a script from the internet on my laptop - well, it
least wasn't `sudo bash -c ...` - but it was a no-go nonetheless. Suddenly I
realized that I didn't need to install it, the hexadecimal color codes were
written on the page showing the color schemes available. All I had to do is
copy-and-pasta - yeah, my boss is Italian and now I can't help myself.

It all worked beatifully and I have a great color scheme! Yeah, I wish, the
background color of neomut became a terrible gray that made everything terrible
to read. The solution was to grab the color from the Gnome color scheme for
that specific one.
