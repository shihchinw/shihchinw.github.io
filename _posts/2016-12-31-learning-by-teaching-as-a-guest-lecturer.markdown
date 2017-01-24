---
title: Learning by Teaching, as a Guest Lecturer
layout: post
categories: []
tags: [Study]
published: True
---

I was privileged to be a guest lecturer to give five lectures about Computer Graphics at National Cheng Kung University (NCKU) in Taiwan. When I prepared
the course materials, I deeply realized again that the best way to learn a topic is to teach someone else (especially to a person who never heard about Computer Graphics before). In order to avoid teaching wrong ideas, I had carefully reviewed what I known from ground up and derived each mathematical equation in the textbooks. This post is about what I've learned in those few months.

# Blind Spots

In my re-studying process, I found I had several blind spots in my knowledge base. There were few assumptions I took for granted that I didn't deeply think over it before:

1. Is quaternion (or unit quaternion) a representation for rotation?
2. Why is there a magic 1/2 scale when we use quaternion for rotation?
3. What is the singularity of rotation?
4. What are the problems of rotation interpolation in matrix form?
5. Why is radiance so important in light transport process?

To be honest, I still can't answer all the questions in very detailed. Since my time is limited while the knowledge is boundless. Take rotation for example, after I saw a [website](http://rotations.berkeley.edu/) about rotations from Berkeley, I just realized I didn't know much about rotation at all. If we want to **truly** understand every details about it, we might need to study <span class="green">*group theory*</span>. As for Global Illumination, even though I had implemented a simple ray tracer for my thesis, I still find there are always new things I didn't know very well in the past and feel extremely excited to study new things to fill my knowledge gap.

# Coherent Explanations & Topic Selection

Making slides is a great way to build a coherent explanation for a topic. It helps me organize the order of each subject properly. The other important thing is how to combine several ideas into a theme. In my personal experience, a lecture without a theme is just like sailing in the ocean without a compass. It makes me think what are the most important ideas about Physically-Based Rendering (PBR) and Global Illumination (GI), and how to introduce those concepts gradually to make students understand the whole concept without being lost in the wild.

About the homeworks, we didn't ask students to implement a software rasterizer, which is the the conventional arrangement of many Computer Graphics courses in Taiwan. It is because I think we have few chances to implement the rasterization algorithm at work (but I still think it is a great exercise to learn how to become a graphics programmer).

Most of the time we are trying to make something like skin, hair or cloth look more photo-realistic or __physically plausible__. That's why I decided to emphasize on the reflection model and the implementation challenges in production environment. I hope I could bring a little different course materials to shrink the gap between school and industry.

For PBR & GI, I chose to introduce following subjects:

* Bidirectional Reflectional Distribution Function (BRDF)
* Microfacet BRDF model
* Disney artist friendly BRDF
* Ray tracing frameworks for Global Illumination
* Difficult lighting conditions and challenges for production renderer

# Research Papers & Software Developments

As a guest lecturer with production experiences, I think the most valuable things I could provide is the connections between research papers and production software developments. In spite of lots mathematical equations or theories, I showed several demo reels of cutting-edge CG software and explained the difficulties behind the implementations. One of the most interesting things came from our dialog in class:

>Me: Why is this software or demo awesome?  
>Students: ... (silence)  
>Me: Please raise your hand if you can't find any awesomeness at all.  
>(more than half of class students raised their hands)  
>Me: Please let me explain why it is awesome. The devil is in the details!!  
>(after few minutes...)  
>Students: WOW~ WOW~!!

There is a gap from research papers to production tools. To implement a production software we need to take more care of a lot things such as parallelism, memory footprint, maintenance, etc. Additionally, since most of our software are used by artists, we definitely need to pay attention to their requirements. _Soft skills are really matters_. Before diving into actual development, we have to make sure we truly understand what artists want. 

As a software engineer in an animation studio, any production problem can be solved either by artists or programmers. Don't obsess over using fancy programming languages or technologies. Sometimes, artists' solutions are more effective than our own. We should <span class="blue">_open up our mind to any possibility, put ourselves in others' shoes_</span>. Respect each others, work together closely to solve the problem at hand in an efficient and creative way.

# Inspired by Others

I enjoy talking to passionate people from different backgrounds and ages. They always show me a new way to examine a problem. As for the students who studies Computer Graphics for the first time, their questions can help me find out the blind spots of my understanding. This make me re-think the problem in a direction forward fundamental ideas. It opens up my mind not only to discover the relation between different topics but also to digest more core ideas behind certain findings.

Frankly speaking, the one who learns the most valuable things from this teaching experience is myself. In my lectures, I often said I still have lots of things I don't know very well. If they have figured out something new afterwards, please don't hesitate to share with we. Just like I said to the students in my first lecture: I'm not a teacher. <span class="blue">_I just a tour guide who wants to share the beauty of Computer Graphics I've seen in my journey. Please join me and have FUN!_</span>

# Footnotes

Here are my slides for the course, hope it would help someone who wants to learn about Computer Graphics. Enjoy!

1. [Computer Animation](/slides/2016_CG@NCKU/2016.05.10-Computer_Animation.pdf)
2. [Physically-Based Rendering](/slides/2016_CG@NCKU/2016.05.24-Physical-Based_Rendering.pdf)
3. [Global Illumination Part 1](/slides/2016_CG@NCKU/2016.05.31-Global_Illumination_PartI.pdf)
4. [Global Illumination Part 2](/slides/2016_CG@NCKU/2016.06.07-Global_Illumination_PartII.pdf)
5. [Self-Study Tips](/slides/2016_CG@NCKU/2016.05.10-Self-Study_Tips.pdf)
