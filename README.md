# github-fork-bookmarklet

The Github network member fork page (`https://github.com/<user>/<repo>/network/members`) unfortunately doesn't show any information which forks are really active.

The following bookmarklet will retrieve this information using a call to `https://github.com/${user}/${repo}/branch-infobar/main` and update the list of forks. It will remove all forks which aren't ahead by at least one commit (because these are likely accidental forks).

Caveats: 
 - This will take some time unforunately.
 - If you aren't on the `networks/members` page, the bookmarklet will take you to the right page, but then you need to click the bookmarklet AGAIN.

```js
javascript:(async () => {
    /* Function to fetch the default branch of a repository */
    const getDefaultBranch = async (user, repo) => {
        const response = await fetch(`https://api.github.com/repos/${user}/${repo}`);
        if (!response.ok) throw new Error('Failed to retrieve repository information.');
        return (await response.json()).default_branch;
    };

    /* Function to fetch branch information for a given user, repo, and branch */
    const getBranchInfo = async (user, repo, branch) => {
        try {
            const response = await fetch(`https://github.com/${user}/${repo}/branch-infobar/${branch}`, { headers: { accept: 'application/json' } });
            return response.ok ? (await response.json()).refComparison : null;
        } catch (error) {
            console.error(`Error fetching branch info for ${user}/${repo}:`, error);
            return null;
        }
    };

    try {
        /* Ensure the script runs on a GitHub repository page */
        const match = window.location.href.match(/^https:\/\/github\.com\/([^/]+)\/([^/]+)(\/network\/members\/?)?/);
        if (!match) {
            alert('Run this from a GitHub repository page.');
            return;
        }
        if (!match[3]) {
            window.location.href = `https://github.com/${match[1]}/${match[2]}/network/members`;
        }
        const [_, mainUser, mainRepo] = match;
        const defaultBranch = await getDefaultBranch(mainUser, mainRepo);

        /* Collect all fork links excluding the original repo */
        const forkLinks = [...document.querySelectorAll('div.repo a:last-of-type')].slice(1);

        for (const link of forkLinks) {
            try {
                /* Extract user and repository name from the link */
                const [_, user, repo] = link.href.match(/github\.com\/([^/]+)\/([^/]+)/) || [];
                if (!user || !repo) continue;

                /* Attempt to fetch branch information, with fallback to repo's default branch */
                let branchInfo = await getBranchInfo(user, repo, defaultBranch);
                if (!branchInfo) branchInfo = await getBranchInfo(user, repo, await getDefaultBranch(user, repo));

                /* Check downstream forks for modifications */
                const childLinks = [...link.closest('.repo').querySelectorAll('.network-tree + a')];
                const childrenHaveMods = await Promise.all(childLinks.map(async (childLink) => {
                    const [_, childUser, childRepo] = childLink.href.match(/github\.com\/([^/]+)\/([^/]+)/) || [];
                    if (childUser && childRepo) {
                        const childBranchInfo = await getBranchInfo(childUser, childRepo, defaultBranch);
                        return childBranchInfo && childBranchInfo.ahead > 0;
                    }
                    return false;
                })).then(results => results.some(Boolean));

                /* Hide or show branch details for forks based on modifications */
                if (branchInfo) {
                    const { ahead, behind } = branchInfo;
                    if (ahead === 0 && !childrenHaveMods) {
                        link.closest('.repo').style.display = 'none';
                    } else {
                        const branchDetails = `Ahead: <font color="#0c0">${ahead}</font>, Behind: <font color="red">${behind}</font>`;
                        link.insertAdjacentHTML('afterend', ` - ${branchDetails}`);
                    }
                }
            } catch (error) {
                console.error(`Error processing link: ${link.href}`, error);
            }
        }
    } catch (error) {
        console.error('Error in bookmarklet execution:', error);
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

Caveat: The forks page is paginated so you will only see a couple of entries at a time.

```js
javascript:(async () => {
    /* Ensure the script is run on a valid GitHub repository page */
    const ensureForksPage = () => {
        const match = window.location.href.match(/^https:\/\/github\.com\/([^/]+)\/([^/]+)(\/forks\/?)?/);
        if (!match) {
            alert('Run this from a GitHub repository page.');
            throw new Error('Not on a valid GitHub repository page.');
        }
        if (!match[3]) {
            window.location.href = `https://github.com/${match[1]}/${match[2]}/forks`;
            throw new Error('Redirecting to the forks page.');
        }
        return { user: match[1], repo: match[2] };
    };

    /* Fetch the default branch of a repository */
    const getDefaultBranch = async (user, repo) => {
        const res = await fetch(`https://api.github.com/repos/${user}/${repo}`);
        if (!res.ok) throw new Error(`Failed to fetch default branch for ${user}/${repo}`);
        return (await res.json()).default_branch;
    };

    /* Fetch branch information */
    const fetchBranchInfo = async (user, repo, branch) => {
        try {
            const res = await fetch(`https://github.com/${user}/${repo}/branch-infobar/${branch}`, { headers: { accept: 'application/json' } });
            if (res.ok) return (await res.json()).refComparison;
        } catch (e) {
            console.error(`Error fetching branch info for ${user}/${repo}:`, e);
        }
        return null;
    };

    try {
        /* Ensure on the forks page and get main repository details */
        const { user: mainUser, repo: mainRepo } = ensureForksPage();
        const defaultBranch = await getDefaultBranch(mainUser, mainRepo);

        /* Process fork links */
        const forks = [...document.querySelectorAll('ul[data-view-component="true"] > li')];
        for (const fork of forks) {
            const repoLink = fork.querySelector('a[href*="/"]:nth-of-type(2)');
            const match = repoLink?.href.match(/github\.com\/([^/]+)\/([^/]+)/);
            if (!match) continue;

            const [_, user, repo] = match;
            let branchInfo = await fetchBranchInfo(user, repo, defaultBranch);
            if (!branchInfo) branchInfo = await fetchBranchInfo(user, repo, await getDefaultBranch(user, repo));

            if (branchInfo) {
                const { ahead, behind } = branchInfo;
                const info = `Ahead: <font color="#0c0">${ahead}</font>, Behind: <font color="red">${behind}</font>`;
                const infoDiv = document.createElement('div');
                infoDiv.innerHTML = info;
                infoDiv.style.marginTop = '10px';
                infoDiv.style.fontSize = 'small';
                fork.appendChild(infoDiv);
                if (ahead === 0) fork.style.display = 'none';
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
