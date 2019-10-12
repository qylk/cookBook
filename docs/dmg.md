# DMG制作
未加密dmg
```bash
hdiutil create
-srcfolder ./docs ##输入目录
-ov a.dmg ##输出
```
加密dmg
```bash
echo -n "password" | hdiutil create 
-encryption -stdinpass 
-srcfolder ./docs
encrypted.dmg
```
将dmg转换为可读写模式
```bash
hdiutil convert input.dmg  -format UDRW -o output.dmg
```

iso制作
```bash
hdiutil makehybrid -iso <srcFolder> -o dest.iso
```

hdiutil参数查询：<https://ss64.com/osx/hdiutil.html>