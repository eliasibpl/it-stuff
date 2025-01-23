# Github
First we have to add our remote repository, into a new directory we run these 2 commands, one to initiate git and then add our repo with **origin** as its name.
```
git init
git remote add origin git@github:<name of the repo>
```
Check your repositories with this command
```
git remote -v
```

Now we check if we have the permissions to push
```
git push -u origin main
```

If we get this error we have to add our **ssh key** to our trusted keys in **Github**
![Screenshot_20241114_121505](https://hackmd.io/_uploads/rkRToLXGke.png)

First if we don't have one we'll have to create an **RSA key** associated with our email. To do this we can simply run this command
```
ssh-keygen -t rsa -b 4096 -C "eibanezpl@gmail.com"
```
The `-t` flag specifies with kind of key, in this case **rsa**, `-b` flag indicates que bits to the key **4096** being strong, `-C` to comment the mail associated with the key.

In your **Github Account** go to `Settings` 
![Screenshot_20241114_121848](https://hackmd.io/_uploads/Hyy2hIXM1e.png)

Head to `SSH and GPG Keys`
![Screenshot_20241114_121927](https://hackmd.io/_uploads/BJvAnI7GJl.png)

Click `New SSH Key`
![Screenshot_20241114_122018](https://hackmd.io/_uploads/ryxsaUQM1g.png)

And now you have to paste the content of your `id_rsa.pub` key. you can do so by running this command unless you changed it's default name.
:warning: Never share your `PRIVATE key` (`id_rsa` or any kind of name you gave it) only use your `public key` (with the `.pub` extension) :warning: 
```
cat ~/.ssh/id_rsa.pub
```

Copy the contents right here, give it a title and hit `Add SSH Key`
![Screenshot_20241114_122351](https://hackmd.io/_uploads/rkiA6IXzye.png)

After the **ssh key** has been added now we can start working with our repository

First `fetch` the remote
```
git fetch <ssh of the repo>
```

Now we have to create some content like a simple fingerprint check command:
```
echo "ssh-keygen -lf ~/.ssh/id_rsa.pub" > fingerprint-ssh.sh
```

With our new bash file we have to `add` it to the `commit`, comment it and `push` it
```
git add fingerprint-ssh.sh 
git commit -m "fingerprint check bash"
git push -u origin main
```

We could simply `push` every file we have created with a `.` after the `add`, but we're choosing to only add our **bash file**, moreover we could `commit` without the `-m` flag however it's highly frowned upon to commit without **commenting** about the changes. We're pushing with the `-u` flag to the `origin` repo being the name we gave it in the beginning (instead of the long ssh link) to the `main branch`, if we're working in a team we have to learn about `forking` before committing such changes.

We can also delete the files locally using the `rm` command, if we want to remove it from the repo we need to use this one instead
```
# For a single file
git rm <file>

# To delete a whole directory
git rm -r path/to/directory
```

Now if we're working in a team and somebody `pushed` another commit we could simply `pull` the new files
```
git pull
git log
```
![Screenshot_20241114_132126](https://hackmd.io/_uploads/rJKLiD7zkx.png)

By modifying the new files we can simulate our workflow.
```
echo 'echo "collaborating to the project"' >> test.sh && cat test.sh
```

It'll display we have added a new line to our `test.sh`, we can check it's status running the following command
```
git status
```
![Screenshot_20241114_132603](https://hackmd.io/_uploads/B1ydnvQMJx.png)

We'll add the new changes using the flag `-u` to the `add` command
```
git add -u
git status
```
![Screenshot_20241114_132751](https://hackmd.io/_uploads/HkDR2wXGyx.png)

It's time to `commit` these changes and `push`, it'll push to the **working repository** on so it'll automatically set the `-u origin main` to it
```
git commit -m "Another line for the test"
git push
git log
```
We can confirm these changes happenned with the `log` command
![Screenshot_20241114_133210](https://hackmd.io/_uploads/S1eyRDQMkg.png)

If we mess up and want to clone our repository once again we're going to use the `clone` command
```
git clone <ssh or https-link> <name>
git log
```

Now we can keep working on our repo, if we want to "turn back in time" after every `commit` we can see an `commit-hash`:
![Screenshot_20241114_134435](https://hackmd.io/_uploads/rkI-WO7zkg.png)

If we use the `checkout` command we can add the `commit-hash` from the `commit`
```
git checkout 88bf60a9253cc716c7fee9754e801ca54a222cbc
```
![Screenshot_20241114_134751](https://hackmd.io/_uploads/B1ItbdmGkl.png)
![Screenshot_20241114_135012](https://hackmd.io/_uploads/Hy4Gz_7fJe.png)

In this commit we don't see the `clone-pull` change
![Screenshot_20241114_135103](https://hackmd.io/_uploads/r1OHf_mfJg.png)

With the `git branch` command we can see that we're in a different branch (detached)
![Screenshot_20241114_135229](https://hackmd.io/_uploads/Hkk3z_7M1g.png)

We're branching off `main` while in `checkout` with this command
```
git switch -c <branch-name>
```
![Screenshot_20241114_140240](https://hackmd.io/_uploads/HklISdXfkl.png)

We can also create a branch directly or start a branch from another commit using these commands
```
# Creates a branch named staging
git branch staging

# Creates a branch from an existing commit
git branch <commit-hash>

# Create a branch linked to another one, in this case it's a local one called new/feature from origin's new/feature
git branch --track new/feature origin/new/feature
# Switches to an existing branch
git switch <branch-name>

# Rename a branch we're working on
git branch -m <new-branch-name>

# Rename another branch
git branch -m <other-branch> <new-branch-name>
```
You can always check your `commit-hash` from the last commits published
![Screenshot_20241114_141947](https://hackmd.io/_uploads/SJ0GY_7M1x.png)

Now we can add a new file to work on
```
echo "df -H" > disk-check.sh && chmod +x disk-check.sh
```

Let's `add` to see what happens
```
git add .
git status
```
![Screenshot_20241114_143227](https://hackmd.io/_uploads/Hk1Fhd7Gye.png)

We can see that the file `test.sh` from the main branch is no longer here and we have a new file `disk-check.sh`
```
git commit -m "Uploading a disk-check"
```
![Screenshot_20241114_143644](https://hackmd.io/_uploads/ByjlaOmGJl.png)

Pushing now into our repo `scripts` with the new branch `no-clone`
```
git push scripts no-clone
```
![Screenshot_20241114_144602](https://hackmd.io/_uploads/B1bDyKQG1e.png)

We have successfully created, worked and published a new `branch`
![Screenshot_20241114_144952](https://hackmd.io/_uploads/BJRzxFmG1l.png)

Now we can merge the `branch` into `main`, we can specify a merge message or simply close the prompt
```
git switch main
git merge no-clone
git log
```
![Screenshot_20241114_145806](https://hackmd.io/_uploads/BJSzGKmfkx.png)

Finally we push the changes to `main`
```
git push -u scripts main
```
![Screenshot_20241114_150406](https://hackmd.io/_uploads/rkDqQKXzJl.png)
![Screenshot_20241114_150517](https://hackmd.io/_uploads/B112XKXzkl.png)

If we're done with the `branch` we can delete it, you have to be in **another branch**, you cannot delete the one you're currently on
```
git switch main
git branch -d no-clone
git push scripts --delete no-clone
```
![Screenshot_20241114_150849](https://hackmd.io/_uploads/HkeKNYmzJx.png)
![Screenshot_20241114_151555](https://hackmd.io/_uploads/Ski4IYmMyl.png)

We have successfully `merged` and `deleted` the leftover `branch`
![Screenshot_20241114_151721](https://hackmd.io/_uploads/BJ5c8YXGyg.png)
