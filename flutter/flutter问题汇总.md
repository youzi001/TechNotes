# 安装Flutter时报错 ideviceinfo returned an error

https://github.com/flutter/flutter/issues/24600

I don't not why. But I solved it with restart Computer or run:

```
brew update
brew uninstall --ignore-dependencies libimobiledevice
brew uninstall --ignore-dependencies usbmuxd
brew install --HEAD usbmuxd
brew unlink usbmuxd
brew link usbmuxd
brew install --HEAD libimobiledevice
brew install ideviceinstaller
```
