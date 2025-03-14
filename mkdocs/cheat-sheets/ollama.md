---
date:
  created: 2025-02-02
  updated: 2025-02-02
draft: false
tags:
  - ollama
title: Ollama
icon: simple/ollama
---
# Ollama
Ollama is a lightweight, open-source framework for running and managing large language models (LLMs) locally.

These are my cheat sheets to use it for everyday tasks!

## Commit messages
Note that I use Mac and Zsh and this build is for Mac and Zsh.

In my case I have an extra bin folder at the user level eg ~/bin, you probably have to adjust it to run from somewhere else
```bash linenums="1"
#!/usr/bin/env zsh

DIFF=$(git diff --cached $1)

SUMMARY=$(
  ollama run llama3.2 <<EOF
Generate a raw text commit message for the following diff.
Keep commit message concise and to the point.
Make the first line the title (100 characters max) and the rest the body:
Do not show raw output of Diff in the message
Do not add meta data such as characters
$DIFF
EOF
)

STRIPPED_SUMMARY=$( echo $SUMMARY | sed -e 's|<think>.*</think>||g' )

echo -n $STRIPPED_SUMMARY

echo -n " [Use result for commit, rerun or abort y/r/n] "
read DECISION

if [[ $DECISION == [rR] ]]; then
  exec ~/bin/oc.sh
fi

if [[ $DECISION == [yY] ]]; then
  git commit -m "$STRIPPED_SUMMARY"
fi

if [[ $DECISION == [nN] ]]; then
  echo "Bye not commiting anything"
fi
```