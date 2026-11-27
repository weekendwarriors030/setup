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

## Steps to run repo migration with gh ado2gh
Set the env values
```
cp .env.example .env
# Add PATs to .env
export $(cat .env)
echo $ABC
```
This will set your PAT values correctly to run gh ado2gh

```
ORG=weekendwarriors030
REPO=test1
gh ado2gh migrate-repo \
  --ado-org $ORG \
  --github-org $ORG \
  --ado-repo $REPO \
  --github-repo $REPO \
  --ado-team-project $REPO
```

## Other migration tools
ADO2GH is just one of many
most require a PAT, so once you can run ADO2GH, you can likely run the other commands as well

