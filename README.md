# Electron Nuxt Action

**GitHub Action for building and releasing [electron-nuxt](https://github.com/michalzaq12/electron-nuxt) apps**

## Setup
**Add a workflow file** to your project (e.g. `.github/workflows/build.yml`).
Using the workflow below, GitHub will build your app every time you push a commit to master branch.

   ```yml
   name: Build/release

   on:
     push:
       branches:
         - master

   jobs:
     release:
       runs-on: ${{ matrix.os }}

       strategy:
         matrix:
           os: [macos-latest, ubuntu-latest, windows-latest]

       steps:
         - name: Check out Git repository
           uses: actions/checkout@v1

         - name: Install Node.js, NPM and Yarn
           uses: actions/setup-node@v1
           with:
             node-version: 12

         - name: Build/release Electron app
           uses: michalzaq12/action-electron-nuxt@v1
           with:
             # GitHub token, automatically provided to the action
             # (No need to define this secret in the repo settings)
             # type: string
             github_token: ${{ secrets.github_token }}

             # If the commit is tagged with a version (e.g. "v1.0.0")
             # type: boolean
             release: ${{ startsWith(github.ref, 'refs/tags/v') }}
   ```


## Configuration

### Code Signing

If you are building for **macOS**, you'll want your code to be [signed](https://samuelmeuli.com/blog/2019-04-07-packaging-and-publishing-an-electron-app/#code-signing). GitHub Actions therefore needs access to your code signing certificates:

- Open the Keychain Access app or the Apple Developer Portal. Export all certificates related to your app into a _single_ file (e.g. `certs.p12`) and set a strong password
- Base64-encode your certificates using the following command: `base64 -i certs.p12 -o encoded.txt`
- In your project's GitHub repository, go to Settings → Secrets and add the following two variables:
  - `mac_certs`: Your encoded certificates, i.e. the content of the `encoded.txt` file you created before
  - `mac_certs_password`: The password you set when exporting the certificates

Add the following options to your workflow's existing `action-electron-builder` step:

```yml
- name: Build/release Electron app
  uses: michalzaq12/action-electron-nuxt@v1
  with:
    # ...
    mac_certs: ${{ secrets.mac_certs }}
    mac_certs_password: ${{ secrets.mac_certs_password }}
```

The same goes for **Windows** code signing (`windows_certs` and `windows_certs_password` secrets).

### Snapcraft

If you are building/releasing your Linux app for Snapcraft (which is `electron-builder`'s default), you will additionally need to install and sign in to Snapcraft. This can be done using an `action-snapcraft` step before the `action-electron-builder` step:

```yml
- name: Install Snapcraft
  uses: samuelmeuli/action-snapcraft@v1
  # Only install Snapcraft on Ubuntu
  if: startsWith(matrix.os, 'ubuntu')
  with:
    # Log in to Snap Store
    snapcraft_token: ${{ secrets.snapcraft_token }}
```

You can read [here](https://github.com/samuelmeuli/action-snapcraft) how you can obtain a `snapcraft_token`.

### Notarization

If you've configured `electron-builder` to notarize your Electron Mac app [as described in this guide](https://samuelmeuli.com/blog/2019-12-28-notarizing-your-electron-app), you can use the following steps to let GitHub Actions perform the notarization for you:

1.  Define the following secrets in your repository's settings on GitHub:

    - `api_key`: Content of the API key file (with the `p8` file extension)
    - `api_key_id`: Key ID found on App Store Connect
    - `api_key_issuer_id`: Issuer ID found on App Store Connect

2.  In your workflow file, add the following step before your `action-electron-builder` step:

    ```yml
    - name: Prepare for app notarization
      if: startsWith(matrix.os, 'macos')
      # Import Apple API key for app notarization on macOS
      run: |
        mkdir -p ~/private_keys/
        echo '${{ secrets.api_key }}' > ~/private_keys/AuthKey_${{ secrets.api_key_id }}.p8
    ```

3.  Pass the following environment variables to `action-electron-builder`:

    ```yml
    - name: Build/release Electron app
      uses: michalzaq12/action-electron-nuxt@v1
      with:
        # ...
      env:
        # macOS notarization API key
        API_KEY_ID: ${{ secrets.api_key_id }}
        API_KEY_ISSUER_ID: ${{ secrets.api_key_issuer_id }}
    ```


### Credits

https://github.com/samuelmeuli/action-electron-builder
