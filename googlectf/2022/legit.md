# LeGit

**CTF:** GoogleCTF 2022  
**Category:** misc  
**Original writeup:** <https://galtashma.com/posts/googlectf-2022-legit-git-challenge-write-up> — by **galtashma.com**. All credit goes to the original author; this is an archived copy.

---

This year I joined my friend @omerye solving this challenge. We have a server exposing some basic capabilites and a main menu. It can clone a repo, list and print files in it. In this challenge we can see the source code and the target is to print the flag, which is stored at `/flag` which is inaccessible to us. We started it as a local service using docker and connected to it using netcat.

ctf/googlectf/legit ❯ nc localhost 1337 _ ____ _ _ | | ___ / ___(_) |_ | |/ _ \ | _| | __| | | __/ |_| | | |_ |_|\___|\____|_|\__| >>> Repo url: https://github.com/bitterbit/googlectf.git Repo cloned! Welcome! How can I help you? 1- List files in repository 2- Show file in repository 3- Check for updates 4- Pull updates 5- Exit

Openning up the code we can see there is a menu function that prints the prompt we just saw and a waterfall handling the different operations, looks simple.

if __name__ == '__main__': print(_BANNER) clone_repo() while True: print("cwd = ", os.getcwd()) option = menu() if option == 1: list_files() elif option == 2: show_file() elif option == 3: check_updates() elif option == 4: pull_updates() elif option == 5: print("kthxbye") break else: print("Invalid option :(")

My friend started searching for funky implemntations until he found a `list_files`. This function uses `os.chdir` to change the current working directory and of the program (man), but the author forget to "unwind" these changes and reset the current working directory to the project root, letting us have partial control over the current working directory.

What can we use this for?

Our first instinct was git hooks, what if we can execute a custom hook each time a git operation is done by this git client. Git hooks are configured in the `.git/config` file. What if there is a .git directory somewhere inside our project where we can control the config file?

Problem is, git config is local to each user and is not inherited from the git server. Pushing and pulling these files is prohibited by the client and server

It is not possible to automatically include your custom configuration file through git alone, because it creates a security vulnerability.

https://stackoverflow.com/questions/18329621/how-to-store-a-git-config-as-part-of-the-repository

To bypass this looked for different ways to include a custom git config.

Bare Repositories

A bare repository is the same as default, but no commits can be made in a bare repository. The changes made in projects cannot be tracked by a bare repository as it doesn’t have a working tree. A working tree is a directory in which all the project files/sub-directories reside. Bare repository is essentially a .git folder with a specific folder where all the project files reside.

https://www.geeksforgeeks.org/bare-repositories-in-git/

This part of the solution follows a nice article by Jake Edge, The risks of embedded bare repositories in Git.

Bare repositories are different than regular repositories, instead of storing the config file in the .git subdirectory of the repository, a bare repository stores these files directly in the directory where the repository is created.

$ cd tmp1; git init; ls .git/ FETCH_HEAD HEAD config description hooks info $ cd tmp2; git init --base; ls FETCH_HEAD HEAD config description hooks info

By default bare repositories don't have a "work tree", so running `git fetch` in `tmp2` would fail, but this can be easily fixed by configuring a work tree. Now we control a git config, we only have to configure it to execute something!

Git fetch hook

So far our respoistory looks like this:

googlectf-2022-legit/ my-bear-repo/ FETCH_HEAD HEAD config description hooks info worktree

Our plan is to make the server clone the `googlectf-2022-legit` repository and use command 1 to enter the `my-bear-repo` folder. Now we control the git config but we are still lacking the malicious pwn. Following Jake's article we learn about `fsmonitor` and it's wide use in git.

The idea behind fsmonitor is to reduce the search space for commands like git status by returning a list of files that may have changed since a given date and time. The directive can be set to a command to run that should return the list; if it returns a failure exit code

Using `fsmonitor` we can simply configure it to print the contents of `/flag` into a local file named `flag` which we can access.

fsmonitor = "cat /flag > flag; false"

Final Solution

Steps

Clone our crafted repository

Use command 1 to make server chdir into inner bare repository

Use command 3 to request git fetch, this will trigger our fsmonitor script script will copy contents of `/flag` into `my-bear-repo/worktree/flag`

Use command 2 to print the contents of our newly created file `my-bear-repo/worktree/flag`

Success!
