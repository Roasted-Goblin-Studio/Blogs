# How to build a Github pipeline and deploy a Unity project to Itch.io

## Prereq
  - A Github repo
  - A Unity project
  - A Unity account
  - A Itch.io project

## Definitions
  - When a step instructs you to open a file for write purposes, please do so with your preferred text editor.
  - When a step instructs you to navigate to a webpage, please do so with your preferred web browser
  - If a step instructs you to download a file, it is recommended you do so in a new folder

## Setup
### Get the Unity license
Before we can run our pipeline we will need to request a license from Unity for our new Github pipeline.

##### Steps
###### Create license
  1. Create the following folders+file in the root level of your project folder; `.github/workflows/activation.yml`
  2. In the `activation.yml` file add the following;
```
name: Acquire activation file
on:
  workflow_dispatch: {}
jobs:
  activation:
    name: Request manual activation file 🔑
    runs-on: ubuntu-latest
    steps:
      # Request manual activation file
      - name: Request manual activation file
        id: getManualLicenseFile
        uses: game-ci/unity-request-activation-file@v2
      # Upload artifact (Unity_v20XX.X.XXXX.alf)
      - name: Expose as artifact
        uses: actions/upload-artifact@v2
        with:
          name: ${{ steps.getManualLicenseFile.outputs.filePath }}
          path: ${{ steps.getManualLicenseFile.outputs.filePath }}
```
  3. Save the file and push the code to your remote repository
  4. Navigate to your Github project repository page in your browser
  5. Select `Actions` in the repository options
  6. Under the `Workflow` select `Acquire activation file`
  7. Once the pipeline finishes, download the license artifact and extract it
  8. Navigate to your Github project repository page in your browser
  9. Select `Settings` in the repository options > Then select `Secrets`
  10. Create the following secrets;
    - UNITY_LICENSE - (Copy the contents of your license file into here)
    - UNITY_EMAIL - (Add the email address that you use to login to Unity)
    - UNITY_PASSWORD - (Add the password that you use to login to Unity)
  
  
###### Create a Personal Access Token (PAT)
  - https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token


###### Create a Itch.io API Key
  - https://itch.io/docs/api/serverside
    - Under the header `API keys` 
    - Create a secret in your github repository named `BUTLER_API_KEY` to store the api key. 

## Steps
### Write the pipeline
  1. Create the following folders+file in the root level of your project folder; `.github/workflows/cicd-unity.yml`
  2. In the `cicd-unity.yml` file add the following;
```
name: Build, Archive Artifact and Publish to Itch.io

on: 
  workflow_dispatch: {}
  
env:
  ITCH_PROJECT_NAME: <REPLACE WITH YOUR ITCH.IO PROJECT NAME>

jobs:
#-----------------------------------------------------------------------------
# BUILD
#-----------------------------------------------------------------------------
  buildForAllPlatforms:
    name: Build for ${{ matrix.targetPlatform }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        targetPlatform:
          - StandaloneOSX
          - StandaloneWindows
          - StandaloneWindows64
          - StandaloneLinux64
          - WebGL
        unityVersion:
          - 2020.3.30f1
          
    steps:
      - name: Inject slug/short variables
        uses: rlespinasse/github-slug-action@v3.x
      - uses: actions/checkout@v2
        name: Checkout Source
        with:
          fetch-depth: 0
          lfs: true

      - uses: actions/cache@v2
        name: Check Cache
        with:
          path: Library
          key: Library-${{ matrix.targetPlatform }}
          restore-keys: Library-

      - uses: game-ci/unity-builder@main
        name: Build for Platform - ${{ matrix.targetPlatform }}
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          targetPlatform: ${{ matrix.targetPlatform }}
          unityVersion: ${{ matrix.unityVersion }}

#-----------------------------------------------------------------------------
# ARCHIVE
#-----------------------------------------------------------------------------      
      - uses: montudor/action-zip@v1
        name: "Zip Builds - ${{ matrix.targetPlatform }}"
        #if: github.event.action == 'published'
        with:
          args: zip -qq -r build/build-${{ matrix.targetPlatform }}.zip build/${{ matrix.platform }}

      - uses: svenstaro/upload-release-action@v2
        name: Upload build-${{ matrix.targetPlatform }}.zip to Github Release
        #if: github.event.action == 'published'
        with:
          repo_token: ${{ secrets.ACCESS_TOKEN }}
          asset_name: build-${{ matrix.targetPlatform }}.zip
          file: build/build-${{ matrix.targetPlatform }}.zip
          tag: ${{ github.ref }}
          body: "Automated deployment from ${{ ENV.GITHUB_REF_SLUG }} branch"
          #prerelease: true
          overwrite: true        

#-----------------------------------------------------------------------------
# DEPLOY
#-----------------------------------------------------------------------------
      - name: Upload StandaloneOSX build to Itch.io
        uses: josephbmanley/butler-publish-itchio-action@master
        if: contains(matrix.targetPlatform, 'StandaloneOSX')
        env:
          BUTLER_CREDENTIALS: ${{ secrets.BUTLER_API_KEY }}
          CHANNEL: osx
          ITCH_GAME: ${{ env.ITCH_PROJECT_NAME }}
          ITCH_USER: roasted-goblin-studios
          PACKAGE: build/build-StandaloneOSX.zip
          VERSION: "develop"
      
      - name: Upload StandaloneWindows build to Itch.io
        uses: josephbmanley/butler-publish-itchio-action@master
        if: contains(matrix.targetPlatform, 'StandaloneWindows')
        env:
          BUTLER_CREDENTIALS: ${{ secrets.BUTLER_API_KEY }}
          CHANNEL: windows
          ITCH_GAME: ${{ env.ITCH_PROJECT_NAME }}
          ITCH_USER: roasted-goblin-studios
          PACKAGE: build/build-StandaloneWindows.zip
          VERSION: "develop"
      
      - name: Upload StandaloneLinux64 build to Itch.io
        uses: josephbmanley/butler-publish-itchio-action@master
        if: contains(matrix.targetPlatform, 'StandaloneLinux64')
        env:
          BUTLER_CREDENTIALS: ${{ secrets.BUTLER_API_KEY }}
          CHANNEL: linux
          ITCH_GAME: ${{ env.ITCH_PROJECT_NAME }}
          ITCH_USER: roasted-goblin-studios
          PACKAGE: build/build-StandaloneLinux64.zip
          VERSION: "develop"
      
      - name: Upload WebGL build to Itch.io
        uses: josephbmanley/butler-publish-itchio-action@master
        if: contains(matrix.targetPlatform, 'WebGL')
        env:
          BUTLER_CREDENTIALS: ${{ secrets.BUTLER_API_KEY }}
          CHANNEL: HTML5
          ITCH_GAME: ${{ env.ITCH_PROJECT_NAME }}
          ITCH_USER: roasted-goblin-studios
          PACKAGE: build/build-WebGL.zip
          VERSION: "develop"
```
  3. Save the file and push the code to your remote repository
  4. Navigate to your Github project repository page in your browser
  5. Select `Actions` in the repository options
  6. Under the `Workflow` select `Build, Archive Artifact and Publish to Itch.io`
  7. Keep an eye on your Itch.io page 
