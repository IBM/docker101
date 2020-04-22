**NOTA BENE: this repo has MOVED to IBM/docker101**

# Docker

Series of labs and instructions to introduce you to containers and Docker. Learn to run a container, inspect a container and understand the isolation of processes, create a Dockerfile, build an image from a Dockerfile and understand layers, tag and push images to a registry, scale and update containers, and more.

To view the Docker workshop online, go to:
* <https://ibm-developer.gitbook.io/docker/>.

To view the Docker workshop in Github, go to:
* [workshop/README.md](workshop/README.md).

This repository has the following structure:
```ini
- workshop (workshop labs)
|_ .gitbook (images)
|_ <language> (localization support) 
  |_ <folder-n> (workshop labs)
    |_README.md (steps for labs in Markdown)
  |_ README.md (gitbook home page)
  |_ SUMMARY.md (table of contents)
.gitbook.yaml (GitBook read-only instructions)
.travis.yaml (runs markdownlint by default)
README.md (GitHub.com README)
```

## Markdown lint tool

Install the [Markdown lint tool](https://github.com/markdownlint/markdownlint),
```
$ npm install -g markdownlint-cli
```

To use markdownlint, run the following command,
```
$ markdownlint workshop -c ".markdownlint.json" -o mdl-results.md
```

## Build Gitbook 

Install the [gitbook-cli](https://github.com/GitbookIO/gitbook-cli),
```
$ npm install -g gitbook-cli
```

To build the Gitbook files into the `_book` sub-directory with the `gitbook-cli`, run the following command,
```
$ gitbook build ./workshop
```

Serve the Gitbook files locally with the following command,
```
$ gitbook serve ./workshop
```



