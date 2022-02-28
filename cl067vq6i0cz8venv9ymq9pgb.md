## Pwning a Server using Markdown

# Background

Hashnode is a blogging platform for developers where you can host your blogs for free with your custom domains. 
This is packed with features and one such feature is the "Bulk Markdown Importer". 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1644732483145/G0KYaMO56.png)

While I was working on migrating my blog from Jekyll to Hashnode, I was searching for an import feature. Fortunately, Hashnode has a Markdown importer that allows importing markdown posts in bulk but that needs to be in a certain format. 
For some reason, I kept getting errors when importing my posts. I couldn't figure it out because they were no descriptive errors on the UI. Then I looked at the response in my Burp and that's when I noticed the error that led me to write this blog. 

This was affected by LFI that allowed us to fetch internal files from the server. It was escalated to an RCE by finding the actual IP of the server behind Cloudflare by my mate [Adhyayan](https://twitter.com/nullvoiddeath). 

Here's how we escalated a vulnerable Markdown Parser to Code Execution on the server. 

---
# The Exploit

### Finding the LFI

Markdown has its own quirks and features to allow referencing images in the files. To include an image in the blog post, or any MD file, here's the common syntax:

`![image.png](https://image.url/image_file.png)`

The Bulk Importer at Hashnode accepts a ZIP file containing all the Markdown posts to be published. 
Here's how their sample post format looks:

```
---
title: "Why I use Hashnode"
date: "2020-02-20T22:37:25.509Z"
slug: "why-i-use-hashnode"
image: "Insert Image URL Here"
---

Lorem ipsum dolor sit amet
```

This `.md` file needed to be zipped into an archive to be uploaded to the platform. 
Here's how the response looks inside Burp Suite.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1644735982896/4SoCpzKr3.png)

Nothing interesting about it. It's just a normal Markdown parsed post format. 
This made us wonder about the Markdown quirk that allows a user to insert images by specifying their paths:

`![anotherimage.png](/images/blog.jpg)`

When observed in the Burp Suite, surprisingly, Hashnode triggered an `ENOENT` error saying that it was not able to find the file as can be seen in the below screenshot.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645974578557/5KijgHplj.png)

From here, it was just a matter of connecting the dots to fetch the internal files from the server. Instead of a non-existent path, we decided to give the location of an actual file like the `/etc/passwd` hoping it would give us the file contents in the response. 

Here's the markdown file we used as our final payload: 

```
---
title: "Why I use Hashnode"
date: "2020-02-20T22:37:25.509Z"
slug: "why-i-use-hashnode"
image: "Insert Image URL Here"
---

![notimage.png](../../../../../etc/passwd)
```

This time, instead of directly using the image as shown in the markdown body, the application tried to fetch the image using the location specified in the path. 
The application traversed the directories and fetched the `passwd` file for us but instead of the contents being displayed in the response, it uploaded the file to Hashnode CDN. 
![parrot.gif](https://emojis.slackmojis.com/emojis/images/1643509534/38083/wfh_parrot.gif)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645974007434/A70Ng2fvT.png)

The `contentMarkdown` parameter gave the CDN URL with the path on which the internal file was uploaded. We were able to directly download the file with the contents of the `/etc/passwd`. Neat.

Since we already had the name for the user and path of their home directory from the `passwd` file, we thought about escalating it a bit further trying for RCE. 

When you create an SSH key, it gets stored in a default location at `~/.ssh/id_rsa` for the private and `~/.ssh/id_rsa.pub` for the public key. 
We modified our payload accordingly to fetch the private key from the server and got lucky. It was uploaded to the CDN as well. 

`![notimage.png](../../../../../home/username/.ssh/id_rsa)`

Now, all we needed to get into the server was to find the IP address since it was hiding behind Cloudflare. 

---

### Server IP and SSH

We started looking for historical DNS records in order to find the IP Address but were unsuccessful. 
This is where [Adhyayan]() came through and we looked into the file `/proc/net/tcp`. 
These /proc interfaces provide information about currently active TCP connections.
Here's how it looks on my server:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645981912952/8tE6bWflI.png)

The [kernel.org](https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt) documentation explains the table pretty well. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645981773414/w5xemzdBl.png)

The column which we are interested in is the local address. These addresses are stored as hex values of the decimal notation of the reversed IP addresses. 
Here's a nifty one-liner I found on the internet to do all the work and return the IPs in a human-readable format. 

```
grep -v "rem_address" /proc/net/tcp | awk  '{x=strtonum("0x"substr($2,index($2,":")-2,2)); for (i=5; i>0; i-=2) x = x"."strtonum("0x"substr($2,i,2))}{print x":"strtonum("0x"substr($2,index($2,":")+1,4))}'
```

This effectively gave us what we were looking for - the server's IP address along with port 22. Here's how it looked:


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645982447552/B9jJ5rWZA.png)

# Conclusion
Who would have thought that Markdown parsers can lead to command executions on the server? 
Even the smallest of low severity issues can be escalated when chained with other vulnerabilities. 
Here, a simple information disclosure bug in the descriptive stack trace helped us figure out the behavior of the markdown parser which in turn allowed us to fetch internal files from the server. 

It's always a good idea to implement proper error handling in your code and log the descriptive errors in the backend. 
Hashnode team fixed the vulnerability in the Markdown parser and rotated all their private keys to remediate the bug. 

it's always a bad idea to trust your users' input!

> **Links for reference**

> Adhyayan Panwar's Twitter: https://twitter.com/nullvoiddeath

> Kernel.org Documentation: https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt 

> One-liner for reading the `/proc/net/tcp`: https://gist.github.com/staaldraad/4c4c80800ce15b6bef1c1186eaa8da9f

