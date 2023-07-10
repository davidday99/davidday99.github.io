# The Magic of Bootup

For as long as I've known of the boot stage of a computer, I've been fascinated by it. 
I use boot stage loosely to refer to pretty much everything that happens between power-on 
and the moment CPU control enters whatever kernel is loaded into memory. 

Part of the reason I enjoy this level of computer startup is because the boot stage serves as a sort of bridge
between hardware and software. It initializes the physical system in all its ugliness
(or beauty; I suppose it depends on who you ask.) and hands to the operating system a set of relatively simple 
interfaces for leveraging all the powerful peripherals attached to the processor.

Another reason for its appeal is its mystique. In my experience, it's not so easy finding resources describing low-level boot processes. 
Much of what I find are either consumer-oriented Youtube videos that walk you through disabling UEFI
or, on the opposite end of the spectrum, dense 1000-page technical documents. I haven't come across any substantial Geeks for Geeks
articles, unfortunately, although I have stumbled upon a few excellent blog posts such as
[this one](https://www.happyassassin.net/posts/2014/01/25/uefi-boot-how-does-that-actually-work-then/).


I think bootup appeals to me in a similar way that the Big Bang appeals to physicists. 
