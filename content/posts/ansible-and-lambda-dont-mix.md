---
title: "Ansible and Lambda Don't Mix"
date: 2022-02-17T11:10:54-05:00
---

TL;DR - Don't attempt to run Ansible scripts within AWS Lambda; it won't work.

I recently needed to run some event-driven Ansible tasks. The workflow even lent itself to an event that could easily trigger an AWS lambda function. I spent a day or two whittling away at a Docker image for the lambda to do exactly what I wanted. I even tested it locally using a lambda emulator. I worked to setup all the directories used by Ansible to only write to the `/tmp` dir (which is all you can do in Lambda).

And then in production, it didn't work. After a little more time debugging I found the underlying issue. Ansible relies on access to `/dev/shm` for shared-memory coordination. Lambda doesn't allow it. That's it; end of story. You can check the references below for the nail-in-the-coffin on why this just ain't gonna happen.

## References

* [running ansible as simple user fails - (ansible github.com issues)](https://github.com/ansible/ansible/issues/13830)
* [Please provide /dev/shm access inside Lambda - (AWS support forums)](https://forums.aws.amazon.com/thread.jspa?threadID=219962)
