# Adding Custom Finder Menu Actions

1. Finder->Applications->Automator
2. New->Service
3. No input in Finder
4. New "Run Shell Script"

```bash
STATUS=`defaults read com.apple.finder AppleShowAllFiles`
if [ $STATUS == YES ];
then
defaults write com.apple.finder AppleShowAllFiles NO
else
defaults write com.apple.finder AppleShowAllFiles YES
fi
killall Finder
```
 
6. Save as "Toggle Hidden Files"
7. Test by opening Finder, Finder->Services->Toggle Hidden Files
