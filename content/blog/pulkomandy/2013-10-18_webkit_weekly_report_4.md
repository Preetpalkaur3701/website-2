+++
type = "blog"
author = "PulkoMandy"
title = "WebKit weekly report #4"
date = "2013-10-18T07:07:23.000Z"
tags = ["contract", "webpositive", "WebKit", "serviceskit", "Services Kit", "contract work"]
+++

Time for a report again !

So, over last week-end and monday morning I finally got Ninja working. I already said some words about it, Ninja is meant to replace Make for building projects. It has less features, because it is designed to run files that are generated, rather than hand-written. In my case, CMake generate the Ninja failes. I had problems building Ninja that turned out to be  abug in our Python port. I didn't fix it, but I found a way to work around it.

The use for all this buildsystem work? Well, we're now using one of WebKit standard build systems (CMake), and using it in a way that's fast! CMake can also generate makefiles, but running these would take about a minute just to figure out which files to recompile. With Ninja, this is done in a matter of seconds, because the simpler rules are faster to parse. Ninja was actually written just for that purpose. One minute may not seem much when the full build of WebKit takes about an hour, but when I'm working on just one or two files to track down a bug, the edit/compile/test cycle is now much faster.

So, with this out of the way I was able to more easily try out things and fix the issues as I found them. I mentionned the Opera cookie testsuite in last week report : most of the tests are now passing, and we even get better results than the old Curl backend in some places. The remaining issues are rather minor ones, but I can fix them if you can find a website where they actually matter.

In one of the blog post comments someone mentioned that Windows Live Mail (or Outlook, as it's now called) didn't allow to display messages. I tried to do that with HaikuLauncher and the new code and noticed I couldn't even login to the website. Getting this to work was a bit tricky, leading to fixes in 3 different places: settings cookies from Javascript code, proper handling of URL redirections, and telling WebKit that "aspx" files served without a mimetype are web pages.

I already told you about the cookies, so here are some more details about the two other issues: the URL redirection problem happened when an HTTP server gave us a 302 answer (that is, a redirection) with a relative URL. The HTTP specification says an URL should be used, most likely starting with "http://". But the URL specification was later replaced with the URI one, which allows URIs without that part, for example "/path/to/page.aspx?param=foo". Our BUrl class wasn't able to understand such things and was desperately looking for the "//" part, leading to an endless loop. I replaced the homegrown parser with a regular expressionbased one, using the regex that is kindly provided inthe RFC for URIs. I had to fix our RegExp class in the shared kit (this is a collection of internal classes that are not, yet, part of the official API). So, I had to fix the RegExp class as it didn't handle optional capture groups in the regular expression properly. This includes things like: the URI may, or may not start with "scheme:", then, maybe there will be an authority part, of the form "//authority", then a path, which is anything following the authority and up to th first ? or # or the end of the URI, and then there is the query and fragment, and they are also optional. RegExp allow to write this in a more compact and computer understandable format and get the parts extracted to separate strings.

As for the "aspx" file handling, the HTTP protocol says that the server should give a "content-type" header with the MIME type of the file it's going to send. This fits well with the use of MIME types in Haiku and would be very nice. However, Microsoft servers for login.live.com sometimes don't include the header. Web+ used to assume the file was to be displayed on screen in that case. You may already have seen it trying to render a video file as plain text, as a huge page full of strange characters. Well, that issue was fixed, and if in doubt, it will now download the file to disk instead. A list of well-known file extensions is used to attempt a guess when the "content-type" is missing. I added aspx to that list and mapped it to text/html, so, the URLs ending in aspx will now be shown on screen.

I had a run of testing after all these fixes and things are getting much better. It helped fix some more issues in github and gist, and I could also browse some documents in Google Drive. I think we still have some problems with file upload, however, I'll have to look into that. Uploading video to youtube, as someone pointed out in the forums, also fails.

I caught a crash when opening a new window in HaikuLauncher, but I'm not sure if that would be a problem for Web+. Also, I wanted to work on HTTP authentication (the login/password prompt) and this is not available in HaikuLauncher, so a working Web+ will be needed. To get there I wanted to build a package for WebKit. To do this, I first tried writing an haikuporter recipe. I didn't get very good results with this. First,it means working separately from my development git checkout, so it takes twice the sapce on disk (and we're talking multiple gigabytes) and it takes forever to checkout. Second, there's no easy way to use distcc with haikuporter, so things are keeping my CPU very busy for a long time. And since this isn't my development tree, there are a lot of things to rebuild.

So, I decided to try a different solution: using CMake built-in support for generating packages. I tried that yesterday and found out that it wasn't working. I tried on a very small test project, and with help from the CMake IRC channel I found the issue: our CMake port is lacking some features that are available in the Linux version, and not needed on Windows. This confuses the Ninja generator which isn't ready for that case. This is a rather technical issue related to the way executables and libraries are linked together. This is different in the tests (the lib and executable sit together in a directory) and installed version (they are installed somewhere on the filesystem and the OS, more precisely the runtime_loader) must bind them together. On Linux, CMake makes up for that by rewriting part of the executables when installing or packaging them. This isn't enabled in the Haiku version, so it will instead try to link executables again when installing them. However, the Ninja generator is skipping the link step (it was never tested in that case) and tries to install or package non-existing files. This shouldn't be too hard to get working just like on Linux, more news next week !

There is another missing step to make this complete: CPack, the CMake component that generates packages, knows how to make tar archives (with various compression schemes), zip archives, and deb and rpm packages. This is not very useful for us, so I started adding support for hpkg files there. This will make it super-easy to get a package built from any project using cmake: "make package", or "ninja package", depending on the generator you decided to use.

Anyway, I've uploaded an unsupported build in a manually crafted zip file so you can try things for yourself: http://pulkomandy.tk/drop/HaikuWebKit_20131018.zip
I'm waiting for your reports on how well it's working. Note you need a fairly recent build of Haiku (a nightly from today or yesterday should do) as I made some changes to the BHttpRequest class again, to fix some API design issues that led to memory leaks. That class was also added to the Haiku Book.

That's it for this week!