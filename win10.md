# Windows 10 CLI Tools

## Package Management with Scoop
Install scoop from Windows Powershell
``` sh
Set-ExecutionPolicy RemoteSigned -scope CurrentUser
iwr -useb get.scoop.sh | iex
```

## PowerShell
Put following line to $PROFILE to use Emacs edit mode.
``` sh
Set-PSReadLineOption -EditMode Emacs
```

### oh-my-posh

## Windows Terminal

## Alacritty

## NeoVim

``` sh
scoop install neovim
```

## Clang/Clang++

``` sh
scoop install llvm
```

## Miscs

### grep

### curl

### wget

### Swapping Left Ctrl & CapsLock keys

In regedit tool, find ``Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout``, add key ``Scancode Map`` as type ``REG_BINARY``, with value:

```
00 00 00 00 00 00 00 00
03 00 00 00 1D 00 3A 00
3A 00 1D 00 00 00 00 00
```

Then reboot windows.

Consider using [Autohotkey](https://www.autohotkey.com/) if you only want to swap occasionally. Just put a simple line in your ``ahk`` file.
```
Capslock::Ctrl
```

