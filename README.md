# setup
Setting up GH

## Steps to get PAT
https://github.com/settings/tokens

Settings -> Developer Settings -> Personal Access Tokens -> Tokens (classic)

Classic key is easier to setup with all permissions

## Steps to get SSH key
https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent
```
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-agent
cat ~/.ssh/id_ed25519
```
add to ssh keys:

https://github.com/settings/keys

Settings -> SSH and GPG Keys
