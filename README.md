khoivan273
==========
#region

using System;
using System.Collections.Generic;
using System.Linq;
using LeagueSharp;
using LeagueSharp.Common;
#endregion
namespace RefundExploiter
{
internal class Program
{
public static Menu Menu;
public static List<int> Consumables = new List<int> { 2003, 2004, 2009, 2010, 2037, 2039, 2043, 2044, 2047 };
public static List<int> NoCD = new List<int> { 3074, 3140, 3153, };
public static int RefundItemId = 0;
private static byte RefundInventorySlot;
public static bool Swapped = false;
public static bool Refunded = false;
public static bool BotrkInSlot = false;
private static void Main(string[] args)
{
CustomEvents.Game.OnGameLoad += Game_OnGameLoad;
}
private static void Game_OnGameLoad(EventArgs args)
{
Menu = new Menu("RefundExploiter", "RefundExploiter", true);
Menu.AddItem(new MenuItem("Enabled", "Enabled").SetValue(false));
Menu.AddItem(new MenuItem("Consumables", "Exploit Consumables").SetValue(true));
Menu.AddItem(new MenuItem("NoCD", "Exploit No-CD Items").SetValue(true));
Menu.AddItem(new MenuItem("Cast", "Cast").SetValue(new KeyBind(32, KeyBindType.Press)));
Menu.AddToMainMenu();
Game.OnGameProcessPacket += Game_OnGameProcessPacket;
Game.OnGameUpdate += Game_OnGameUpdate;
Game.PrintChat("Refund Exploit by Trees and blm95 loaded!");
//rito hire me pls
}
private static void Game_OnGameUpdate(EventArgs args)
{
if (!BotrkInSlot || !Menu.Item("Cast").GetValue<KeyBind>().Active)
{
return;
}
ObjectManager.Player.IssueOrder(GameObjectOrder.MoveTo, Game.CursorPos);
foreach (var p in
ObjectManager.Get<Obj_AI_Hero>()
.Where(h => h.IsValidTarget(500))
.Select(enemy => new Packet.C2S.Cast.Struct(enemy.NetworkId, SpellSlot.Trinket)))
{
Packet.C2S.Cast.Encoded(p).Send();
}
}
private static void Game_OnGameProcessPacket(GamePacketEventArgs args)
{
if (!Menu.Item("Enabled").GetValue<bool>())
{
return;
}
if (args.PacketData[0] == Packet.S2C.BuyItemAns.Header)
{
var dp = Packet.S2C.BuyItemAns.Decoded(args.PacketData);
if (dp.NetworkId != ObjectManager.Player.NetworkId || dp.SpellSlot == SpellSlot.Trinket)
{
return;
}
if ((Consumables.Contains(dp.Item.Id) && Menu.Item("Consumables").GetValue<bool>()))
{
Refunded = true;
Packet.C2S.Undo.Encoded().Send();
}
else if (NoCD.Contains(dp.Item.Id) && Menu.Item("NoCD").GetValue<bool>())
{
Refunded = true;
RefundItemId = dp.Item.Id;
RefundInventorySlot = dp.InventorySlot;
Packet.C2S.Undo.Encoded().Send();
}
}
else if (args.PacketData[0] == Packet.MultiPacket.Header &&
args.PacketData[5] == Packet.MultiPacket.UndoConfirm.SubHeader && Refunded)
{
Refunded = false;
if (RefundItemId == 0) // consumable item
{
args.Process = false;
return;
}
if (ObjectManager.Player.Spellbook.GetSpell(SpellSlot.Trinket).Name == "nospelldata")
{
SwapToTrinket();
return;
}
SellTrinket();
}
else if (args.PacketData[0] == Packet.S2C.SwapItemAns.Header && RefundItemId != 0 && Swapped)
{
var dp = Packet.S2C.SwapItemAns.Decoded(args.PacketData);
if (dp.ToInventorySlot == 6)
{
RecvItemToTrinket(RefundItemId);
RefundItemId = 0;
RefundInventorySlot = 0;
Swapped = false;
}
}
else if (args.PacketData[0] == Packet.S2C.SellItemAns.Header)
{
var dp = Packet.S2C.SellItemAns.Decoded(args.PacketData);
if (dp.SpellSlot == SpellSlot.Trinket)
{
SwapToTrinket();
}
}
}
