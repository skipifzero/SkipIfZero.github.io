---
layout: post
title: "The Orange Cube"
categories: games
comments: true
---

# The Orange Cube

Earlier this summer I finally finished my education and got my Master of Science in Computer Science degree from Chalmers University of Technology. Leading up to this moment I had spent the last few months working on my Master's Thesis ([GPU-Accelerated Real-Time Stereo Matching](http://studentarbeten.chalmers.se/publication/249761-gpu-accelerated-real-time-stereo-matching)), and it left me a bit burnt out on programming in general. So I decided to take a break from programming for a few months, in order to feel ready for it again once I started my new job.

During this period I looked back on one of my favorite video game consoles from my childhood, the Nintendo GameCube, and I realized a few interesting modifications has been developed for it in recent years. So I decided to start a short project to create the ultimate GameCube. This blog post details my ups and downs during this journey.

## The Nintendo GameCube

My love for the GameCube might simply be nostalgia as it was my first non-handheld dedicated video game console. However, I still play a lot of the games to this day, and I discover new gems every now and then. The picture below shows some of the best parts of my collection, I can vouch that every single game is still worth playing to this day.  I would even argue that some of them, such as Super Smash Bros Melee, F-Zero GX, and Metroid Prime, still have not been surpassed in their respective genres. 

![](/assets/posts/2017-08-08-the-orange-cube/games.jpg)

