In this episode we'll introduce you to how we upload content to Codemy.net

We're doing all this using a gem called [Writefully](http://writefully.codemy.net) created in house right here at Codemy to help stream line the process of updating content on our site.

And today we're open-sourcing it to the world.

## Why Writefully?

Because everything else was either too slow, buggy, or not flexible enough. We've  used all kinds of CMS in the past and all of them have a few big flaws. 

Its difficult to get content to look the way we want with WYSIWYG editor. Uploading images can also be slow, and its difficult to customize the business logic that governs the site. Usually its done through theme customizations and plugins for the CMS, which may or may not be good depending on how you look at it.

Starting a new rails app may be easy but we needed a nice interface for writing content. We didn't want to waste time building another editor, a nice interface for handling image uploading, handling contributor access control, collaboration tools and the likes.

So we thought hard, and felt that we were already using the best tools. We're software writers (thx DHH), we create code using our text editor, so we figured we can just write our content using our favorite text editor. We wanted to adopt the idea "Bring Your Own Editor" or "BYOE".

With writefully the content broadcaster (our website) is completely de-coupled from the content creation tool. What this means is for the non tech-savvy we can create beautiful native experiences that works offline on their computer, and when they get back online and want to publish all they have to do is hit one button and their content is live on the site. No need to copy / paste from MS Word. Thats just one example, if you want to use a cloud based tool like [prose.io](http://prose.io) you can do that too!

## How does it work?

Writefully turns your github repository into a content management system. What that means is we don't have to worry about creating contributor access control layer, or tools that allow content creators to collaborate better. All we have to do is use github to control who can contribute to the content. Github also has a great collaboration tool. So we got all these features for free.

![Writefully Diagram](assets/wf-diagram.png)

You can see the repository for the content on Codemy.net [right here](https://github.com/zacksiri/codemy-net)

## Not everyone knows git / github

Yes we know that. Content creators for your blog may not know what git or github is. We initially made this tool our ourselves, however we realize the potential this gem has. 

This could potentially be the start of a new eco-system. Imagine services like github but geared towards content creators who are not developers (strenght of de-coupling). Imagine having a variety of editors you can choose from, de-coupled yet beautifully integrated to the content broadcaster through the intermediate service (like github). Imagine having different themes that can help you change the way your site looks. You can let your theme live in github while your content directory live in a writer friendly service.

## Flexibility

You have the choice between the flexibility of building a custom rails app using writefully or using a pre-built CMS version (not available yet) that supports multi-sites and themes.

## Final Thoughts

Writefully has allowed us to: 

+ Use markdown to write our content
+ Work with images locally in our content
+ Use github's feature without having to build it ourselves.
+ Build our content locally and let writefully worry about updating our site.

Writefully is currently in heavy development, we're definitely looking for people to help collaborate with us on this project. So if your ruby / rails / gems savvy don't be afraid to submit a pull-request!

We have open-sourced writefully to the public, which means you can install it on your own computer and start building rails apps with it right away. We're working on documentation and videos taht will help you get started developing and building amazing sites with writefully. Check out our [Getting Started Guide](https://github.com/codemy/writefully#getting-started).



