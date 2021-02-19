---
layout: post
title: Working with raw logging from Logstash and journalctl
tags: [Troubleshooting, Linux, Java, Logstash]
---

The other day I was troubleshooting an application I contribute to that is used as a Linux startup service managed by systemd. I wanted to find logging items that matched either on their message or on their metadata.

The application is JVM based and as is often the case uses Logstash and Logback to produce logs in a json format to provide metadata in addition to the message passed to the logger.

_**So what was my problem, how come I didn't just use the web ui?**_

The system I was working on doesn't have the ELK stack installed so I didn't have the typical and friendly tools to search through my logs. However, I did have access to the host running the application.

First things first. Given that I have access to the host I should be able to access the output from the application. I know that I can use journalctl to observe what the application is writing to standard out. I also know that the tool `jq` can be used to filter and transform json. Using these two command line utilities together I'll show you how to effectively find what you are looking in json formatted logstash output.

## Help 1
All the logstash encoder messages use `@timestamp`, so use `grep` to filter output from the process's stdout not produced by logging. Then use `jq` to start looking for the desired data.

Here's a quick way to see the raw logging messages since the most recent "boot" of the service

```bash
journalctl --boot=0 -o cat -u foobar.service | grep '@timestamp' | jq -M '.message'
```

Now you can start getting at what you want in those messages. Let's find messages with `lizard` or `turtle`.

```bash
journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '.message' | grep -i 'lizard\|turtle'
```

Don't forget that you can filter out repetitive logs with `uniq`.

```bash
journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '.message' | uniq | grep -i 'lizard\|turtle'
```

It's important that uniq goes after jq because each raw message will have unique timestamps and you will defeat the purpose of getting only the unique part of the individual message field

## Help 2
What if other fields besides message could contain data about lizards and turtles, how do we find them?

Suppose some of the logging messages produced have a json encoded format like the following.

```json
{
  "foo": {
    "lizard": "Reggie 4.0"
    "bar": {
      "lizard": "Scaly Beast that ate my sandwich"
      }
    }
  }
}
```

Will will use the `(.. | .lizard?)` operation with jq to find any logging output that contains a `lizard` field anywhere in the json hiearchy.

```bash
journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '(.. | .lizard?)' | grep -v null
```

The one caveat is I'm not yet sure how to avoid jq returning null for all the messages where there wasn't a match. That's what the trailing `| grep -v null` is for above.

### References
[journalctl man page](https://www.commandlinux.com/man-page/man1/journalctl.1.html)
[jq json query](https://stedolan.github.io/jq/)
[Logstash](https://www.elastic.co/logstash)