In order to play GameCube games today you essentially have two options. Either you play the games on an actual GameCube/Wii, or you play them on the excellent emulator [Dolphin](https://dolphin-emu.org/). In most cases Dolphin will be the best choice as it allows to render games a higher resolutions. There are, however, some cases where actual hardware is a better choice.

Games such as [Zelda Four Swords Adventures](https://en.wikipedia.org/wiki/The_Legend_of_Zelda:_Four_Swords_Adventures) and [Final Fantasy Crystal Chronicles](https://en.wikipedia.org/wiki/Final_Fantasy_Crystal_Chronicles) technically [work on Dolphin](https://wiki.dolphin-emu.org/index.php?title=The_Legend_of_Zelda:_Four_Swords_Adventures), but it is probably not as good of an experience as the whole point is that each player has a Game Boy as a controller. Some games, including Super Smash Bros Melee and F-Zero GX, are very fast paced and thus extremely sensitive to lag and stutter. These games also often work better on real hardware. In any case, you would still need a Wii or a GameCube in order to [rip your games](https://wiki.dolphin-emu.org/index.php?title=Ripping_Games) for use with Dolphin.

Another pro to using a GameCube (but not a Wii) is support for the [Game Boy Player](https://en.wikipedia.org/wiki/Game_Boy_Player). The Game Boy Player is an add-on that allows Game Boy games (from all Game Boy revisions and models produced) to be played. This is an add-on I had never gotten around to acquiring, so the first step of this journey was to acquire a GameCube and a Game Boy Player.

## Acquiring a GameCube

After looking around a bit I found a great deal for an orange GameCube and Game Boy Player on ebay. The orange GameCube was only released in Japan, so it is actually somewhat rare here in Sweden (and the west in general). Most importantly, it looks amazing.

![](/assets/posts/2017-08-08-the-orange-cube/orange_cube_dirty.jpg)

After receiving the package I discovered that the hardware was packed in dirty newspapers that reeked of cigarette smoke, neat! The hardware itself was fairly unharmed, except that it was dirty and smelled like cigarette. All things considered this was probably to be expected considering the price. The only way forward was to clean the Cube!

![](/assets/posts/2017-08-08-the-orange-cube/disass1.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/disass2.jpg)

The GameCube turned out to be a fairly simple console to disassemble, the internal design is quite simple and elegant. Once all plastic parts had been separated from the internals, they were thoroughly scrubbed and washed with water, soap and isopropanol. After everything had dried the Cube was reassembled. Pictures of the cleaned Cube below:

![](/assets/posts/2017-08-08-the-orange-cube/cleaned2.jpg)

It might be hard to see the difference in the picture, but the Cube now feels almost new (besides some scratches and slight discoloration). Most importantly, it no longer smells like cigarettes.

# Region mod

Now that the hardware was acquired and cleaned it was time to try out some games to make sure it worked right? Not so fast! I previously mentioned that the orange Cube was only released in Japan, which means that it has a Japanese bios and can only play Japanese games.

![](/assets/posts/2017-08-08-the-orange-cube/japanese_bios.jpg)

This is no good, I can't read Japanese. Fortunately, it is possible to mod a Japanese Cube into an American one (or vice versa). All that is necessary is to bridge (i.e. connect using soldering) two points near the GPU on the motherboard. In the images I saw on the internet this seemed fairly simple, so I opened the Cube up again and wow:

![](/assets/posts/2017-08-08-the-orange-cube/bridge_these_points.jpg)

They turned out to be way, way smaller in real life than what I was expecting from the images. See the Swedish 10 SEK coin for comparison (approximately 2cm in diameter). At this point it had gotten quite late (around 00:30) and I was very tired, but I stupidly decided to charge ahead and solder it anyway. After some very nervous attempts I managed to create a blob of solder that looked like it correctly bridged the points without touching anything else. I reassembled the Cube and voila:

![](/assets/posts/2017-08-08-the-orange-cube/american_bios.jpg)

Success! Looking back I should not have done this so late at night, I was way too tired and it is a miracle it turned out well.

At this point I also got a bit of the usual oh-god-what-am-I-doing panic that usually sets in when modding hardware. These things seem a lot easier in tutorials, then you open up the hardware yourself and everything is a lot smaller than they appeared in the instructions. In my experience the best way to continue is to try to stay calm and continue on slowly but carefully. Most of the times things turn out fine in the end, but there has been times where everything just broke. That's a risk one has to accept.

## Installing an HDMI port

The GameCube has a digital and an analog output port. The analog port can at best output 576i ([interlaced](https://en.wikipedia.org/wiki/Interlaced_video)) through an RGB [SCART](https://en.wikipedia.org/wiki/SCART) cable. SCART itself is kind of a problem because modern TVs do not have good support for the port, and often have bad internal scalers to scale up the image. In addition, the interlaced signal often looks terrible on big LCD TVs.

Luckily, the digital port can output 480p (progressive) through [component](https://en.wikipedia.org/wiki/Component_video) cables. Unluckily the official component cables costs a [ton of money](https://www.ebay.com/sch/i.html?&_nkw=gamecube+component+cable), literally several times the cost of a GameCube in good condition. The reason for this is that not that many cables were created, as not that many people were interested (i.e. did not have HD TVs) when the GameCube was new. Normally third party knockoffs cables would have been created, but in this case the cable itself apparently contains a [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter) that converts the raw digital signal from the GameCube to analog video. This makes it expensive and/or complicated to recreate.

![](/assets/posts/2017-08-08-the-orange-cube/digital_port.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/digital_port_no_case.jpg)

However, recently the situation has been improved by a project called [GCVideo](https://github.com/ikorb/gcvideo). This project allows the digital port to be replaced with an HDMI port, all powered by an FPGA. Pre-programmed FPGAs are available for [purchase](http://www.knjn.com/ShopBoards_RS232_Parallel.html), all that is necessary from the user is some soldering. The output from a GCVideo should in theory be even better than that from a Wii, meaning a GameCube is once again the best way to play GameCube games (besides Dolphin).

![](/assets/posts/2017-08-08-the-orange-cube/fpga_and_3dprinted.jpg)

The image above shows the GCVideo FPGA and two [3D printed mounts](https://www.thingiverse.com/thing:2332094) for it. The first step of the installation was to remove the old digital port. Ideally the connections should be desoldered so the port can be cleanly removed, but it was a bit of a pain so I decided to carefully cut it off using a knife instead.

![](/assets/posts/2017-08-08-the-orange-cube/digital_port_removed.jpg)

This worked out well enough, the port itself was a bit damaged. But I didn't need it anymore.

![](/assets/posts/2017-08-08-the-orange-cube/fpga_placed.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/butiful_soldering1.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/butiful_soldering2.jpg)

The solder points was easier to access compared to the region mod, so the soldering was actually a bit easier. It was tedious though and took way more time to finish. I had a bit of a problem with interference for one of the cables, the 54Mhz one, which caused banding artifacts in the output image. In order to fix it I had to replace the cable with a thicker, shorter one and drag it further away from the other ones. In total it took about 6-8 hours for the initial soldering, and then another 6-8 hours to debug and fix the cable with interference.

![](/assets/posts/2017-08-08-the-orange-cube/port_in_place.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/hdmi_installed.jpg)

It was also necessary to modify the heatsink in order to fit the FPGA. The final installation is shown in the images above. I have no images to show, but I can confirm that the resulting image quality is way better than with the standard cables. I have not compared with a component cable, but it is at least as good.

## Installing a mod chip

Since my collection is comprised of both american and european games, I need some way of booting the european ones. A simple way of booting out-of-region games is using a [Freeloader](https://en.wikipedia.org/wiki/Wii_Freeloader), but there are more modern tools, such as the homebrew [Swiss GC](https://github.com/emukidid/swiss-gc). Swiss allows to force different video modes, including 480p for european games which does not normally support them. Swiss can also force some games to render in widescreen instead of 4:3. Aside from Swiss, there are also other homebrews, such as [Game Boy Interface](https://www.gc-forever.com/forums/viewtopic.php?t=2782), that we want to boot.

One of the quickest ways of booting a homebrew program is to burn it to a disc and boot directly into it. The GameCube will of course not normally accept such a disc, as it would allow people to burn and play pirated games. In order to disable this anti-piracy measure and make the system accept any disc a mod chip can be installed. So the next order of business is to install a mod chip!

![](/assets/posts/2017-08-08-the-orange-cube/mod_chip.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/mod_chip_placed.jpg)

This is however where the story turns sad. The mod chip had a quicksolder board, which basically means you just place it on the board and solder it directly to the solder points. It's supposed to be "easier" to solder, but in the back of my head I had a voice telling me that I should be using wires anyway. Unfortunately, I ignored it and tried anyway.

Of course points that should not be bridged got bridged underneath the chip. If I had been using wires it would have been a simple matter of fixing it. But because it was underneath the chip I first had to desolder everything and remove it. This turned out to quite hard because the solder slid in more underneath the chip at each attempt. In the end the chip got ruined and some of the solder points got ripped of in the process.

![](/assets/posts/2017-08-08-the-orange-cube/mod_chip_failed.jpg)

![](/assets/posts/2017-08-08-the-orange-cube/mod_chip_borked.jpg)

Luckily, the drive itself still worked fine after this botched attempt. This whole event ties back a bit to what I previously wrote, sometimes you will screw up. At this stage it is good to consider why you screwed up, so you can gain some wisdom for next time. The valuable lesson this time was to not use quicksolder boards, as they are harder to remove if something goes wrong. The next step is to remain calm and consider ones options going forward.

One option would be to buy a new GameCube and mod chip and try again. The mod chip is installed to the drive assembly, so it would just be a matter of swapping the drives. The HDMI mod would not need to be reinstalled. I considered this for a while, but decided against it as the project was starting to take up a bit too much time.

The next best option was to "softmod" the GameCube. A softmod is essentially a hacked save file that triggers an exploit in a game, and then loads custom code specified by the user. I choose to install the [WW-Hack](https://github.com/FIX94/ww-hack-gc), which crashes the game Zelda Wind Waker after the title screen and loads homebrew. This way I am able to boot homebrew and do everything I would have been able to with a mod chip. The drawback is of course that I have to boot Wind Waker and go through the splash screens every time I want to use homebrew.

## Conclusion

I don't really know if there is a conclusion at this point. I now have a beautiful orange GameCube with an HDMI port. I can boot european games in 480p by crashing into Swiss from Wind Waker. Similarly I can play Game Boy games on the Game Boy Player by first crashing into Game Boy Interface from Wind Waker. Will I play games on this system instead of using Dolphin in the future? Perhaps sometimes, but mostly I'm still going to be using Dolphin. I don't regret it though, it was a fun project all things considered.

I find it fascinating the things you can do with hardware modifications. As a software guy I'm amazed that you can make things work at all without any software modifications to the original hardware. In my mind its something that should not work, but clearly it does. Maybe someday I will attempt to actually understand how all this stuff really works, so I can create modifications of my own. But I suspect there is probably years of learning needed to get to that level, which is a tad bit more time than I'm currently willing to spend on it.