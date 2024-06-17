## PlayMusicLocal / 本地播放管弦乐琴

界面所用的 `PageIndex` 和实际数据表中的 `RowID` 需要转换

```csharp
private delegate nint PlayMusicPreCheckDelegate(nint a1, uint index);
[Signature("40 53 48 83 EC ?? 4C 8B 81 ?? ?? ?? ?? 48 B8 ?? ?? ?? ?? ?? ?? ?? ?? 4C 2B 81 ?? ?? ?? ?? 48 8B D9 44 8B CA 49 F7 E8 49 03 D0 48 C1 FA ?? 48 8B C2 48 C1 E8 ?? 48 03 D0 44 3B CA 0F 83 ?? ?? ?? ?? 49 6B C9", DetourName = nameof(PlayMusicHotelPreCheckDetour))]
private static Hook<PlayMusicPreCheckDelegate>? PlayMusicHotelPreCheckHook;

[Signature("E8 ?? ?? ?? ?? 32 C0 48 83 C4 ?? 5B C3 CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC 48 89 5C 24", DetourName = nameof(PlayMusicHomePreCheckDetour))]
private static Hook<PlayMusicPreCheckDelegate>? PlayMusicHomePreCheckHook;

[Signature("E8 ?? ?? ?? ?? 32 C0 48 83 C4 ?? 5B C3 48 8B CB E8 ?? ?? ?? ?? 32 C0 48 83 C4 ?? 5B C3 48 8B CB", DetourName = nameof(AddToPlaylistDetour))]
private static Hook<PlayMusicPreCheckDelegate>? PlayMusicAddToPlaylistHook;

private delegate nint PlayMusicDelegate(uint a1);
[Signature("40 57 48 83 EC ?? 0F B7 F9 B9")]
private static PlayMusicDelegate? PlayMusic;

private static nint PlayMusicHotelPreCheckDetour(nint a1, uint pageIndex)
{
    UnlockAndParseOrchestions(a1, pageIndex, out var musicIndex);
    
    PlayMusic(musicIndex);
    // PlayMusicHotelPreCheckHook.Original(a1, pageIndex);
    return 0;
}

private static nint PlayMusicHomePreCheckDetour(nint a1, uint pageIndex)
{
    UnlockAndParseOrchestions(a1, pageIndex, out var musicIndex);

    PlayMusic(musicIndex);
    // var original = PlayMusicHomePreCheckHook.Original(a1, pageIndex);
    return 0;
}

private static nint AddToPlaylistDetour(nint a1, uint pageIndex)
{
    UnlockAndParseOrchestions(a1, pageIndex, out _);

    var original = PlayMusicAddToPlaylistHook.Original(a1, pageIndex);
    return original;
}

private static void UnlockAndParseOrchestions(nint a1, uint pageIndex, out uint musicIndex)
{
    var musicAddress = (uint*)(*(long*)(a1 + 1744) + (120L * pageIndex));
    musicIndex = *musicAddress;

    var musicUnlock = *((byte*)musicAddress + 8);
    if (musicUnlock != 1)
        SafeMemory.Write((nint)((byte*)musicAddress + 8), (byte)1);
}
```



## UnlockFreeCompanyChestLode / 强制解锁部队箱加载锁

在 `Agent.Show` 的开头函数就会送出 `Inventory` 的请求, 也是有缓存数据到本地的, 如果网络通畅大概 `100ms` 可以接收到返回的数据, 而加载锁为 `3s` 左右, 因此这里可以直接返回 `true` 以解锁, 但其内部也包含了请求 `Inventory` 的代码, 可能需要手动发送以保证数据正确 

```csharp
private delegate bool SendInventoryRefreshDelegate(InventoryManager* instance, int inventoryType);
[Signature("48 89 5C 24 ?? 48 89 74 24 ?? 57 48 83 EC ?? 8B F2 48 8B D9 33 D2 0F B7 FA", DetourName = nameof(SendInventoryRefreshDetour))]
private static Hook<SendInventoryRefreshDelegate>? SendInventoryRefreshHook;

private static bool SendInventoryRefreshDetour(InventoryManager* instance, int inventoryType)
{
    // 直接返回 true 防锁
    Service.ExecuteCommandManager.ExecuteCommand(ExecuteCommandFlag.RequestInventory, inventoryType);
    return true;
}
```



## PlaySoundEffect / 播放系统音效

玩家可用的 `se` 序号需要将 `sound` 减去 `36` , 如果小于等于 `16` 则为 `se` , 大于则为其他的音效文件

```csharp
private delegate void PlaySoundEffectDelegate(uint sound, nint a2, nint a3, byte a4);
[Signature("40 53 41 55 48 81 EC ?? ?? ?? ?? 48 8B 05 ?? ?? ?? ?? 48 33 C4 48 89 84 24 ?? ?? ?? ?? F6 05", DetourName = nameof(PlaySoundEffectDetour))]
private static Hook<PlaySoundEffectDelegate>? PlaySoundEffectHook;

private static void PlaySoundEffectDetour(uint sound, nint a2, nint a3, byte a4)
{
    var se = sound - 36;
    PlaySoundEffectHook.Original(sound, a2, a3, a4);
}
```



## GetGrandCompanyRank / 获取军衔等级

如果 `original` 返回 `0`, 则代表玩家未加入任一大国防联军, 最低为 `1`, 最高为 `19`, 玩家可达到等级为 `11`, 相关返回值会存储在别的地方, 建议存储原始值以便还原, 相关界面根据此处获取值来确认 `筹备稀有品` 和 `军队补给品` 界面的开放状态, 可以提交装备但不可以购买

```csharp
private delegate byte GetGrandCompanyRankDeleagte(PlayerState* instance);
[Signature("E8 ?? ?? ?? ?? 3C ?? 88 44 24", DetourName = nameof(GetGrandCompanyRankDetour))]
private static Hook<GetGrandCompanyRankDeleagte>? GetGrandCompanyRankHook;

private static byte GetGrandCompanyRankDetour(PlayerState* instance)
{
    var original = GetGrandCompanyRankHook.Original(instance);
    if (original == 0) return original;

    OriginalRank ??= original;
    return (byte)CustomRank;
}
```



## Return / 返回

```csharp
private delegate byte ReturnDelegate(AgentInterface* agentReturn);
[Signature("48 89 5C 24 ?? 48 89 74 24 ?? 57 48 83 EC ?? 48 8B 3D ?? ?? ?? ?? 48 8B D9 48 8D 0D", DetourName = nameof(ReturnDetour))]
private static Hook<ReturnDelegate>? ReturnHook;
```

