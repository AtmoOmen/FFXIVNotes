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

导引到 `ActionManager.UseAction` , 如果成功则返回 `1`

```csharp
private delegate byte ReturnDelegate(AgentInterface* agentReturn);
[Signature("48 89 5C 24 ?? 48 89 74 24 ?? 57 48 83 EC ?? 48 8B 3D ?? ?? ?? ?? 48 8B D9 48 8D 0D", DetourName = nameof(ReturnDetour))]
private static Hook<ReturnDelegate>? ReturnHook;
```



## IsTargetable / 游戏物体是否可选中

只要 `LocalPlayer` 不为空就会反复遍历包含自己在内的所有 `GameObject` 的可选中状态, 导引到 `GameObject.GetIsTargetable()`, 会导致 PVP 内使用 `冲天` 的 `龙骑士` 也可被选中, 但实际不可对其使用技能 ~~(可以用做 Framework.Update)~~

```csharp
private delegate bool IsTargetableDelegate(GameObjectStruct* gameObj);
[Signature("40 53 48 83 EC 20 F3 0F 10 89 ?? ?? ?? ?? 0F 57 C0 0F 2E C8 48 8B D9 7A 0A", DetourName = nameof(IsTargetableDetour))]
private static Hook<IsTargetableDelegate>? IsTargetableHook;
```



## GetClipboardData / 获取剪贴板数据

从系统剪贴板内获取数据, 最后返回 Utf8String*

```csharp
private delegate Utf8String* GetClipboardDataDelegate(nint a1);
[Signature("40 53 48 81 EC ?? ?? ?? ?? 48 8B 05 ?? ?? ?? ?? 48 33 C4 48 89 84 24 ?? ?? ?? ?? 48 8B D9 BA", DetourName = nameof(GetClipboardDataDetour))]
private static Hook<GetClipboardDataDelegate>? GetClipboardDataHook;

private static Utf8String* GetClipboardDataDetour(nint a1)
{
    var copyModule = Framework.Instance()->GetUIClipboard();
    if (copyModule == null) return GetClipboardDataHook.Original(a1);

    var originalText = Clipboard.GetText();
    if (string.IsNullOrWhiteSpace(originalText)) return GetClipboardDataHook.Original(a1);
    
    // 多行转单行
    var modifiedText = originalText.Replace("\r\n", " ").Replace("\n", " ").Replace("\u000D", " ")
                                   .Replace("\u000D\u000A", " ");

    if (modifiedText == originalText) return GetClipboardDataHook.Original(a1);

    return Utf8String.FromString(modifiedText);
}
```



## CutsceneInput / 过场剧情输入

整体用于判断处理过场剧情是否可跳过

```csharp
private delegate void CutsceneHandleInputDelegate(nint a1);
[Signature("40 53 48 83 EC 20 80 79 29 00 48 8B D9 0F 85", DetourName = nameof(CutsceneHandleInputDetour))]
private static Hook<CutsceneHandleInputDelegate>? CutsceneHandleInputHook;

    private unsafe void CutsceneHandleInputDetour(nint a1)
    {
        var allowSkip = *(nint*)(a1 + 56) != 0;
        if (allowSkip)
        {
            SafeMemory.WriteBytes(ConditionAddress, [0xEB]);
            CutsceneHandleInputHook.Original(a1);
            SafeMemory.WriteBytes(ConditionAddress, [0x75]);

            TaskHelper.Enqueue(() =>
            {
                if (!TryGetAddonByName<AtkUnitBase>("SelectString", out var addon) || !IsAddonAndNodesReady(addon))
                    return false;

                if (addon->GetTextNodeById(2)->
                        NodeText.ExtractText().Contains(LuminaCache.GetRow<Addon>(281).Text.RawString))
                {
                    if (Click.TrySendClick("select_string1"))
                    {
                        TaskHelper.Abort();
                        return true;
                    }

                    return false;
                }

                return false;
            });

            return;
        }

        CutsceneHandleInputHook.Original(a1);
    }

```



## ProcessSendedChat / 处理将发送聊天信息

位于发送服务器前, 整体处理起来需要关注 `byte**` 和 `500` 的 TextLength

