# github-fork-bookmarklet

The Github fork page (`https://github.com/<user>/<repo>/network/members`) unfortunately doesn't show any information which forks are really active.

The following bookmarklet will retrieve this information using a call to `https://github.com/${user}/${repo}/branch-infobar/main` and update the list of forks. It will remove all forks which aren't ahead by at least one commit (because these are likely accidental forks).

```js
javascript:(async () => {
    try {
        /* Collect all fork links excluding the original repo */
        const forkLinks = [...document.querySelectorAll('div.repo a:last-of-type')].slice(1);

        /* Function to fetch branch information */
        const getBranchInfo = async (user, repo) => {
            const branchInfoUrl = `https://github.com/${user}/${repo}/branch-infobar/main`;
            try {
                const response = await fetch(branchInfoUrl, { headers: { accept: 'application/json' } });
                if (!response.ok) throw new Error(`Failed to fetch branch info for ${user}/${repo}`);
                const data = await response.json();
                return data.refComparison;
            } catch (error) {
                console.error(`Error fetching branch info for ${user}/${repo}:`, error);
                return null;
            }
        };

        for (const link of forkLinks) {
            try {
                /* Extract user and repository name from the link */
                const [_, user, repo] = link.href.match(/github\.com\/([^/]+)\/([^/]+)/) || [];
                if (!user || !repo) continue;

                /* Fetch branch information for the current fork */
                const branchInfo = await getBranchInfo(user, repo);

                /* Check downstream forks for modifications */
                const childLinks = [...link.closest('.repo').querySelectorAll('.network-tree + a')];
                let childrenHaveMods = false;
                for (const childLink of childLinks) {
                    const [_, childUser, childRepo] = childLink.href.match(/github\.com\/([^/]+)\/([^/]+)/) || [];
                    if (childUser && childRepo) {
                        const childBranchInfo = await getBranchInfo(childUser, childRepo);
                        if (childBranchInfo && childBranchInfo.ahead > 0) {
                            childrenHaveMods = true;
                            break;
                        }
                    }
                }

                /* Process display and information rendering */
                if (branchInfo) {
                    const { ahead, behind } = branchInfo;

                    /* Hide forks with no modifications and no downstream modifications */
                    if (ahead === 0 && !childrenHaveMods) {
                        link.closest('.repo').style.display = 'none';
                    } else {
                        /* Show branch details for forks with modifications */
                        const branchDetails = `Ahead: <font color="#0c0">${ahead}</font>, Behind: <font color="red">${behind}</font>`;
                        link.insertAdjacentHTML('afterend', ` - ${branchDetails}`);
                    }
                }
            } catch (error) {
                console.error(`Error processing link: ${link.href}`, error);
            }
        }
    } catch (globalError) {
        console.error('Error in bookmarklet execution:', globalError);
    }
})();
```

# Installation / Usage

1. Make a new bookmark in your browser (right-click on the bookmarks bar and click Add Page...)
2. Assign a useful name e.g. "Github Forks Bookmarklet"
3. Paste the Javascript above as the URL
4. Visit the network members page of a repo  (click on "Forks" in the right bar, then click on "Switch to tree view") and click the bookmark.

# References

The idea for this bookmarket originated from this [stackoverflow discussion](https://stackoverflow.com/questions/54868988/how-to-determine-which-forks-on-github-are-ahead) and in particular [this solution by user 'root'](https://stackoverflow.com/a/68335748/278842), which broke when Github made the branch information async.
