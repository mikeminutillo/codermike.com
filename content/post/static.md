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

Check in changes to the local repo with Github Desktop and then push my changes up to github. Look, [here it is](https://github.com/mikeminutillo/codermike.com/blob/eedd8322dffa52c3189165f4a81491729a3899ad/content/post/static.md#to-github)!

## Netlify

Signing up for Netlify couldn't be easier. All I had to do was log in with my GitHub credentials. I authorized them to get my email address.

Next up I have to pick a github repo. Because I signed up with GitHub credentials this is simple. I have to grant read access to my public repos (not a problem, everyone has that) and then I can pick from a list. I need to specify a branch and it even detects that I am using Hugo and adds the appropriate config!

Now all I have to do is "Deploy Site". Hit the button and wait a few seconds.

And it's done. Wait is that it? I click through and everything is working. Awesome!

Next step is to define a custom domain name. I do happen to already own codermike.com so let's follow the instructions to set up my domain name registrar properly.

OK. That was as easy as changing my A record in the DNS tables. And it's live. Unbelievable!

## Secure

One final thing that is awesome about Netlify. I can switch on https and enforce it for my whole site. I wasn't able to do that before because it was costly.

Netlify basically did it for me. All I had to say "enforce it". And here we are in https land. 


