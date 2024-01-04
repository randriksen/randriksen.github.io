---
title: "The post i never wrote"
author: Ole
tags: Windows Microsoft
categories: Microsoft
author_profile: true
classes: wide
---

Today, I decided to reinstall an old laptop so that my kids would have something to play Minecraft on. During this process, I had to do something that reminded me of the time I almost created a blog a long time ago, because I couldn't find anything that described the entire process I needed to do back then. The interesting thing is that the method I found back then is still valid.

More than a decade ago, back when I was a young informatics student in Oslo, Windows Vista came out, and I decided I would use it on all my computers. As a student, I had access to student licenses for most of the Microsoft products, so getting a valid installation wasn't a problem. The problem was a much more technical one. I had already moved to a laptop with no optical drive, so I needed to get Windows installed via USB. Now, this was actually before this was a common practice, and there weren't a lot of descriptions about how to get this working, so I had to figure it out for myself.

Getting the data from the ISO onto the USB stick wasn't a problem. That was a straightforward copy job. The problem came when the computer found the USB to start from but wouldn't find anything on it to boot from. That was a strange problem at the time for me. So, I Googled everything about it that I could find, and I remember I found a blog post that explained about half the process. The USB didn't have the right type of partition on it. I had to repartition the USB key to give it a primary partition and to make it active. This was all rather simple:

```cmd
diskpart
list disk
select disk 1
list partition
select partition 1
delete partition
create partition primary
select partition 1
active
exit
```

This all assumes that you only have one partition on your USB key.

But it still wouldn't boot into the Windows installer.

So, I did what I do when I get stuck; I start looking deeper into things. On the Windows ISO, there was a folder called boot, so I started there. In that folder, there was an EXE file called bootsect.exe, which turned out to be the tool I needed to get things working. The problem was that the USB drive had the wrong boot sector type. And between Windows XP and Windows Vista, as far as I remember, they changed the type of boot sector Windows uses.

```cmd
bootsect /nt60 x:
```

This was the command that changed everything and made me able to install Windows Vista on all my computers.

At the time, this saved me a whole lot of work, because I had somehow managed to crash the old XP installation on my main computer, so getting it back up and running was really important, since I needed it for classes. So my nerves calmed down, and I was able to sleep that night.

As mentioned, I considered making a blog just to post this solution because at the time I didn't find anyone else who had written about it, but I never did. But I'm trying to learn from my past mistakes, so I'm writing about it today.

These days, there are tools that make it a lot easier to fix USB installation media. For Windows 11, you can download the [official tool](https://www.microsoft.com/software-download/windows11) from Microsoft, or you can use [Rufus](https://rufus.ie/en/) for almost any operating system you can find.

But you can still use the manual method I found back in the days when Vista was something new and shiny.