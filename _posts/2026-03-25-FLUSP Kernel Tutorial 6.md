---
categories: [Dev, Kernel_Dev]
date: 2026-03-25
tags: [kernel, mac5856]
title: Tutorial 6 Sending patches with git and a USP email
---

(available at <https://flusp.ime.usp.br/git/sending-patches-with-git-and-a-usp-email/>)

In this tutorial we move from theory to practice, configuring the tools necessary to send patches using a **USP email account**.

In the previous tutorial, we learned how patches are structured and how they can be sent using `git send-email`. However, using a USP email account requires additional configuration. Normally, Gmail allows external tools to authenticate using **App Passwords**, but this requires **2-Step Verification**, which is not available for USP-managed accounts. Another option, **Less Secure Apps**, has been deprecated and is also disabled for USP emails due to security restrictions. Because of these limitations, `git send-email` cannot authenticate directly with USP email accounts. To solve this problem, the tutorial introduces an **email proxy** that handles the **OAuth authentication process**. 

The tutorial is divided into three main parts:

- Running the email proxy
- Configuring `git send-email` with `kw send-patch`
- Testing the configuration

## Running the email proxy

The first step is to run the proxy container using Docker. While following the instructions, I encountered a small issue. The correct command requires a hyphen between docker and compose: `docker-compose up --build`. Without the hyphen, the command fails.

## Finding the container name

When trying to enter the container, I initially ran: 
`docker exec -it emailproxy-container-server-1 /bin/bash`. This produced an error: permission denied while trying to connect to the docker API. Running with `sudo` solved the permission issue, but another error appeared: "No such container: emailproxy-container-server-1". To identify the correct container name, I used: `sudo docker ps`. 
The output showed the actual container name: "emailproxy-container_server_1". Notice the underscore (`_`) instead of the hyphen (`-`). After correcting the command, it worked.


## OAuth authentication

After entering the container and starting the proxy, the terminal displayed a URL. By opening this URL in the browser, it is possible to authenticate using the USP email account. Once the authentication is completed, the browser redirects to a URL that fails and starts with something like:

`http://localhost/?code=...`

This entire URL must then be copied from the browser and pasted back into the terminal. After completing this step, the proxy obtains the OAuth token required to send emails. As a result, I successfully received the patch email in my USP inbox.

One thing that I really liked about this workflow is that tools like **git send-email** and **kw send-patch** greatly simplify the patch submission process. Without these tools, we would need to:

- manually format patch files
- carefully construct email messages
- ensure that code formatting is preserved

Using these tools ensures that the patches are properly formatted and safely sent, without leaving the local repository unnecessarily. After finishing this tutorial, I felt relieved to know that the patch formatting and sending process can be largely automated.