```csharp
private delegate byte ProcessSendedChatDelegate(nint uiModule, byte** message, nint a3);
[Signature("E8 ?? ?? ?? ?? FE 86 ?? ?? ?? ?? C7 86 ?? ?? ?? ?? ?? ?? ?? ??", DetourName = nameof(ProcessSendedChatDetour))]
private static Hook<ProcessSendedChatDelegate>? ProcessSendedChatHook;

private byte ProcessSendedChatDetour(nint uiModule, byte** message, nint a3)
{
    var originalSeString = MemoryHelper.ReadSeStringNullTerminated((nint)(*message));
    var messageDecode = originalSeString.ToString();

    if (string.IsNullOrWhiteSpace(messageDecode) || messageDecode.StartsWith('/'))
        return ProcessSendedChatHook.Original(uiModule, message, a3);

    var ssb = new SeStringBuilder();
    foreach (var payload in originalSeString.Payloads)
    {
        if (payload.Type == PayloadType.RawText)
            ssb.Append(BypassCensorship(((TextPayload)payload).Text));
        else
            ssb.Add(payload);
    }

    var filteredSeString = ssb.Build();

    if (filteredSeString.TextValue.Length <= 500)
    {
        var utf8String = Utf8String.FromString(".");
        utf8String->SetString(filteredSeString.Encode());

        return ProcessSendedChatHook.Original(uiModule, (byte**)((nint)utf8String).ToPointer(), a3);
    }

    return ProcessSendedChatHook.Original(uiModule, message, a3);
}
```



## BypassCensorship / 绕过屏蔽词显示

屏蔽词均为本地客户端处理, 服务端始终接收原始文本, 聊天框和队员招募两个地方的屏蔽处理都是 `原始字符串->屏蔽词系统->复制字符串->返回`, 因此处理逻辑都是直接绕过中间的屏蔽词系统直接调用复制字符串的函数, 然后返回

```csharp
private delegate void GetFilteredUtf8StringDelegate(nint vulgarInstance, Utf8String* str);
[Signature("E8 ?? ?? ?? ?? 48 8B C3 48 83 C4 ?? 5B C3 CC CC CC CC CC CC CC 48 83 EC")]
private static GetFilteredUtf8StringDelegate? GetFilteredUtf8String;

private delegate nint Utf8StringCopyDelegate(nint a1, nint a2, nint a3 = 1);
[Signature("48 89 5C 24 ?? 57 48 83 EC ?? 48 8B FA 48 8B D9 48 3B D1 74 ?? 48 8B 52 ?? 41 B0 ?? E8 ?? ?? ?? ?? 4C 8B 43 ?? 48 8B 17 48 8B 0B E8 ?? ?? ?? ?? 0F B6 47 ?? 88 43 ?? 48 8B 47 ?? 48 89 43 ?? 48 8B C3 48 8B 5C 24 ?? 48 83 C4 ?? 5F C3 CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC 48 89 5C 24")]
private static Utf8StringCopyDelegate? Utf8StringCopy;

private delegate nint LocalMessageDisplayDelegate(nint a1, nint a2);
[Signature("40 53 48 83 EC ?? 48 8D 99 ?? ?? ?? ?? 48 8B CB E8 ?? ?? ?? ?? 48 8B 0D", DetourName = nameof(LocalMessageDisplayDetour))]
private static Hook<LocalMessageDisplayDelegate>? LocalMessageDisplayHook;

private delegate nint PartyFinderMessageDisplayDelegate(nint a1, nint a2);
[Signature("48 89 5C 24 ?? 57 48 83 EC ?? 48 8D 99 ?? ?? ?? ?? 48 8B F9 48 8B CB E8", DetourName = nameof(PartyFinderMessageDisplayDetour))]
private static Hook<PartyFinderMessageDisplayDelegate>? PartyFinderMessageDisplayHook;

private static nint LocalMessageDisplayDetour(nint a1, nint a2)
    => Utf8StringCopy(a1 + 1096, a2);

private static nint PartyFinderMessageDisplayDetour(nint a1, nint a2)
    => Utf8StringCopy(a1 + 10488, a2);

// 获取屏蔽词处理后的文本
private string GetFilteredString(string str)
{
    var utf8String = Utf8String.FromString(str);
    GetFilteredUtf8String(Marshal.ReadIntPtr((nint)Framework.Instance() + 0x2B40), utf8String);

    return (*utf8String).ToString();
}
```



## IsFlightProhibited / 是否禁止飞行

用于检测当前区域是否禁止飞行, 也可以手动传回 `true` 来在本地客户端解锁飞行, 但在他人视角来看是在地面瞬移

