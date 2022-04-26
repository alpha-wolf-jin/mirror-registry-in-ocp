# mirror-registry-in-ocp

In the disconnection environemnt, we need one mirroring registry. 

But we cannot found extra VM to create mirroring registry and port 5000 in bastion is blocked. 

Here, we create one serice in OCP for mirroring registry.

## update git

```
echo "# mirror-registry-in-ocp" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/mirror-registry-in-ocp.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main


git add . ; git commit -a -m "update README" ; git push -u origin main
```
