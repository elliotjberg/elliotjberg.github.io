To migrate a subversion folder into Git, follow these steps:


* Create a users file for mapping SVN users to Git:
 * Get a list of users: `svn log -�xml | grep author | sort | uniq`

 * Add them to a text file in this format:

    user1 = FirstName LastName <email@address.com>

    user2 = FirstName LastName <email@address.com>

* Clone the repository: `git svn clone --stdlayout --no-metadata --authors-file=users.txt https://svnserver/svn/folder`

* Convert the SVN tags and branches to local Git tags and branches, using these bash one-liners:
 * SVN branches:

    for branch in `git branch -r | grep "branches/" | sed 's/ branches\///'`; do  git branch $branch refs/remotes/$branch; done
 
 * SVN tags:

    for tag in `git branch -r | grep "tags/" | sed 's/ tags\///'`; do  git tag -a -m "Converting SVN tags" $tag refs/remotes/$tag; done

* Set the new remote: `git remote add origin https://user@gitserver/project.git`

* Push to remote repository:
 * if there are tags/branches: `git push origin --mirror`
 * if there aren't: `git push origin master`
 
