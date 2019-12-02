# appmeshworkshop

https://appmeshworkshop.com

This is how I set up my environment:
(I am using gitpod.io for editing)

1. fork the repo to your own github account
2. prepend `gitpod.io#` to the beginning of your github url. Mine becomes: `https://gitpod.io#github.com/brentley/appmeshworkshop`
3. once gitpod has started, in the terminal, run `npm install && npm run theme`
This will install the dependencies and clone the theme submodule.

From here, you can use the online IDE to edit /content/chapter/filename.md...
If you want to preview your edits, in the terminal, run:
`npx hugo server`.
That will start the local hugo server.

A dialog box will pop up telling you "A service is listening on port 1313, but is not
exposed." -- press the expose button. After that, choose "open browser" to get a new
tab with your preview site. As you save edits, this tab should refresh.

When you're happy with your edits, commit, push, and open a pull request to the upstream
repo's master branch. Once merged, the preview site (linked above) will be refreshed.

