---
layout: post
title: Working with raw logging from Logstash
img: wall-of-text.png
tags: [Troubleshooting, Linux, Java, Logstash, journalctl, jq]
---

## Debugging JVM services without the [Elk stack](https://www.elastic.co/what-is/elk-stack)

The other day I was troubleshooting an application I contribute to written in a JVM based language that uses logstash encoded logging messages. If you're not familiar with Logstash, it is common to have it encode messages as JSON. There will be a JSON field `message` with the actual message plus other metadata stored in sibling JSON fields.

I wanted to find logging items that matched either by the pattern of a metadata field name or within the value of the actual messages.

_**So what was my problem, how come I didn't just use Kibana?**_

The system I was working on doesn't have the [ELK stack](https://www.elastic.co/what-is/elk-stack) installed so I didn't have the typical and friendly tools to search through my logs. 
In this case it was additionally undesirable to go about installing the ELK stack.

However, I did have access to the host running the application.

First things first. Given that I have access to the host I should be able to access the output from the application. I also know that the command line tool [jq](https://stedolan.github.io/jq/) can be used to filter and transform JSON.

The following helps are assuming that the service is controlled by systemd. With systemd you use the `journalctl` command line tool to access what services are writing to stdout. Still much of what follows is also applicable with `docker logs`, `kubectl logs`, or other platforms that will provide you with the raw logging output. The important piece is how to use any of those with [jq]() to parse through the JSON encoded logging.

Using `journalctl` and `jq` together with a sprinkling of `grep` I'll show you how to effectively find what you are looking in json formatted logstash output.

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
