# Porting Linux kernel

Here are some notes about porting linux kernel, with an example of the Midas (Note 2, S3) boards. A bit disclaimers here: probably this guide is not by any means complete, nor it's claiming to be an ultimate truth, so any suggestions and updates to this are welcome.

Ok, so lets get started. I'm assuming that you already know how to compile a Linux kernel and how to work with the Git version-control system, so I won't cover it here.

First of all, I'm assuming we already have some working kernel source. 
In this example I'll start with the Midas mainline [kernel](https://github.com/fourkbomb/linux/commits/zz-old/linux-4.13) provided thanks to Simon Shields. Unfortunately, we currently lack support of some drivers like the GPU and the DRM subsystem. Of course we have a [Lima](https://gitlab.freedesktop.org/lima/mesa) driver in the kernel, but getting this to work probably deserves another article on that.
So, we also have some older kernels, e.g. the LineageOS kernel (Linux 3.0), mine upgrade of LineageOS kernel to Linux 3.4, [Dorimanx kernel](https://github.com/dorimanx/Dorimanx-SG2-I9100-Kernel) (partial updates upto Linux 3.14). An ultimate point is to take some drivers from these kernels and port them into the mainline.

Ok, starting by now and just porting any stuff from such and old kernel version 3.4 to the mainline doesn't seem any straightforward by now.
The reason why I'm starting with Linux 4.13 instead of the mainline one (which is at Linux 5.0 at the moment of writing) is because we want to downgrade our mainline kernel a bit in order it was possible for us to port our old drivers, lets say, into Linux 3.18 and hopefully get them working, do some tests and upgrade the kernel back to the mainline now with the support of the newly ported drivers.

## The algorithm

A quite important part here is to get some basic stuff in the mainline kernel working, e.g. the display, USB so that we can know our kernel port is alive rather than dead. It's very convenient, that the mainline kernel already has the support for all the basics stuff (CPU, DRAM, USB, display etc) works thanks to a brilliant work of Simon Shields. We also have a live kernel boot up logs on the screen, so we can judge if the kernel is alive or dead quite quickly.

So our next steps will be:
  1) Downgrade the kernel version to e.g. Linux 4.12
  2) Rebase our patches on top of this new kernel version
  3) Try to boot this up
  4) Doesn't boot up or crashes (or any other quirky stuff you could notice)? Go to step 1 and try a higher kernel version.
  
Quite a dumb approach, as some might've been noted, but this is what I came up with so far. This doesn't require for you to know (in the most of the cases), why the kernel isn't booting, you can do a binary search to find a working kernel version.

For now might sound quite easier than it's, but I'll cover in a details how we do that. For our convenience the Git version-control system provides an instrument that can automate this process a bit. I'm talking about `git bisect`. It was seem to me that this is usually used to find a first bad commit in our project, but since we are going to downgrade the kernel version, essentially we're searching for a first good commit. So I had to do some googling on that to find how to use `git bisect` in our case, and I've found an answer on [StackOverflow](https://stackoverflow.com/questions/15407075/how-could-i-use-git-bisect-to-find-the-first-good-commit):

> As of git 2.7, you can use the arguments --term-old and --term-new.
> For instance, you can identify a problem-fixing commit thus:
> `git bisect start --term-new=fixed --term-old=unfixed`

Now you can mark good kernel revisions with `git bisect fixed` and the bad ones with `git bisect unfixed`.

So, to wrap things up, the below is step by step algorithm:
1) Start by making another local clone of the git repository you're working. Because doing `git bisect` on repo will always reset the HEAD commit, we don't want this mess on the main kernel repo, which we actually using for compiling the stuff. So we will use this repo only for bisects:
`cd kernel/samsung`
`git clone midas midas1`
`cd midas`
`git -C ../midas1 bisect start --term-new=fixed --term-old=unfixed`
2) Lets assume our working revision is Linux 4.13 and we want to downgrade to Linux 4.12 (which isn't booting straight away):
`git -C ../midas1 bisect fixed lk-4.13`
`git -C ../midas1 bisect unfixed lk-4.12`
3) So we got some new git revision to test in our midas1 repo, lets make it some meaningful name. We are currently in between of Linux 4.12 and 4.13. To show our progess we want to know how far we are from Linux 4.12:
```
check_count() {
     git log --oneline $1...$2 | wc -l
}
```
   I'm implementing a bash function that will show amount of commits between commit 1 and 2. And to actually checkout to this revision:
```
checkout() {
    base_branch=$1
    commit=$2
    git checkout -b $base_branch+$(check_count $base_branch $commit) $commit
}
```
`checkout lk-4.12 <SHA-1_commit_you_got_from_git_bisect>`
This will create and checkout to the new branch with the following name:
`lk-4.12+xxxxx`, xxxxx is the amount of commits from Linux 4.12 to our revision we're currently testing.

4) At this point we want to rebase our patches from Linux 4.13 to this new branch, e.g. cherry-pick a list of those commits will suite our needs. Resolve merge conflicts if necessary.

5) Finally that we have a buildable kernel, we can build the kernel. I'll leave the build script that I'm using for kernel build for S3 [here](https://gist.github.com/ChronoMonochrome/8cc4d08cb2172765c6ce5cbc6767f38f).
Note that this require some Android build system stuff, like host binaries and the ramdisk. So for first build the kernel with `mka bootimage`, then adjust paths in the script as necessary, and you can switch to using this script. It is convenient to have the current testing branch in the build kernel image name, so this script will produce a boot image with the following name: `boot-<branch name>-<revision>.img`
Some more links for compiling TWRP with the mainline I'll include below.

6) As I've previously mentioned, at this step you need to test the built kernel. Not working? Go to step 1 and mark tested revision: `git bisect unfixed`.

7) Found a working revision? Respectively, this should be marked it as fixed.
At this point you might want to repeat these steps over and over until you can deduce the difference between working and non-working revision is relatively small, and you can figure out what exactly commits are containing this diff.

## Misc

There are some examples of commits found by bisecting in my github repository:

https://github.com/ChronoMonochrome/android_kernel_samsung_smdk4412/commits/fixes/lk-4.5_ak8975 (you can find them in those branches prefixed by `fixes/` in the branch name).

At the moment of writing, the lowest successfully booted kernel version is [Linux 4.5](https://github.com/ChronoMonochrome/android_kernel_samsung_smdk4412/commits/linux-4.5). By saying "successfully booted" I mean it at least doesn't crash during the boot process and doesn't lead to any unusual behaviors.
The more work is probably will need to be done, because I never tested how this kernel is working in the actual Android builds due to the lack of the working GPU drivers. Anyways, that all I can do by now.

Also I've started by testing if kernel can work in TWRP first, but then I just realized that the actually booting an Android system is bit more complicated and indeed it led to some more fixes. 
Anyway I'll leave [build script for TWRP](https://gist.github.com/ChronoMonochrome/c5a7d1c5816427c0e79a1045b79dc049) as well. As previously, this still needs building recovery image with `mka recoveryimage` in the Android build system first, then you can use this script.

An important note: for some reason, I couldn't get graphics in TWRP to work with the LineageOS S3 device tree.
Probably that's because due to a huge difference between our stock 3.0 and the mainline 4.X kernels. [An unified Midas device tree](https://github.com/LineageOS-midas/android_device_samsung_midas/) worked fine though.
