# Emma's Xbox 360 Research Notes

My personal reversing and research notes on the Xbox 360 that I've come to
understand from my own reverse engineering efforts, as well as information I've
gleamed from other projects and learned from other fellow hackers.

This is all in a Git repo in lieu of a solid wiki for documenting Xbox 360
software. If you copy any of it out to a wiki, please attribute back to this
repo.

I'm trying to document as much as I can about:
* Official, offline software (bootloaders, hypervisor, kernel, etc)
* Behaviour of production and devkit hardware and accessories
* "Essential" homebrew (NAND image patches, XeLL/libxenon, Dashlaunch)

I don't plan on documenting anything relating to Xbox Live, or community-made 
homebrew. I also won't be posting any key material here - although the information
I provide could send you well on your way for finding that out for yourself.

**Please don't let any of this information die with me,** but please don't take it
as entirely your own. And if you're reading this and know anything useful, please
write it down and publish it somewhere, ideally a public wiki or Git repo.
You never know who might need it in the future, when you forget or aren't around.

An additional request is that if you found anything I've written here useful,
please make whatever you make with it open source (or have plans to) - too many
times in this scene has a useful piece of software been made, but decayed over
the years and unable to be fixed as the source code is long gone. Of course,
this choice is still up to you. Please, take the right one.

If you want to contribute to any of these pages, feel free to send a pull request,
and put your name on it.

## Shoutouts

There's too many people to shoutout here. But as for projects and as sources of
information:

* The [Free60 Project](https://github.com/Free60Project), [wiki](https://free60.org/)
  and [libxenon](https://github.com/Free60Project/libxenon).
* emoose's [ExCrypt](https://github.com/emoose/ExCrypt) and
  [idaxex](https://github.com/emoose/idaxex).
* GoobyCorp's [Xbox_360_Crypto](https://github.com/GoobyCorp/Xbox_360_Crypto).
* The [Xenia](https://github.com/xenia-project/xenia) and
  [Xenia-Canary](https://github.com/xenia-canary/xenia-canary/) emulator projects.
* The [RGLoader](https://github.com/RGLoader) project, and the
  [XDKbuild](https://github.com/xvistaman2005/XDKbuild) project.
* [XenonLibrary wiki](https://xenonlibrary.com/wiki/Main_Page)
* [XenonWiki wiki](https://www.xenonwiki.com/Main_Page)
* [DrSchottky's X360 Reversing tutorials](https://www.razielconsole.com/forum/guide-e-tutorial-xbox-360/943-%5Bx360-reversing%5D-intro.html)
* [TEIR1plus2's Xbox-Reversing](https://github.com/TEIR1plus2/Xbox-Reversing)
* [ihatecompvir's x360-hv-notes](https://github.com/ihatecompvir/x360-hv-notes)
* [Grimdoomer's Xbox 360 hypervisor article](https://icode4.coffee/?p=1047)
* [Byrom's Xbox_360_Research](https://github.com/Byrom90/Xbox_360_Research)

I likely wouldn't know nearly as much as I do if not for all the great open source
projects detailing how parts of this system work, and all the people behind them. 

And an extra shoutout to folks from the #coding-corner channel in the
[Xbox 360 Hub](https://xbox360hub.com/) Discord.

## Removal Requests

If you've written any of the homebrew that I've documented the internals of, and
you'd like it removed, please don't hesitate to contact me and ask me to delete
it. I'll gladly oblige.
