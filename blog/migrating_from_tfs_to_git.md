There are a couple of tools to do this, however one appears more recently maintained: https://github.com/git-tfs/.

The steps in principle are fairly straight-forward, however there are a couple of useful things to note.

The executable can be called as git-tfs.exe, providing git's in your path. If you set it up properly, you're meant to be able to call `git tfs` as well, but I haven't bothered as I'm not intending to use it much and regularly move between machines.

The basic command is as follows:

    git-tfs clone http://tfs-server:8080/tfs/Repo $/Project.name

However, I had a couple of issues - firstly I stumbled across this bug: https://github.com/git-tfs/git-tfs/issues/845.

That's easily fixed by running it as below (setting an environment variable prior to running):

    MSYS_NO_PATHCONV=1 git-tfs clone http://tfs-server:8080/tfs/Repo $/Project.name

The second issue I had is that I was cloning a large project, and obviously doing so over the network - sometimes that worked, but sometimes it failed part way through. In order to provide some protection against this I started running the clone in batches, like this:

    MSYS_NO_PATHCONV=1 git-tfs clone --batch-size=100 http://tfs-server:8080/tfs/Repo $/Project.name

There is also a `--resumable` option, but I've not tried it - I've seen issues with subversion where everything except one commit was fine, which it's difficult to know until you find it. Given the nature of these repositories, I'm loathe to try any complex features such as stopping and resuming, unless absolutely necessary. This kind of thing also usually makes operations much slower.
