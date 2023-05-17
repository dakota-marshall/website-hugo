---
title: "Work Log Files"
date: 2021-08-31T23:44:40-04:00
draft: true
toc: false
images:
tags:
  - Asterisk
  - Python
  - SQL

---

## Summary

At work there are often times where a client or myself will need a report of when and which phones disconnected from our phone system. These logs are traced in our logging system, but the info the log is only relevant internally. I took it upon myself to write a script that will take our log files, and insert the relevant info into the lines to make them more usable for my T1 techs and our clients IT companies. It ended up becoming a lot bigger than I expected it to be so I figured it was worth writing down my thought-process at each step of the way and walking through what it does, as I am pretty happy with the results.

The script functions as follows:

- Get a list of extensions and names on the system (if live).
- Generate list of peers and their associated IP, either via file or directly from Asterisk if live.
- Grab the log file, either via file or directly from Asterisk if live
- Re-write the file with the newly grabbed info, insterting the info as necessary.

Its fairly simple and straight forward, but the biggest challenge came in trying to get the info live off of the system instead of presenting the script curated files.

Overall its definitely the most feature packed Python script I've written and am definitely proud of the way this turned out. If you are interested in reading how it works or the issues I ran into, check out below.

## How it works

The script starts by reading switches passed to it and sets the appropriate variables via this block of code:

```python
argv = sys.argv[1:]
try:
    options, arguments = getopt.getopt(argv, "i:o:p:L:h:e")
except getopt.GetoptError:
    print(help_output)
    sys.exit((2))

for option, argument in options:
    if option == '-i': #input file
        log_file_location = argument
    elif option == '-o': #output file
        fixed_log_location = argument
    elif option == '-p': #peer file
        peer_file_location = argument
    elif option == '-L': #Live mode and number of logs
        live_mode = True
        num_logs = int(argument)
    elif option == '-e':
        extension_mode = True
    elif option == '-h':
        print(help_output)
        exit()
```

This was shamelessly ripped off a Stack Overflow article, so I take no credit for it. It works flawlessly however, so its what I used. 

### Getting the Extensions

Next the script has to get the list of extensions and names. The easiest method to do this is just providing it a file, but this isnt as *auto-magic* as I would like it to be, so I worked on a way to be able to do this live on the server.


## Challenges I ran into