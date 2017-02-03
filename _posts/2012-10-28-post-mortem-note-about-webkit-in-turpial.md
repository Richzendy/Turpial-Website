---
layout: post
title: Why Webkit sucks? A post-mortem note about Webkit in Turpial
published: true
author: satanas
comments: true
date: 2012-10-28 12:10:30
tags:
    - development
    - GTK3
    - performance
    - Python
    - turpial
    - Turpial 2.0
    - webkit
categories:
    - desarrollo-2
    - noticias
    - oficial
permalink: /2012/10/post-mortem-note-about-webkit-in-turpial
---
Almost 8 months ago we offered a [brand new turpial version][1], with a lot of new features: Multiple accounts, unlimited columns and a really good looking interface. That looked promising, however it was an unkwon path for us. Very innovator but very uncertain. Guess what? We stucked up. Yes, Webkit let you do **anything** you want, imagination is the limit, but as the uncle Ben said: "With great power comes great responsabilities". And that quote never had such a big meaning for me as now. This apparently unlimited power requires a extremely solid base of development and in our case, we hadn't. 
  


## The unknown path Technically speaking, there are no tools or frameworks to do what we wanted the way we wanted, it was: integrate HTML and Javascript into Python (desktop) applications using just a Webkit container. Well, actually Pyjamas and PyQt does something like that, but the first it's traumatizing and with the second we should marry with Qt toolkit and this was not the idea. We wanted to deliver the same application with pretty much the same code in any graphic toolkit with webkit support: Gtk, Qt, WxWidget, et al and pure Python. That was our goal. 


  


## First approach First thing I had to deal with was Webkit and threads. Threads? Yes sir, threads. Threading is the only way you can do a long task without freezing the whole UI, it's not very pleasant to get a gray and useless window while your app is fetching your timelime, for example. At he beginning was very frustrating and I got "solutions" like: "You have to enclose all the methods that access the UI from the thread with this": 

`gtk.threads_enter()
my_method()
gtk.threads_leave()` Can you imagine doing so for every single method that should access to UI widgets? It's at least overwhelming. I continued my research and after like 2 weeks of frantic reading I finally got a reasonable explanation: Gtk is thread-aware not thread-safe, so you just need to start the threads support and enclose the main loop with those instructions and Gtk will know what to do. Ok the code was simple, at the beginning of your file do this: `gtk.threads_init()` And then enclose your main loop with this: `gtk.threads_enter()
gtk.main()
gtk.threads_leave()` That was one of happiest moments of my life, Webkit was running smoothly whilst the app was executing a lot of task on background. If you are curious about the threading implementation checkout the code of the worker in our [legacy branch][2]. Passed this obstacle I started to work in a [HTML Render Engine for Turpial][3], based on Python, Webkit and Jinja2. It worked like a charm, Jinja2 is a really amazing solution for template render but there was something else, Javascript integration. We needed to find a way to call Python functions from Javascript code. After a little research we did it. We intercepted the click event generated by Webkit and changed the protocol for the URL according our needs. Regular links started with the common **http://** and they trigger a new tab on the default browser whilst links to call Python functions started with **cmd://**. This did the trick. 
  


## Passing arguments Intercepting clicks wasn't the end, it was just the beginning. Despite the idea of calling Python methods from URLs in HTML seems motivating, we faced a new challenge: Passing arguments to those Python methods. I say a "challenge" because we only could build links, so arguments should be passed as a part of the link and then parsed out. After lot of testing I decided to implement this schema: 

`cmd://method_name-%&%-argument1-%&%-argument2...` Where **-%&%-** acts as arguments separator. "Why such a complicated characters sequence?" You may ask. Well, because I thought that making a hard sequence is less probable that parsing fails by misunderstanding the separator with the content of one argument. I tried with simpler separators and base64 encoding but we needed to pass other params (like status messages) that have a lot of non-ASCII chars and base64 is not very good for that. For those arguments I ended using URL encoding. At the moment of writing this I think that maybe a better solution could be making [RESTful][4] links, this means to use slashes (/) as separators and URL encoding each argument. Like this: cmd://method\_name/url\_encoded\_argument1/url\_encoded\_argument2/url\_encoded_argument3/... But, hey! it's done. Now I'm relaxed, working on GTK3 migration and probably my brain is not under stress. Sh*t happens. 
  


