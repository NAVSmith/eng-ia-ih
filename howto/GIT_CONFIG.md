# How to prepare your git environment
- Request access to [UMG GitHub](https://github.com/umg)

- Step 1: Generate new [SSH private key(s)](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
  - Generating a private key with a **passphrase** is a requirement on Mac :warning:
  - Enter this command in the terminal
    ```shell
    ssh-keygen -t rsa -b 4096 -C "your@github.email" -f ~/.ssh/id_rsa_github

- Step 2: Locate file and copy SSH key 

  - Either use this command:
      ```shell
     pbcopy ~/.ssh/id_rsa_github.pub
       ```
  -  Or locate the file itself on your machine and open it to copy the key.

- Step 3: Copy key onto your github account here:
https://github.com/settings/keys

- Step 4: [Make keys "persistent"](https://unix.stackexchange.com/a/560404/171941) so they automatically load after Mac reboot
   - Enter this command in the terminal:
    ```shell 
      ssh-add --apple-use-keychain ~/.ssh/<.your-privatekey-filename.>
    ```
  - Create an ssh config file if you don't have one
  - Navigate to the ssh directory and enter this command in the terminal:
     ``` shell 
    nano config
    ```
  - Then copy and paste this into the editor

  
    ```
    Host *
        UseKeychain yes
        AddKeysToAgent yes
        IgnoreUnknown UseKeychain

    Host github.com
        identityfile = ~/.ssh/id_rsa_github
        user = git
    ```
- For GitHub make sure you authorized your ssh-key with [UMG organization](https://docs.github.com/en/enterprise-cloud@latest/authentication/authenticating-with-saml-single-sign-on/authorizing-an-ssh-key-for-use-with-saml-single-sign-on)
- Step 5: Generate new [GPG signing key(s)](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key)
  - Note your GPG key ID, It begins after the / character in the sec paragraph when executing the command
      ```shell
      gpg --list-secret-keys --keyid-format LONG <EMAIL>
      ```
  - Set up `~/.gitconfig` so that it automatically picks up your git config based on the folder prefix
    - `~/.gitconfig` (we suggest to follow folder paths for all UMG cloned repositories: Projects/github/umg)
      ```ini
      [core]
      editor = nano

      [commit]
      gpgsign = true

      [user]
      email = your@github.email
      name = your-github-username
      signingkey = your GPG key ID
      ```

- Make sure that authentication works by cloning some repositories using SSH protocol (`git clone git@github.com:umg/...`)
- Make sure that signing works by committing and pushing a commit (automatically signed using `git commit -m "..."`)

## Troubleshooting

- If you get the error error: `gpg failed to sign the data` try running `export GPG_TTY=$(tty)` before committing and add it to your `~/.zshrc` config if it helped
    ```shell
    echo 'export GPG_TTY=$(tty)' >> ~/.zshrc
    source ~/.zshrc
    ```
