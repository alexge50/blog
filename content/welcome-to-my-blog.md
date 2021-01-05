Title:  Welcome to my blog!
Date: 2021-01-05 05:37
Category: blog
Summary: This is a test article!

Welcome to [b.alexge50.com](https://b.alexge50.com), my blog - the blog where 
I will write about my latest findings about demonic incantations (aka `C++` articles), about my mainframe
or maybe some travel `/var/log`s, the bandwidth of my internet connection is the limit!
In the reminder of this article I will describe technical details behind how I run my blog and 
then some projects I am planning on pursuing in this shiny new year. 

# Technical details
On the journey to finding my most comfortable stack of technologies for releasing this blog, I had 2 findings:

* [pelican](https://blog.getpelican.com/) - a static site generator, written in Python. 
While [jekyll](https://jekyllrb.com/) is probably more popular, the downside - for me - is that it's written in Ruby, a language which I do not know. 
As such, pelican is a great choice for my comfort, since I am able to write custom themes, and maybe plugins in the future.

* [Gitlab CI](https://docs.gitlab.com/ee/ci/) - while not exactly new to me, initially I tried to use jenkins, 
however after a night full of testing jenkins out I conceded to some weird behaviour: when running a job, under docker, 
for some reason jenkins would apparently crush - without any error logs. 
I bit the bullet and I tried out Gitlab CI (self hosted, on my network) and the example worked well. 
While jenkins seems more powerful, the learning curve seems steeper as well, so I decided to stay with Gitlab CI.

The website is served on an nginx server and the CI is configured to build the project 
and then upload the artifact files via ssh to the nginx server. 
All this happens on vms and containers on my network.

# The projects
In fall of 2018, I started working on my biggest project - [GIE](https://github.com/alexge50/gie) - a node based image editor. 
Unfortunately, I stalled the development in the summer of 2019, 
with a fair amount of ideas on my TODO list (kanban list, if I dare) for the project. 
The first thing on this list is completely redoing the GUI, from scratch: 
I started working on a [node editor GUI](https://github.com/alexge50/NodeEditor), 
I will continue with doing basic widget inputs for the node editor, and the GUI will have overlays for various menus, 
that are going to be implemented in [imgui](https://github.com/ocornut/imgui).

In the past term, I polished a past project of mine, [a sphere music visualiser](https://www.youtube.com/watch?v=QCSV2f6JyZw),
and I worked on another music visualiser project - a project that synchronises an RGB light strip to music. 
One article will be dedicated to this project, in the near future (hopefully the first quarter), 
after the project is fully implemented and I am content with the results. 
Moreover, in this year I will be delving into more synthesizing related projects, 
which I will write about on this blog as well!

Some of these projects I am planning on implementing live, 
on stream (which can be found [here](https://www.twitch.tv/alexge50).

\- alexge50 :x