Git
====

## Transferring from TFS to git

* NOTE: "Git" is not the same as "GitHub"
        - Git is a Version Control Repository Framework
        - GitHub is a company (the most popular) that hosts git repositories and allows them to be viewed from a web interface
        


### Commands

|TFS (From inside Visual Studio)   |Git (Using command line interface)   |
|------|--------|
|"Get latest"| `>$ git pull`  |
|"Check in Pending Changes"| `>$ git commit -m "[Commit Message]"`<br> `>$ git push`  |
| "Undo Pending Changes"| `>$ git add .`<br> `>$ git reset --hard`|
| "Compare"| `>$ git status`|
|||