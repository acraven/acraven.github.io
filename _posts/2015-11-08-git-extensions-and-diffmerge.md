---
layout: post
title: "Installing and configuring Git Extensions with DiffMerge"
date: 2015-11-08
categories: git extensions diffmerge
featured_image: /images/cover.jpg
---

Install Git, Git Extensions and DiffMerge using Chocolately.

{% highlight batch %}
choco install git
choco install gitextensions
choco install diffmerge
{% endhighlight %}

Add the following to C:\Users\{UserName}\.gitconfig

{% highlight batch %}
[mergetool "DiffMerge"]
   cmd = \"C:/Program Files/SourceGear/Common/DiffMerge/sgdm.exe\" --title2=\"Base/Result\" --merge --result=\"$MERGED\" \"$REMOTE\" \"$BASE\" \"$LOCAL\"
[merge]
   tool = DiffMerge
[difftool "DiffMerge"]
   cmd = \"C:/Program Files/SourceGear/Common/DiffMerge/sgdm.exe\" --caption=\"$REMOTE\" --title1=\"Remote ($REMOTE)\" \"$LOCAL\" \"$REMOTE\"
[diff]
   guitool = DiffMerge
{% endhighlight %}
