---
layout: post
title: "10 steps to submit patches to the Ubuntu Kernel"
date: 2017-08-12 22:17:42 +0800
comments: true
categories: [Kernel, Upstream]
published: false
---

[Ubuntu Kernel 攻略 - 十個步驟讓你送patch到Ubuntu kernel]
https://twlinuxkernelhackers.hackpad.com/ma0mqwjifEs…
一般distribution kernel使用的版本比較舊，各個distribution需要完善kernel的穩定性，會持續的backport bug-fix到自己的kernel版本。以Ubuntu來說目前線上的kernel版本有
Ubuntu 12.04 Precise 3.2 (2012 04發行)
Ubuntu 14.04 Trusty 3.13 (2014 04發行)
Ubuntu 14.10 Utopic 3.16 (2014 10發行)
Ubuntu 15.04 Vivid 3.19 (2015 04發行)
其中Precise和Trusty是LTS的版本，分別support 五年。Precise到2017，Trusty到2019。如果自己要安裝server的話，建議用LTS的版本，會有比較長的maintainance。
詳見下圖：
https://wiki.ubuntu.com/Kernel/LTSEnablementStack
接下來切入主題，要如何送patch做backport，也就是俗稱的SRU(Standard Release Update)程序[3]。一般Ubuntu kernel是不收沒有上Upstream的patch，所以要記得先送kernel upstream再cherry-pick回來。拿個自己的bug為例 https://lkml.org/lkml/2015/4/23/228：
commit 4066c33d0308f87e9a3b0c7fafb9141c0bfbfa77
Author: Gavin Guo <gavin.guo@canonical.com>
Date: Wed Jun 24 16:55:54 2015 -0700
mm/slab_common: support the slub_debug boot option on specific object size
因為slub_debug在實作上有些問題，導致沒有辦法把slub_debug限制在某些object size上面。上面commit是bug fix。由git show可以知道這是在June 24才剛進去mainline(在v4.2-rc1的merge windows裡面)。因此所有的Ubuntu release都受影響。進而需要backport這個commit到所有的Ubuntu release。最後因為某些原因，SRU只做Trusty, Utopic, and Vivd。
1) 先到 launchpad的linux package開個public bug: 
support the slub_debug boot option on specific object size
https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1456952
在bug description[1]裡面記得填寫
[Impact]: 現象和造成的問題
[Fix]: upstream相關的fix, commit 之類的資訊
[Test case]: 如何測試
並且加入這個bug影響的release到tags，ex: Trusty, Utopic, and Vivid.
2) 到 http://kernel.ubuntu.com/git/?ofs=650 找到Trusty的tree
git://kernel.ubuntu.com/ubuntu/ubuntu-trusty.git
# clone trusty的kernel tree到本地端
git clone git://kernel.ubuntu.com/ubuntu/ubuntu-trusty.git
git remote add linus git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
git fetch linus
3) 分別針對Trusty, Utopic, and Vivid 複製一份 branch
git checkout -b backport-trusty-1456952 origin/master
git checkout -b backport-utopic-1456952 origin/lts-backport-utopic
git checkout -b backport-vivid-1456952 origin/lts-backport-vivid
4) cherry-pick commit到本地的branch
值得一提的是-x 會把commit 最後加上
(cherry picked from commit 4066c33d0308f87e9a3b0c7fafb9141c0bfbfa77)
可以很清楚的知道這個cherry-picked commit來自upstream的哪個commit id
git cherry-pick -s -x 4066c33d0308 -e
# -e 有加上所以會進入edit mode。之後在 commit title 之後空一列加入
# BugLink: http://bugs.launchpad.net/bugs/1456952
# 這時候常常不會很順利可以clean cherry-picked，在fix完conflict，git add file, git cherry-pick --continue以後，要把commit裡面的cherry picked改成backported。並且用scripts/checkpatch.pl掃過確認沒有格式問題。
5) 產生patch並且修改為Ubuntu patch格式[2]
git format-patch -1 --cover-letter
# 這之後會在當下目錄生出
0000-cover-letter.patch
0001-mm-slab_common-support-the-slub_debug-boot-option-on.patch
vim 0000-cover-letter.patch把Subject改成
[Trusty/Utopic/Vivid][SRU][ PATCH] LP#1456952 -- mm/slab_common: support the slub_debug boot option on specific object size
commit message加入剛剛在bug description裡面寫的[Impact][Fix][Test Case]。
0001-mm-slab_common-support-the-slub_debug-boot-option-on.patch patch的Subject也要加上[Vivid][SRU]。
在checkout到其他的branch之前記得把0001-mm-slab_common-support-the-slub_debug-boot-option-on.patch更名，免得format-patch會被蓋掉。因為Subject相同，所以patch名稱也會相同。再來就是checkout到backport-utopic-1456952和backport-trusty-1456952。個別再做一次cherry-pick，加入buglink。然後git format-patch -1，這次就不用加上--cover-letter。
如果真的不熟悉，可以到Ubuntu mailing list上面看看大家怎麼送的。慢慢就會知道格式怎麼寫。
6) build kernel並且測試cherry-picked commit是否工作正常[4]
fakeroot debian/rules clean binary-generic
sudo dpkg -i xxx.deb
# doing test cases
7) 註冊Ubuntu kernel team mailing list
到 https://lists.ubuntu.com/mailman/listinfo/kernel-team 註冊maillist之後就會收到Ubuntu kernel開發的email。
8) 用git送出patch到Ubuntu kernel mailing list
git send-email --no-chain-reply-to --suppress-cc all --to "kernel-team@lists.ubuntu.com" 000* --force
# 在送出patch之前，一定要先送一份到自己的mail，檢查有無錯誤內容。
9) Waiting for Acks
在 kernel mailing list收到maintainers兩個人Ack以後就會被apply到你所tag的release git tree的master-next。然後進入到Proposed kernel。
10) Verification
最後等到proposed kernel出來以後，系統會寄mail通知你。這時候要下載ppa上面的proposed kernel來做測試(系統會自動寄出comment並且提供link做完整的說明)，看看是否解決了bug。如果測試通過則把剛剛所申請 launchpad bug 裡面的verification-needed tag改成verification-done。通過之後update版本就會包含這個patch。
Reference: 
[1]. KernelUpdates https://wiki.ubuntu.com/KernelTeam/KernelUpdates
[2]. StablePatchFormat https://wiki.ubuntu.com/Kernel/Dev/StablePatchFormat
[3]. A day in the life of a fixed bug https://wiki.edubuntu.org/Kernel/Dev/KernelBugFixing
[4]. BuildYourOwnKernel https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel
#kernel #ubuntu #upstream #tlkh_upstream
