---
title: "Static"
date: 2018-07-25T14:56:06+08:00
---

I used to blog and I've been meaning to get back to it. This tweet has finally prompted me to do something about it:

{{< tweet 1017967674263994373 >}}

<!--more-->

The author of the linked article described using [Hugo](https://gohugo.io) and [Netlify](https://www.netlify.com) to get everything up and running so I figured I'd give it a shot. 

It was pleasantly easy to get everything running.

## Making a site

Hugo is just an executable that I can add to my path. Once that's done I can spin up a new Hugo site with:

```
hugo new site <sitename>
```

This gives me the basic structure. From there I'm pretty much just editing markdown and toml files.

I had to pick a theme but there's loads. I probably spent the most time doing this.

Once everything is generated, I can run hugo as a server to check the content.

```
hugo server
```

This sits and watches my files for changes and updates in realtime too. Very handy for trying out different config settings and seeing what they do.

Now that I am generating a site I need to put it somewhere.

## To github

Github seems like the logical place to put it. I already have code on github and Netlify knows how to read from there.

The easiest way to do this step is to create a new repo using the github web UI, and then clone it to my local machine and copy the contents of my hugo website into it.

Git doesn't like tracking empty folders so I needed to drop some `gitkeep` files around the place to keep things in place. I'm lazy so I spent more time writing a LINQPad script to do this for me than it would have taken to just do it by hand. In case you're lazy, here is that script:

```cs
(from dir in Directory.EnumerateDirectories(@"C:\code\web\codermike.com", "*", SearchOption.AllDirectories)
 where dir.Contains(".git") == false
 where Directory.EnumerateFileSystemEntries(dir, "*.*", SearchOption.TopDirectoryOnly).Any() == false
 select Path.Combine(dir, ".gitkeep")
).ToList()
 .ForEach(x =>
{
	using (var fs = File.Create(x)) {}
});
```

Check in changes to the local repo with Github Desktop and then push my changes up to github. Look, here it is!

## Netlify



