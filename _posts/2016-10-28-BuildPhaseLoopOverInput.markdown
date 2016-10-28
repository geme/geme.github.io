---
layout: post
title:  "Loop over input files in a script in Build Phases"
date:   2016-10-27 12:00:00 +0100
categories: xcode
description: "In my Build Phases I had a custom script which should loop over every input. I am not so familar with bash scripting so there was a little bit of googling and try and error, here I want to share what I ended up using."
author: "@gerritmenzel"
---

Navigate to `Target -> Build Phases` on the top left is a plus sign to add a new Build Phase "New Run Script Phase".

Here you can enter the custom script to run. If you want to access the input or output files you can define use:
- `${SCRIPT_INPUT_FILE_0}`
- `${SCRIPT_INPUT_FILE_1}`
- `${SCRIPT_INPUT_FILE_n}` 
- `${SCRIPT_INPUT_FILE_COUNT}`
- `${SCRIPT_OUTPUT_FILE_0}`
- `${SCRIPT_OUTPUT_FILE_1}`
- `${SCRIPT_OUTPUT_FILE_n}` 
- `${SCRIPT_OUTPUT_FILE_COUNT}`

So to iterate over all input file i used:

``` bash
COUNTER=0
while [$COUNTER -lt ${SCRIPT_INPUT_FILE_COUNT}]; do
    tmp="SCRIPT_INPUT_FILE_$COUNTER"
    FILE=${!tmp}

    echo $FILE

    let COUNTER=COUNTER+1
done

```

same goes for output files.

