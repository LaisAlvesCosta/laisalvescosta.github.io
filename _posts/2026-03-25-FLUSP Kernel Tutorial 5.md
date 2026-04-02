---
categories: [Dev, Kernel_Dev]
date: 2026-03-25
tags: [kernel, mac5856]
title: Tutorial 5 Sending patches by email with git
---

(available at <https://flusp.ime.usp.br/git/sending-patches-by-email-with-git/>)

For this tutorial, we were instructed to read and understand the workflow without necessarily executing all the steps.

One of the goals of the **MAC5856 Free Software Development course** is to collaborate on a real open-source project. Contributing to someone else's codebase is not trivial, especially in large projects like the Linux kernel. Because of that, the tutorials from this point forward start preparing us for contributing patches to the kernel.

As I mentioned before, I am still a beginner in this area. To help consolidate what I learned and possibly help other beginners, I decided to write a short summary of some important concepts.

## What is a patch?

Before learning how to send patches, it is important to understand what a patch actually is. If you search online, you may find definitions like:

> “A small piece of material, code, or data designed to fix, update, or improve a larger item.”

However, in practice (especially in kernel development), I find the following definition more helpful:

> A **patch** is a small file that contains:
>
> - a short commit message (usually less than 50 characters)
> - a description of the changes
> - a diff showing the modifications in the code

In other words, a patch represents a **proposed change to the codebase**. However, it is **not just any file**. There are formatting conventions and style expectations that must be followed so maintainers can properly review the changes.

## Who are maintainers?

Maintainers are the people responsible for reviewing and integrating changes into a project. They are typically responsible for:

- reviewing patches
- providing feedback
- accepting or rejecting contributions
- merging changes into the repository

When you send a patch, you need to **convince the maintainers that your change should be included**. Because of this, writing a clear and well-structured patch description is very important.

## Structure of a good patch:

A well-written patch usually contains two main parts.

### 1. Subject

The subject line is a **short description** of the change. It should:

- be concise
- clearly indicate what the patch does
- include the appropriate **driver prefix**

For example, if the patch modifies a **staging driver**, the subject should use the prefix used in previous commits for that driver. It is also recommended to include the **checkpatch warning or error** that the patch fixes.

### 2. Body

The body of the patch explains **why** the change was made. This is important because the **"why" is usually more important than the "what"**. If your description only explains what you changed, for example:

> "I changed function X to not do Y"

then the description is probably not detailed enough. A good patch description should:

- explain the motivation behind the change
- provide context for reviewers
- help maintainers understand the impact

At the end of the description, the patch must include the **Signed-off-by** line. For example: "Signed-off-by: Your Name email@example.com". Sometimes it is also useful to include the **full output of a checkpatch.pl warning or error** that the patch fixes.

## Patchsets

Sometimes one change is not enough. When multiple related patches are needed, they are grouped into a **patchset**. A patchset is a **series of patches that implement a larger change**, such as:

- adding new functionality
- fixing several related bugs
- refactoring a driver

Instead of sending one huge patch, the idea is to **break the work into smaller logical steps**. When sending a patchset, each email subject is automatically formatted like this: [PATCH 1/N], [PATCH 2/N], [PATCH 3/N], etc. Where `N` is the total number of patches.


## Cover letters

When sending a patchset, it is common to include an additional email before the patches. This email is called the **cover letter**. It usually:

- summarizes the purpose of the patchset
- explains the overall change
- provides context for reviewers

The cover letter is typically indexed as: [PATCH 0/N]

## Useful options when sending patches:

Two options mentioned in the tutorial are particularly helpful.

- **Dry-run**: This option allows testing the email sending process without actually sending the emails. This is very useful for verifying that everything is correct.

- **Annotate**: It allows you to review and edit the email before sending it. So, it provides a final chance to check the patch content, the email formatting and the commit message.

- **Patch versions**: Sometimes maintainers request changes. When resending an improved version of a patch, you usually increment the version number: v2, v3, etc. It is good practice to include comments explaining what changed between versions, which helps reviewers understand the differences between iterations.

## Sending patches with git

Git provides tools that simplify this workflow. One common approach is:

1. Use `git format-patch` to generate patch files.
2. Send those patches via email using `git send-email`.

At first glance, the patch workflow may seem complicated, since there are many conventions and rules to follow. However, these conventions are important for maintaining code quality in large projects like the Linux kernel.

Another option (which we will see later) is letting `git send-email` handle both the formatting and the sending of patches. This approach is convenient because it reduces the chances of formatting mistakes. Even though the tools automate part of the process, it is still important to understand what each step is doing and how the patch files are generated. Before moving on to the practical tutorial, I spent some time reviewing these concepts and reading additional links referenced in the tutorial to better understand the workflow.
