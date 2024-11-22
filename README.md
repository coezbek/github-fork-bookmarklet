# github-fork-bookmarklet

The Github network member fork page (`https://github.com/<user>/<repo>/network/members`) unfortunately doesn't show any information which forks are really active.

The following bookmarklet will retrieve this information using a call to `https://github.com/${user}/${repo}/branch-infobar/main` and update the list of forks. It will remove all forks which aren't ahead by at least one commit (because these are likely accidental forks).

```js
javascript:(async () => {
    try {
        /* Determine the main repository's default branch */
        const repoPath = window.location.pathname.split('/').slice(1, 3).join('/');
        const apiUrl = `https://api.github.com/repos/${repoPath}`;
        const mainRepoResponse = await fetch(apiUrl);
        if (!mainRepoResponse.ok) throw new Error('Failed to retrieve main repository information.');
        const mainRepoData = await mainRepoResponse.json();
        const defaultBranch = mainRepoData.default_branch;

        /* Collect all fork links excluding the original repo */
        const forkLinks = [...document.querySelectorAll('div.repo a:last-of-type')].slice(1);

        /* Function to fetch branch information */
        const getBranchInfo = async (user, repo) => {
            const branchInfoUrl = `https://github.com/${user}/${repo}/branch-infobar/${defaultBranch}`;
            try {
                const response = await fetch(branchInfoUrl, { headers: { accept: 'application/json' } });
                if (response.ok) {
                    const data = await response.json();
                    return data.refComparison;
                }
            } catch (error) {
                console.error(`Error fetching branch info for ${user}/${repo}:`, error);
            }
            return null;
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

# Adapted bookmarket for the Forks page

The new default page for forks (`https://github.com/<user>/<repo>/forks`) is a different structure. Handling both in a single bookmarklet would be too long. Thus a second version just for the forks pages:

```js
javascript:(async () => {
    try {
        /* Determine the main repository's default branch */
        const repoPath = window.location.pathname.split('/').slice(1, 3).join('/');
        const apiUrl = `https://api.github.com/repos/${repoPath}`;
        const mainRepoResponse = await fetch(apiUrl);
        if (!mainRepoResponse.ok) throw new Error('Failed to retrieve main repository information.');
        const mainRepoData = await mainRepoResponse.json();
        const defaultBranch = mainRepoData.default_branch;

        const fetchBranchInfo = async (user, repo) => {
            try {
                const res = await fetch(`https://github.com/${user}/${repo}/branch-infobar/${defaultBranch}`, {
                    headers: { accept: 'application/json' }
                });
                if (!res.ok) {
                    throw new Error(`Failed to fetch branch info for ${user}/${repo}`);
                }
                const data = await res.json();
                return data.refComparison;
            } catch (e) {
                console.error(`Error fetching branch info for ${user}/${repo}:`, e);
                return null;
            }
        };

        /* Select the list of forks on the Forks page */
        const forkList = document.querySelectorAll('ul[data-view-component="true"] > li');
        if (!forkList.length) {
            alert('No forks found on this page.');
            return;
        }

        /* Process each fork */
        for (const forkItem of forkList) {
            try {
                /* Get the second link (user/repo) from the list item */
                const repoLink = forkItem.querySelector('a[href*="/"]:nth-of-type(2)');
                if (!repoLink) continue;

                const match = repoLink.href.match(/github\.com\/([^/]+)\/([^/]+)/);
                if (!match) continue;

                const [, user, repo] = match;

                /* Fetch branch information */
                const branchInfo = await fetchBranchInfo(user, repo);
                if (branchInfo) {
                    const { ahead, behind } = branchInfo;
                    const infoText = `Ahead: <font color="#0c0">${ahead}</font>, Behind: <font color="red">${behind}</font>`;
                    
                    /* Create a div to display branch information */
                    const infoDiv = document.createElement('div');
                    infoDiv.innerHTML = infoText;
                    infoDiv.style.marginTop = '10px';
                    infoDiv.style.fontSize = 'small';

                    forkItem.appendChild(infoDiv);
                    if (ahead === 0) {
                        forkItem.style.display = 'none';
                    }
                }
            } catch (e) {
                console.error(`Error processing fork item:`, forkItem, e);
            }
        }
    } catch (e) {
        console.error('Error in bookmarklet execution:', e);
    }
})();
```

# Changelog

- 2024-11-22: Fix #1 - supports repos with master branch.
- 2024-11-22: initial release

# References

The idea for this bookmarket originated from this [stackoverflow discussion](https://stackoverflow.com/questions/54868988/how-to-determine-which-forks-on-github-are-ahead) and in particular [this solution by user 'root'](https://stackoverflow.com/a/68335748/278842), which broke when Github made the branch information async.