```csharp
private delegate nint IsFlightProhibited(nint a1);
[Signature("48 89 5C 24 ?? 57 48 83 EC 20 48 8B 1D ?? ?? ?? ?? 48 8B F9 48 85 DB 0F 84 ?? ?? ?? ?? 80 3D", DetourName = nameof(IsFlightProhibitedDetour))]
private static Hook<IsFlightProhibited>? IsFlightProhibitedHook;

// a1 可以从 48 8D 0D ?? ?? ?? ?? E8 ?? ?? ?? ?? 85 C0 0F 85 ?? ?? ?? ?? 48 89 B4 24 获取
private nint IsFlightProhibitedDetour(nint a1) 
{ 
    return isProhibited ? IsFlightProhibitedHook.Original(a1) : 0; 
}
```



## GetActionRange / 获取技能释放距离

获取技能可释放距离, 最多可在本地 `+2`

```csharp
private delegate float GetActionRangeDelegate(uint actionID);
[Signature("E8 ?? ?? ?? ?? F3 0F 10 8F ?? ?? ?? ?? 0F 28 F8", DetourName = nameof(GetActionRangeDetour))]
private Hook<GetActionRangeDelegate>? GetActionRangeHook;

private float GetActionRangeDetour(uint actionID)
{
    var original = GetActionRangeHook.Original(actionID);
    return original + 2f;
}
```



## OnSendPacket / 本地对服务器发包

所有的网络活动都走此处上传至服务器, `a1` 为 `Opcode` , `a2` 和 `a3` 为数据包具体内容

```csharp
private delegate byte PacketDispatcher_OnSendPacket(nint a1, ushort* a2, nint a3, byte a4);
[Signature("48 89 5C 24 ?? 48 89 6C 24 ?? 48 89 74 24 ?? 57 41 56 41 57 48 83 EC 70 8B 81 ?? ?? ?? ??",
DetourName = nameof(PacketDispatcher_OnSendPacketDetour))]
internal Hook<PacketDispatcher_OnSendPacket>? PacketDispatcher_OnSendPacketHook;
```



## IsNoviceNetworkFlagSet / 检测新人频道特定标志设定情况

已知情况: `8U : 是否开启自动加入新人频道`

```csharp
private delegate bool IsNoviceNetworkFlagSetDelegate(PlayerState* instance, uint flag);
[Signature("8B C2 44 8B C2 C1 E8 ?? 4C 8B C9 83 F8 ?? 72 ?? 32 C0 C3 41 83 E0 ?? BA ?? ?? ?? ?? 41 0F B6 C8 D2 E2", ScanType = ScanType.Text)]
private static IsNoviceNetworkFlagSetDelegate? IsNoviceNetworkFlagSet;
```



## TryJoinNoviceNetwork / 尝试加入新人频道

```csharp
[StructLayout(LayoutKind.Explicit, Size = 40)]
private struct InfoProxyBeginner
{
    [FieldOffset(0)]
    public InfoProxyInterface InfoProxyInterface;

    [FieldOffset(24)]
    public bool IsInNoviceNetwork;

    public static InfoProxyBeginner* Instance() 
        => (InfoProxyBeginner*)InfoModule.Instance()->GetInfoProxyById((InfoProxyId)20);
}

private delegate byte TryJoinNoviceNetworkDelegate(InfoProxyBeginner* infoProxy20);
[Signature("E8 ?? ?? ?? ?? 45 33 F6 41 B4")]
private static TryJoinNoviceNetworkDelegate? TryJoinNoviceNetwork;
```



## IsPlayerOnDiving / 是否正在潜水

持续被调用, 如果返回 `true` 本地玩家将会显示为潜水状态动作, 同时从陆地上进入水下不再需要过图, 在水下交互后正式切换为水下状态

```csharp
private delegate bool IsPlayerOnDivingDelegate(nint a1);
[Signature("E8 ?? ?? ?? ?? 84 C0 74 ?? F3 0F 10 35 ?? ?? ?? ?? F3 0F 10 3D ?? ?? ?? ?? F3 44 0F 10 05", DetourName = nameof(IsPlayingOnDivingDetour))]
private static Hook<IsPlayerOnDivingDelegate>? IsPlayerOnDivingHook;
```



## LeveAllowances / 理符限额

