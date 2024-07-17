+++
title = "My First Patch to Linux"
date = "2024-07-17"
description = "it wasn't good but it got accepted anyways but there's a twist"
[taxonomies]
tags = ["dev"]
+++

About half a year ago, I [submitted](https://lists.freedesktop.org/archives/amd-gfx/2024-January/102810.html) my first patch to Linux. I initially thought the patch wasn't good, but I discovered something as I finished writing this blog post. Anyways, it was interesting to learn a very different system for submitting code to upstream.

### Discovering the issue

I got a brand new graphics card and I wanted to play with it. But Linux wouldn't let me. Some
searches led me to [this issue](https://gitlab.freedesktop.org/drm/amd/-/issues/2991) where people have bisected the cause. So why don't we just revert the commit that caused it? I did that, tried
running with the kernel with that commit reverted and the issue got fixed again.

So.. there's probably something wrong in that commit. Although I have no experience in writing graphics drivers in C, submitting the revert upstream should be easy..

### Following the guide

Okay. Linux uses Git. Surely there's not much difference to contributing to a project on GitHub, right? Well, no. Let's do a brief walkthrough of the [guide](https://docs.kernel.org/process/submitting-patches.html) to submitting a patch to Linux:

* Clone the code. See? It's the same as.. wait a second. Okay. There are many different trees other than the mainline tree and depending on where you send the patch to, you may want to choose the tree _they're_ using.
* Write the change. Well this part is the same as developing on any project, except you'd just be writing code for Linux instead.
* Commit. You'd need to add the relevant Closes: and the Signed-off-by: tags. Not too hard, but still different.
* Submit the patch. As the last step, you have to send an email to the correct place that would review your patch. You'd either have to configure Git to authenticate to your email and send an email with `git send-email`, or configure your email client to send in plain-text your patch.

### What went wrong?

Well.. I had no trouble _producing_ a patch that could get sent upstream, in the end, it was just a simple revert. Even though there were some conflicts, editing the code for a revert was still quite easy. _Sending_ the patch was a bit of nuisance, as I had to configure gmail to send in plain-text mode and copy the patch there, but still, there were no problems.

The problem was something in between. Unfamiliar with the patch format as I was, I got confused about indentation.

```patch
@@ -345,18 +339,18 @@ static void kfd_init_apertures_v9(struct kfd_process_device *pdd, uint8_t id)
        pdd->lds_base = MAKE_LDS_APP_BASE_V9();
        pdd->lds_limit = MAKE_LDS_APP_LIMIT(pdd->lds_base);

-       pdd->gpuvm_base = PAGE_SIZE;
+       /* Raven needs SVM to support graphic handle, etc. Leave the small
+        * reserved space before SVM on Raven as well, even though we don't
+        * have to.
+        * Set gpuvm_base and gpuvm_limit to CANONICAL addresses so that they
+        * are used in Thunk to reserve SVM.
+        */
+       pdd->gpuvm_base = SVM_USER_BASE;
        pdd->gpuvm_limit =
                pdd->dev->kfd->shared_resources.gpuvm_size - 1;

        pdd->scratch_base = MAKE_SCRATCH_APP_BASE_V9();
        pdd->scratch_limit = MAKE_SCRATCH_APP_LIMIT(pdd->scratch_base);
-
-       /*
-        * Place TBA/TMA on opposite side of VM hole to prevent
-        * stray faults from triggering SVM on these pages.
-        */
-       pdd->qpd.cwsr_base = pdd->dev->kfd->shared_resources.gpuvm_size;
 }
```

.. Am I sure that the code in the `+`s of the patch are aligned with the rest of the code there?
Wouldn't that just have one less space when you apply the patch? Shouldn't the patched parts have
an additional space compared to the rest?

In the midst of questioning myself and indentation of patches, I decided to add an extra space to all
the lines starting with a `+` in that patch. That resulted in [this](https://github.com/torvalds/linux/commit/0f35b0a7b8fa402adbffa2565047cdcc4c480153). Indentation looks out of place, all due to my poor formatting..

### Huh?

Okay wait. I wanted to end my blogpost there. That was my original plan. I write about an indentation mistake I did when submitting a patch for Linux, for some reason they decided to accept it, but it was my fault, I would apologize for that and hope people understand the mistake as a first-time contributor. But the indentation looks [perfectly fine](https://gitlab.freedesktop.org/agd5f/linux/-/commit/0f35b0a7b8fa402adbffa2565047cdcc4c480153) in the reviewer's git repo.

Did they fix it for me? Why did the diff on mainline Linux look like that then? Well I had a theory. The indentation in different trees were different, so when a patch based on the amd tree got sent to the mainline, the indentation did not match the code on the mainline tree. So it was not something caused by poor indentation on my side..[^1]

I had taken on guilt for poor indentation that I thought landed to the mainline. When the reviewer probably fixed it themselves, and the indentation problem I thought was caused by me was probably due to something else. And blogging about it was how I discovered the truth. Oh well.

### Closing

So that's my experience of sending a patch to the Linux kernel for the first time. Rust for Linux looks like it is being actively developed. If it actually goes to the mainstream, it might give me more opportunity to contribute to Linux with a language that I actually understand. Hopefully, this won't be the last patch I will submit to Linux!

[^1]: Well that's what I _think_ has happened. If anyone know what actually happened please let me know.