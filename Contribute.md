## 0. Clone Repository
```bash
$ git clone https://jmg7173/kernel_study
```

### Simple Setup at git

* Specify user  
```bash
$ git config --global user.name "YOUR USER NAME"
$ git config --global user.email "YOUR EMAIL"
```

* Set default editor  

In default, editor is set as nano. If you want to use vim, please set default editor for git.
```bash
$ git config --global core.editor vim
```


## 1. Make git branch

<U>**Please MAKE BRANCH**</U>

* Example: `git checkout -b 01-BareMetal`

## 2. Commit and Push
After editing your part, add your file at repository.

See [this](https://gist.github.com/ihoneymon/652be052a0727ad59601) for editing your part using Markdown.

```bash
$ git branch # Check your current branch
$ git add YOURFILE.md
$ git commit -m "Edit YOURFILE.md"
$ git push origin YOUR_BRANCH

# Example for push
$ git push origin 01-BareMetal
```

## 3. Merge
Send Pull request at repository after finish your part.
* Link: https://github.com/jmg7173/kernel_study/pulls


## 4. Update your local repository
```bash
$ git pull origin master
```