```csharp
[Signature("88 05 ?? ?? ?? ?? 0F B7 41 06", ScanType = ScanType.StaticAddress)]
private static byte LeveAllowances;
```



## GameObjectRotate / 游戏物体旋转

主要用于固定面向

```csharp
private delegate void GameObjectRotateDelegate(GameObject* obj, float value);
[Signature("E8 ?? ?? ?? ?? 83 FE 4F", DetourName = nameof(GameObjectRotateDetour))]
private static Hook<GameObjectRotateDelegate>? GameObjectRotateHook;
```



## SetFocusByObjectID / 设置焦点目标

```csharp
private delegate void SetFocusTargetByObjectIDDelegate(TargetSystem* targetSystem, ulong objectID);
[Signature("E8 ?? ?? ?? ?? BA 0C 00 00 00 48 8D 0D", DetourName = nameof(SetFocusTargetByObjectID))]
private static Hook<SetFocusTargetByObjectIDDelegate>? SetFocusTargetByObjectIDHook;

private static void SetFocusTargetByObjectID(TargetSystem* targetSystem, ulong objectID)
{
    if (objectID == 0xE000_0000)
    {
        objectID = Service.Target.Target?.ObjectId ?? 0xE000_0000;
        FocusTarget = Service.Target.Target?.ObjectId;
    }
    else
        FocusTarget = Service.Target.Target.ObjectId;

    SetFocusTargetByObjectIDHook.Original(targetSystem, objectID);
}
```



## AbandonDuty / 退出任务

会受到任何可能的状态限制, `a1` 固定传入 `false`

```csharp
private delegate void AbandonDutyDelagte(bool a1);
[Signature("E8 ?? ?? ?? ?? 48 8B 43 28 B1 01")]
private static AbandonDutyDelagte? AbandonDuty;
```



## CountdownInit / 倒计时初始化

```csharp
public delegate nint CountdownInitDelegate(nint a1, nint a2);
[Signature("E9 ?? ?? ?? ?? 48 83 C4 ?? 5B C3 CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC 40 53 48 83 EC ?? 48 8B 0D", DetourName = nameof(CountdownInitDetour))]
private static Hook<CountdownInitDelegate>? CountdownInitHook;
```



## ReceiveAchievement / 接收成就数据

```csharp
private delegate void ReceiveAchievementProgressDelegate(Achievement* achievement, uint id, uint current, uint max);
[Signature("C7 81 ?? ?? ?? ?? ?? ?? ?? ?? 89 91 ?? ?? ?? ?? 44 89 81", DetourName = nameof(ReceiveAchievementProgressDetour))]
private static Hook<ReceiveAchievementProgressDelegate>? ReceiveAchievementProgressHook;
```



## CancelCast / 取消咏唱

实际游戏内可用于取消咏唱的函数和实际可利用的方式很多, 仅列出我最早拿到的一种

```csharp
[Signature("48 83 EC 38 33 D2 C7 44 24 20 00 00 00 00 45 33 C9")]
private static Action? CancelCast;
```



## ParseActionCommandArgument / 解析 ac 命令参数

实际解析诸如 `<t>` `<tt>` `<focus>` 之类参数的地方, 可以用来添加自定义命令参数

```csharp
private delegate GameObject* ParseActionCommandArgDelegate(nint a1, nint arg, bool a3, bool a4);
[Signature("E8 ?? ?? ?? ?? 4C 8B F8 49 B8", DetourName = nameof(ParseActionCommandArgDetour))]
private static Hook<ParseActionCommandArgDelegate>? ParseActionCommandArgHook;

private static unsafe GameObject* ParseActionCommandArgDetour(nint a1, nint arg, bool a3, bool a4)
{
    var original = ParseActionCommandArgHook.Original(a1, arg, a3, a4);

    var parsedArg = MemoryHelper.ReadSeStringNullTerminated(arg).TextValue;
    if (!parsedArg.Equals("<center>")) return original;

    IsNeedToModify = true;
    return Control.GetLocalPlayer();
}
```



## ReadyCheck / 准备确认

```c#
private delegate nint ReadyCheckDelegate(nint a1, byte a2, ulong contentID, ushort a4)
[Signature("E8 ?? ?? ?? ?? E9 ?? ?? ?? ?? FF 50 ?? 66 44 89 64 24", DetourName = nameof(ReadyCheckDetour))]
private static Hook<ReadyCheckDelegate>? ReadyCheckHook;
```

