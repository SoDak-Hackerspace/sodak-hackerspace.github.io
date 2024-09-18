# Overview
Welcome to the offical website of the SoDak Hackerspace! This repository holds the website for https://sodakhacker.space and is built by github actions. To get started contributing and adding your profile and projects to the space, simply clone the repository using the following command:

`git clone --recurse-submodules <repo-url>`

The `--recurse-submodules` flag is necessary for the submodule located at `themes/hello-friend-ng` so that the theming doesn't break at local testing. To spin up a local instance of the site and make changes, simply run the following command at the root of the repository:

`hugo server` 

