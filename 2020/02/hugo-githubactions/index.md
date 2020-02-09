# Automate your GitHub Pages Deployment using Hugo and Actions


<img class="cp t u fz ak" src="/images/posts/hugo-actions/hugo-actions.png" width="3921" height="2185" role="presentation"/>

I am a fan of [Hugo](https://gohugo.io/) for building static websites and blogs. It is very easy to set up Hugo and start building webpages using markdown. If you want to add some serious customizations to your pages you can achieve that using [shortcodes](https://gohugo.io/content-management/shortcodes/). You have access to thousands of [themes](https://themes.gohugo.io/) which makes bootstrapping easy.

In this article, I am going to show how you can automate the build and deployment of GitHub Pages using Hugo and [GitHub Actions](https://github.com/features/actions). Creating a [Hugo website](https://gohugo.io/getting-started/quick-start/) and setting up a [GitHub page](https://pages.github.com/) is out of scope for this article.

For building and publishing the website, I am going to use [actions-gh-pages](https://github.com/peaceiris/actions-gh-pages).

There are two different ways to set up the deployment process.

Use Single Repository
---------------------

In this case, you are keeping the Hugo source code and generated Hugo site in the same repository configured for GitHub pages. The _Hugo_ _source_ contains all the files that written (or updated) by you for generating the site and _generated Hugo site_ is the contents inside `public` directory. In order to do this, you will have a dedicated branch in your repository (example `develop` ) contains the source and you will be storing the `public` directory contents in a different branch. Please note, in the case of personal repositories you have to keep the GitHub page files in `master` branch.

**Pros:** Less maintenance, as you can manage everything from a single repository.

**Cons:** This means you will have to manage entirely different contents between your source branch and the GitHub pages branch.

Use Multiple Repositories
-------------------------

In this case, you are keeping the _Hugo source_ in a dedicated repository and _generated Hugo site_ contents inside `public` directory in a different repository configured with GitHub pages.

**Pros:** Much cleaner approach and align well with the git branching strategies.

**Cons:** Additional work to set up and configure two repositories.

I am going to use the _multiple repositories_ set up with personal GitHub pages in this article.

Prerequisites:
--------------

*   Create and configure a repository for [GitHub page](https://pages.github.com/)
*   Create an additional repository for Hugo source files.
*   Build a sample Hugo site. For testing the set up you can use a sample Hugo site like [this](https://github.com/matcornic/hugo-theme-learn) (check the `exampleSite` )
*   Test your Hugo site using `hugo server -D`

Configure GitHub Actions
------------------------

You need to set up keys on the above repositories to enable publish.

1.  Generate your deploy key with the following command.

```
ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f master -N ""
\# You will get 2 files:
\#   master.pub (public key)
\#   master     (private key)
```

2\. Add your _private key_ as a _secret_ with name  ACTIONS\_DEPLOY\_KEY in the Hugo Source Repository.

Navigate to _Your Hugo Source Repository > Settings > Secrets > Add new secrets_ and the name the secret as ACTIONS\_DEPLOY\_KEY and paste the contents of `master` file in the value.

3\. Add your _public key_ as _deployment key_ in the GitHub pages repository

Navigate to _Your GitHub pages Repository > Settings > Deploy Key> Add new deploy key_ and set title something you can relate to the private key (like Public Key for my site deploy) and paste the contents of `master.pub` file in the value.

4\. Add below file `.github/workflows/pages.yml` location of your Hugo source code and replace _username/external-repository_ with your GitHub username and GitHub pages repository name.

5\. Commit your Hugo source code repository changes and push to GitHub. If you are using any branch other than `master` , you need to merge the changes to master for the workflow to start. You can change that setting in _on:push:branches_ part of the workflow file.

6\. Check the workflow in the Actions tab of your Hugo Source repository.

Once your workflow completed, you can see new commits on the GitHub pages repository `master` branch and in few minutes the changes will appear on your GitHub page URL.

