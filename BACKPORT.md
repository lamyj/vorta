# Backporting on macOS 10.11 (El Capitan)

Qt 5.12 and later do not support El Capitan anymore while [Qt 5.11](https://doc.qt.io/archives/qt-5.11/supported-platforms.html) still does. The latest Homebrew recipe using this version is [5eb54ce](https://github.com/Homebrew/homebrew-core/blob/5eb54ced793999e3dd3bce7c64c34e7ffe65ddfd/Formula/qt.rb).

## Vagrant image

Use [jhcook/osx-elcapitan-10.11](https://app.vagrantup.com/jhcook/boxes/osx-elcapitan-10.11/versions/10.11.6).

```shell
vagrant up darwin64
vagrant ssh darwin64
```

The `/vagrant` directory is not readable by the default user, however `sudo ls /vagrant` works.

The rsync command run by Vagrant is:

```
Command: "rsync" "--verbose" "--archive" "--delete" "-z" "--copy-links" "--no-owner" "--no-group" "--rsync-path" "sudo rsync" "-e" "ssh -p 2222 -o LogLevel=FATAL   -o ControlMaster=auto -o ControlPath=/var/folders/f7/4f_00c110h96pjgcsj5bkrdr00017j/T/vagrant-rsync-20190824-49689-1z07gfw -o ControlPersist=10m  -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i '/Users/julien/.vagrant.d/insecure_private_key'" "--exclude" ".vagrant/" "/Users/julien/src/vorta/" "vagrant@127.0.0.1:/vagrant"
```

## Homebrew and pip packages

[Homebrew] is installed when provision the Vagrant VM.

Qt recipe requires a full Xcode: fetch version 8.2 from [Apple Developer site](https://developer.apple.com/download/more/) (requires login, free download as of 2019-08-24). Make sure the signature is correct:

```
Macbook:Downloads myself$ pkgutil --verbose --check-signature ./Xcode_8.2.xip 
Package "Xcode_8.2.xip":
   Status: signed Apple Software
   Certificate Chain:
    1. Software Update
       SHA1 fingerprint: 1E 34 E3 91 C6 44 37 DD 24 BE 57 B1 66 7B 2F DA 09 76 E1 FD
       -----------------------------------------------------------------------------
    2. Apple Software Update Certification Authority
       SHA1 fingerprint: FA 02 79 0F CE 9D 93 00 89 C8 C2 51 0B BC 50 B4 85 8E 6F BF
       -----------------------------------------------------------------------------
    3. Apple Root CA
       SHA1 fingerprint: 61 1E 5B 66 2C 59 3A 08 FF 58 D1 4A E2 24 52 D1 98 DF 6C 60
```

Then copy to the VM:
```
rsync -a --inplace --progress myself@192.168.0.1:Downloads/Xcode*xip ./
```

Inside the GUI of the VM, double-click the archive and move the extracted application to `/Applications`.

Install Qt 5.11 :
```shell
brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/5eb54ced793999e3dd3bce7c64c34e7ffe65ddfd/Formula/qt.rb
```

When installing PyQt 5, we get a wheel for version 5.13, compiled with macOS 10.12. Install the wheel of version 5.11.2:

```shell
pip3 install https://files.pythonhosted.org/packages/31/22/e79a35bab2221b7bdbb3cdadb25bc9b492080b7529eec5fcbfd3f2d57606/PyQt5-5.11.3-5.11.2_a-cp35.cp36.cp37.cp38-abi3-macosx_10_6_intel.whl
```

## Borg

As of version 1.1.10, Borg runs correctly on El Capitan. We can use the [official standalone binary](https://borgbackup.readthedocs.io/en/stable/installation.html#standalone-binary):

```shell
mkdir -p bin/darwin
curl -L -o bin/darwin/borg https://github.com/borgbackup/borg/releases/download/1.1.10/borg-macosx64
chmod a+rx bin/darwin/borg
```

## Sparkle

Since this is a highly manual process, the Sparkle update framework is disabled in this branch:

```patch
diff --git a/src/vorta/updater.py b/src/vorta/updater.py
index 0ba3ebd..baf2bf8 100644
--- a/src/vorta/updater.py
+++ b/src/vorta/updater.py
@@ -4,7 +4,7 @@ from vorta.models import SettingsModel
 
 
 def get_updater():
-    if sys.platform == 'darwin' and getattr(sys, 'frozen', False):
+    if False and sys.platform == 'darwin' and getattr(sys, 'frozen', False):
         """
         Use Sparkle framework on macOS.
```

## Build Vorta

Start by disabling copying the Sparkle framework and code signing the in Makefile:

```patch
diff --git a/Makefile b/Makefile
index 5e053ff..911bce4 100644
--- a/Makefile
+++ b/Makefile
@@ -11,8 +11,8 @@ icon-resources:  ## Compile SVG icons to importable resource files.
 
 Vorta.app: translations-to-qm
        pyinstaller --clean --noconfirm vorta.spec
-       cp -R bin/darwin/Sparkle.framework dist/Vorta.app/Contents/Frameworks/
-       cd dist; codesign --deep --sign 'Developer ID Application: Manuel Riel (CNMSCAXT48)' Vorta.app
+       #cp -R bin/darwin/Sparkle.framework dist/Vorta.app/Contents/Frameworks/
+       #cd dist; codesign --deep --sign 'Developer ID Application: Manuel Riel (CNMSCAXT48)' Vorta.app
 
 Vorta.dmg-Vagrant:
        vagrant up darwin64
```

Then `make Vorta.app`. In case of failure, the project can be cleaned by:

```shell
rm -rf build/* dist/* src/vorta/__pycache__/ src/vorta/i18n/qm/vorta.*.qm
```

## Packaging

The packaging step also requires disabling code signing. Note that this must be done in the host, not in the VM.

```patch
diff --git a/appdmg.json b/appdmg.json
index 8915a36..4c99382 100644
--- a/appdmg.json
+++ b/appdmg.json
@@ -5,9 +5,6 @@
     { "x": 162, "y": 144, "type": "file", "path": "dist/Vorta.app" }
   ],
   "format": "ULFO",
-  "code-sign": {
-    "signing-identity": "Developer ID Application: Manuel Riel (CNMSCAXT48)"
-  },
   "window": {
     "size": {
       "height": 300,
```

Afterwards, we can follow the `Makefile` by copying the data out of the Vagrant VM and building the DMG:

```shell
vagrant scp darwin64:/vagrant/dist/Vorta.app dist/
appdmg appdmg.json dist/vorta-el-capitan-0.6.22.dmg
```
