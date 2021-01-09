title: Building Papers using Pandoc and Github Actions
date: 2020-06-24T22:10:37+01:00
tags: Teaching, Github, Pandoc, Docker
Summary: >
	Thoughts on using Github actions to build papers. 

## Getting Github Actions to Work

As always, there was a lot of trial and error (and a bit of Cargo-Cult
programming) while I worked out what was happening.

A lot of this stems from how Git deals with the data in your repo.  I
had assumed that actions would have access to the files in your repo,
but this is not the case.  Instead it appears that each build runs on
a "clean" install.  So we need to make sure it has the data available
by cloning the repo.  This (eventually) makes sense, as it follows a
standard Git workflow. The same goes for writing data back, we can
opeate on files, but they dont get added back to the repository, so we
need to commit and push them back

> Additional Note: There was also the added fun of getting the correct
> dependencies into the Docker Image.  I need the python-filters
> extension.  So had to build a custom version of pandoc/latex
> containing this and host it on dockerhub.

~~~.yaml
# Trying Workflows and Actions
name: Compile with Pandoc

on:
  pull_request:
    types: [closed]

jobs:
  convert_via_pandoc:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
        with:
          persist-credentials: false # otherwise, the token used is the GITHUB_TOKEN, instead of your personal token
          fetch-depth: 0 # otherwise, you will failed to push refs to dest repo
      - run: mkdir output
      - name: Build Documents
        uses: docker://dang42/pandoc-ieee
        with:
          args: "--filter=pandoc-citeproc --filter=templates/tablefilter.py --template templates/ieee.latex -o output/output.pdf ExamplePaper.md"
      - name: Commit Files #Save the files
        run: |
          git config --local user.name "djgoldsmith"
          git add ./output
          git commit -m "Commit Built paper"
      - name: Push Changes
        uses: ad-m/github-push-action@master
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
~~~


## Future Work

There are a few things I need to polish. In an approximate order.

 1. Like any open source project, the documentation is currently
    terrible.  Now I know how things work, I will get that resolved
    ASAP.

 1. Give something back, and add my examples to the Official documentation.

 1. Move the dissertaion template across so it works with this process
 
 1. Literate programming.



