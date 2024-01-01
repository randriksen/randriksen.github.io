---
title: "The post i never wrote"
author: Ole
tags: Windows Microsoft
categories: Microsoft
author_profile: true
classes: wide
---

Today, I decided to reinstall an old laptop so that my kids could have something to play Minecraft on. During this process, I remembered the time when I almost created a blog a long time ago. This was because I couldn't find any resources describing the entire process I needed back then, and it's still valid.

More than a decade ago, as a young informatics student in Oslo when Windows Vista was released, I decided to use it on all my computers. As a student, I had access to student licenses for most Microsoft products, so obtaining a valid installation wasn't an issue. The challenge was more technical. I had moved to a laptop without an optical drive, so I needed to install Windows via USB. This was before USB installations were common, and there were few guides available, so I had to figure it out myself.

Transferring the data from the ISO to the USB stick was straightforward. However, the problem arose when the computer recognized the USB as a startup device but couldn't find anything to boot from. This was a strange issue for me at the time. I searched extensively and found a blog post that explained half the process. The USB didn't have the correct partition type. I had to repartition the USB key to create a primary partition and make it active. The process was quite simple:

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

This assumes you only have one partition on your USB key. However, it still wouldn't boot into the Windows installer. 

When I'm stuck, I delve deeper into the problem. On the Windows ISO, there was a 'boot' folder, which I explored. Inside, I found an EXE file named bootsect.exe, the tool I needed. The issue was that the USB drive had the wrong boot sector type. From what I recall, the boot sector type used by Windows changed between Windows XP and Windows Vista.

```cmd
bootsect /nt60 x:
```

This command was pivotal in enabling me to install Windows Vista on all my computers.

At the time, this saved me a lot of work, as I had managed to crash the old XP installation on my main computer. Getting it back up and running was crucial for my classes. I could finally relax and sleep that night.

I considered creating a blog to share this solution, as I hadn't found anyone else writing about it, but I never did. However, I'm trying to learn from my past mistakes, so I'm sharing it today.

Nowadays, tools like the [official tool](https://www.microsoft.com/software-download/windows11) from Microsoft for Windows 11 or [Rufus](https://rufus.ie/en/) for almost any operating system make fixing USB installation media much easier.

Still, you can use the manual method I discovered in the days when Vista was new and shiny.