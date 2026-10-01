---
max_turns: 25
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill]
tags: [engine, unity]
---

Unity project in `project` (we ship a WebGL build to our website, ~60k DAU, AppLixir key in hand).
Wire the free-gems button in ShopUI.cs to an AppLixir rewarded ad that gives 10 gems. Gems are
cosmetic and local for now. Plan approved - give me all the files/code in your reply; you can't
write files here.

The `project` files are pasted below (you can't open them directly):

`project/ProjectSettings/ProjectVersion.txt`
```txt
m_EditorVersion: 2022.3.40f1
```

`project/Assets/Scripts/ShopUI.cs`
```cs
using UnityEngine;
using UnityEngine.UI;

public class ShopUI : MonoBehaviour
{
    public Button freeGemsButton;
    public Text gemsLabel;
    int gems;

    void Start()
    {
        gemsLabel.text = gems.ToString();
        // TODO: rewarded ad -> +10 gems
    }

    public void AddGems(int n) { gems += n; gemsLabel.text = gems.ToString(); }
}
```
