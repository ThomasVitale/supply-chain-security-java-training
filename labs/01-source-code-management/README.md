# Source Code Management

## Learning Goals

* Signing Git commits
* Verifying Git commits

## Overview

In this exercise, you will see how to configure your Git repository to sign your commits and how you can verify the signatures to make the Git audit trail trustworthy.

## Details

### Signing commits with SSH Keys

GitHub actively supports verifying commits signed with GPG, SSH, or S/MIME. In this excercise, you will generate an SSH key and use it to sign Git commits.

* Generate a new SSH key by following the GitHub [instructions](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key) for your operating system. You will only need to follow the steps in the "Generating a new SSH key" section.
* Next, GitHub needs to know your public key in order to trust commits signed with your newly generated SSH key. Follow the [instructions](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account?tool=webui#adding-a-new-ssh-key-to-your-account) to add the SSH key to your GitHub account.
* Finally, Git also needs to know about your SSH key. From the root folder of this repository, open a Terminal window and run the following commands to configure your copy of this Git repository to sign commits with your SSH key.

```shell
git config gpg.format ssh
git config user.signingkey /PATH/TO/.SSH/KEY.PUB
```

You're all set to start signing commits. Make a change to your copy of this repository. For example, why not adding a new file into the current folder?

```shell
echo "This is real! This is me!" >> star.txt
```

Now you can commit the change and sign it.

```shell
git add star.txt
git commit -S -m "Do you trust me?"
git push
```

Go to GitHub and check the commit history for your copy of this repository. If everything worked fine, you'll find a "Verified" green badge next to your last commit, meaning that GitHub verified the signature on the commit.

Before moving on, you might want to delete the SSH key you used in this excercise. If you're working in a containerized development environment, your private key will be destroyed once you delete the environment. Still, you might also want to remove it from your GitHub account.

Go to `Settings > Access > SSH and GPG keys` from your GitHub account. You'll find the SSH key you created earlier listed there. Go ahead and delete it if you don't want to use it anymore.

### Signing commits with Sigstore Gitsign

An alternative approach for signing commits is based on a keyless strategy. Currently, GitHub doesn't support verifying such signatures, but it's an option worth exploring since it provides a better developer experience and higher security level.

* Follow the [instructions](https://docs.sigstore.dev/signing/gitsign/#quick-start) to install Sigstore GitSign for your operating system.
* Git needs to know about your preferred strategy for signing commits. From the root folder of this repository, open a Terminal window and run the following commands to configure your copy of this Git repository to sign commits with Sigstore Gitsign.

```shell
git config --local commit.gpgsign true  # Sign all commits
git config --local tag.gpgsign true  # Sign all tags
git config --local gpg.x509.program gitsign  # Use Gitsign for signing
git config --local gpg.format x509  # Gitsign expects x509 args
```

You're all set to start signing commits. Make a change to your copy of this repository. For example, why not adding another file into the current folder?

```shell
echo "Look! No keys!" >> awesome.txt
```

Now you can commit the change and sign it.

```shell
git add awesome.txt
git commit -S -m "Trust me, I'm keyless!"
git push
```

When you try to commit a change, you'll be prompt to authenticate from a browser window using one of the available Single-Sign On approaches, including GitHub, Microsoft, and Google.

Go to GitHub and check the commit history for your copy of this repository. If everything worked fine, you'll find an "Unverified" orange badge next to your last commit, meaning that GitHub noticed a signature but wasn't able to verify it. If you click on the badge, you can see that Sigstore, under the hood, issued the certificate related to the signature. Let's hope GitHub will introduce support for it soon!

You can verify the signature via Sigstore.

```shell
git verify-commit HEAD
```