## Layout One of the most annoying things to deal with was the layout. HTML let you build anything, but if you've worked a little bit with HTML you should know how frustrating could be sometimes build a decent layout. Multiplies that per 100 and then you will have an idea of how frustrating becomes to do it for a desktop application. Just a bad css-margin or a wrong floating div is enough to send your layout to hell. Turpial is a desktop application and, as such, it should be flexible, resizable, movable and everything that ends with "able". I had to use a table for layout (yes, shame on me) because it was the only way I could guarantee that layout won't break my app into an unusable window just for 1px extra in a margin. Trust me, I tested A LOT and everything failed, so I used one table. Another need was to show modal dialogs, and this is kind of annoying with pure HTML and Javascript. You probably would be thinking "But dude, there are lots of Javascript libraries that even cook the breakfast for you" and you're right, but this webkit implementation had another limitation. With PyWebkitGtk you can render HTML by loading the URL or just passing a HTML string. Obviuosly, I used the second one for Turpial. The issue is that when you pass a HTML string, you must pass a relative path for the resources (images, CSS, etc) and none of the most used forms was valid. I tested with file:///my/path/, /my/path/, path/ and other possible values, even with and without trailing slashes and nothing worked. Resources weren't loaded correctly. One solution was to host those resource in the cloud but that would make Turpial relies in other services/platform and that wasn't an option for me. The best shot was to embed the whole code (HTML, Javascript and CSS) in a single file and pass it as string. Can you imagine now how hard it was to add lots of javascript libraries? And now try to imagine how hard could be debugging? It was more than a need to keep Turpial with the smallest amount of Javascript libraries. For me this meant one thing: No more libraries than JQuery. 


  


## Performance Keeping only one Javascript library for Turpial wasn't only matter of comfort, it was related with performance too. Rendering a HTML page with thousands and thousands lines of code increase the memory consumption of webkit and becomes Turpial into a fat and slow little bird. Speaking of performance, the minimum webkit consumption was 50MB (with just a Hello World page) plus the 40MB of GTK consumption, 90MB just to open Turpial. We were flushing our lightness through the toilet but maybe we could live with it, we just need to make it worth. We started to work in some javascript methods to optimize the status rendering, for example, we didn't render the whole column (of statuses), we just added new tweets and deleted the old ones keeping a fixed number of tweets on the column (no more than 200 per column). I used that technique in Gtk and it worked, even in Webkit seems to work, but if you used Turpial for a long time you could see how memory increased until cause segmentation fault. Yeap, your operating system had to act to avoid Turpial ate the whole memory and this was tested by Willicab who saw 1.8GB of RAM used by the bird. We had memory leaks. After digging a lot in the source code, testing and profiling memory I realized that webkit had a memory leak when you use the 

**execute_script** method to update the DOM of the page. Everytime you use that method your memory increases a little bit. Tweets were keeping in memory (in an array) and in HTML view ad infinitum, making the memory grows and grows until the OS decided to close the application. Turpial 1.6.x used like 50~60MB of RAM but Turpial 2 was using almost 100MB of RAM while applications like Banshee were using 60MB and gnome-shell 120MB. For my this was unacceptable for a "lightweight" application. This issue taugh me that PyWebkitGtk implementation was not so good as I expected, it lacks of documentation (the only available is the C++ documentation) and that if you want to develop complex applications on top of it you should think twice. 
  


## Conclusion It was a really good experience (in terms of learning) but all the issues explained above make me take the decision of drop the webkit support and back to the limited but stable-and-well-known GTK3 in order to deliver a new Turpial version as soon as possible. Turpial 1.6.9 has been out there for more than 1 year and it's time to release a fresh version of the bird. Webkit is a really powerful tool, I have no doubt about it. You can build anything on top of HTML and present the same look & feel for all platforms, besides using CSS you can implement a theme engine easily. However the complexity of the integration, the frustration you have to deal with everytime you want to make changes on your layout and it breaks and the performance issues put Webkit out of my list of tools for Turpial. Probably I could use it, but not for Turpial. For using Webkit as we want we need a framework, limited and bounded that minimize all the suffering and offer you a stable tool for development. That's all folks, I hope this article can be helpful for someone else. 


  


## Time of death: October 10, 2012



  
If you are interested on this topic you can read more here: 

  * [First Turpial 2.0 official announcement][1]
  * [First 2.0 alpha release][5]/
  * [Alpha release delayed][6]
  * [Mailing list thread about performance issues][7]
  * [Mailing list thread about Gtk3 migration][8]
  * [Official announcement of Gtk3 migration][9]
  * [Interesting article about Web interfaces with Python][10]
  * [GNOME explanation about Gtk threading awareness][11]

 [1]: http://turpial.org.ve/2012/02/turpial-2-0-whats-new/
 [2]: https://github.com/satanas/Turpial/blob/webkit-legacy/turpial/ui/gtk/worker.py
 [3]: http://wiki.turpial.org.ve/dev:htmlrendererengine
 [4]: http://en.wikipedia.org/wiki/Representational_state_transfer
 [5]: http://turpial.org.ve/2012/03/the-first-alpha-release-of-turpial-2-0-is-here
 [6]: http://turpial.org.ve/2012/03/alpha-release-delayed-for-2-0/
 [7]: https://groups.google.com/d/topic/turpial-dev/iz3AWrqzGhs/discussion
 [8]: https://groups.google.com/d/msg/turpial-dev/dl_IJO6oYWE/C68Ll_VPuV0J
 [9]: http://turpial.org.ve/2012/10/im-going-through-changes-said-turpial/
 [10]: http://www.aclevername.com/articles/python-webgui/
 [11]: http://developer.gnome.org/gtk-faq/stable/x481.html