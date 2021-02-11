# Working with raw logging and journalctl

I have a JVM service using the logstash encoder, it is outputting json formatted messages and they can be observed with
journalctl, here are some notes on how to better extract desired information from them

All the logstash encoder messages use `@timestamp` so use grep to filter output from the processe's stdout
not produced by logging.

Then use jq to start looking for the desired data.

Here's a quick way to see the raw logging messages since the most recent "boot" of the service

`journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '.message'`

Now you can start getting at what you want in those messages.

Let's find messages with password or secret
`journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '.message' | grep -i 'secret\|password`

Don't forget that you can filter out repetitive logs with uniq

`journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '.message' | uniq | grep -i 'secret\|password`

It's important that uniq goes after jq because each raw message will have unique timestamps and you will defeat the purpose=
of getting only the unique part of the individual message field

Now since logstash encoding provides other data in addition to just the message let's look at that.
Again say that perhaps there may be other fields besides message that could contain secrets, how do we find them?
Assuming that inside the logstash encoding we have a message like I want to get all the json "secret" properties

```
{
  ...
  "foo": {
    "secret": "first secret"
    "bar": {
      "secret": "second secret"
      }
    }
  }
}
```

Here we use the `(.. | .secret?)' operation with jq, note that secret is just the name of the field I care about

`journalctl --boot=0 -o cat -u foobar.servcie | grep '@timestamp' | jq -M '(.. | .secret?)' | grep -v null

I'm not yet sure how to avoid jq returning null for all the messages where there wasn't a match.

https://www.commandlinux.com/man-page/man1/journalctl.1.html

https://stedolan.github.io/jq/
