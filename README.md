import discord
from discord.ext import commands
import random
import json
import os
import logging
import time

# ------------------- CONFIG -------------------
TOKEN = "YOUR_BOT_TOKEN_HERE"  # <-- Replace with your NEW token
DATA_FILE = "userdata.json"
ADMIN_ID = int(os.getenv("ADMIN_ID", "715309324754223105"))

# Cooldowns (in seconds)
CD = {
    "gamble": 10,
    "slots": 15,
    "daily": 24 * 3600,
    "claim": 5 * 3600,
}

logging.basicConfig(level=logging.INFO)
intents = discord.Intents.all()
bot = commands.Bot(command_prefix="!", intents=intents, help_command=None)

# ------------------- RINGS -------------------
RING_SHOP = {
    "silver":   {"cost": 500_000, "xp_mult": 1.0, "win_bonus": 0.00},
    "gold":     {"cost": 5_000_000, "xp_mult": 1.2, "win_bonus": 0.05},
    "diamond":  {"cost": 10_000_000, "xp_mult": 1.3, "win_bonus": 0.07},
    "platinum": {"cost": 25_000_000, "xp_mult": 1.6, "win_bonus": 0.10},
    "emerald":  {"cost": 75_000_000, "xp_mult": 2.1, "win_bonus": 0.12},
    "ruby":     {"cost": 500_000_000, "xp_mult": 3.0, "win_bonus": 0.15},
}
MAX_EQUIPPED_RINGS = 3

# ------------------- LEVEL PACKS -------------------
PACK_REWARDS = {1: 1, 5: 2, 12: 3, 18: 2, 24: 4, 30: 5}

# ------------------- DATA -------------------
if not os.path.exists(DATA_FILE):
    with open(DATA_FILE, "w") as f:
        json.dump({"users": {}}, f)

def load_data():
    try:
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {"users": {}}

def save_data():
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=2)

data = load_data()

def now():
    return int(time.time())

def get_user(uid: str):
    if uid not in data["users"]:
        data["users"][uid] = {
            "level": 1,
            "xp": 0,
            "coins": 1000,
            "packs": 0,
            "total_coins": 1000,
            "messages": 0,
            "rings_owned": [],
            "rings_equipped": [],
            "last_gamble": 0,
            "last_slots": 0,
            "last_daily": 0,
            "last_claim": 0,
            "casino_stats": {
                "total_winnings": 0,
                "total_losses": 0,
                "slots_played": 0,
                "coinflips_played": 0,
                "coinflips_won": 0
            }
        }
    return data["users"][uid]

def xp_req(level: int):
    base = 30
    increment = 25
    return base + increment * (level - 1) * level // 2

def get_xp_mult(user):
    mult = 1.0
    for r in user.get("rings_equipped", [])[:MAX_EQUIPPED_RINGS]:
        ring = RING_SHOP.get(r)
        if ring:
            mult += (ring["xp_mult"] - 1)
    return max(mult, 1.0)

def get_win_bonus(user):
    bonus = 0.0
    for r in user.get("rings_equipped", [])[:MAX_EQUIPPED_RINGS]:
        ring = RING_SHOP.get(r)
        if ring:
            bonus += ring.get("win_bonus", 0.0)
    return min(bonus, 0.40)

def on_cooldown(user, key):
    left = CD[key] - (now() - user.get(f"last_{key}", 0))
    return max(0, left)

# ------------------- EVENTS -------------------
@bot.event
async def on_ready():
    print(f"✅ Logged in as {bot.user}")

@bot.event
async def on_message(message: discord.Message):
    if message.author.bot:
        return
    uid = str(message.author.id)
    user = get_user(uid)
    user["messages"] += 1
    user["xp"] += int(1 * get_xp_mult(user))
    while user["xp"] >= xp_req(user["level"]):
        user["xp"] -= xp_req(user["level"])
        user["level"] += 1
        await message.channel.send(f"🎉 {message.author.mention} leveled up to **{user['level']}**!")
        if user["level"] in PACK_REWARDS:
            packs = PACK_REWARDS[user["level"]]
            user["packs"] += packs
            await message.channel.send(f"🎁 You earned **{packs} pack(s)**!")
    save_data()
    await bot.process_commands(message)

# ------------------- COMMANDS -------------------
@bot.command()
async def help(ctx):
    embed = discord.Embed(title="Bot Commands", color=0x5865F2)
    embed.add_field(name="Economy", value="`!balance`, `!daily`, `!claim`", inline=False)
    embed.add_field(name="Gambling", value="`!gamble <amount>`, `!slots <amount>`", inline=False)
    embed.add_field(name="Rings", value="`!shop`, `!buyring <name>`, `!equipring <name>`, `!rings`", inline=False)
    embed.add_field(name="Stats", value="`!inventory`, `!mystats`, `!leaderboard`", inline=False)
    await ctx.send(embed=embed)

# --- Balance ---
@bot.command(aliases=["bal"])
async def balance(ctx, member: discord.Member=None):
    member = member or ctx.author
    u = get_user(str(member.id))
    await ctx.send(f"💰 **{member.display_name}** has **{u['coins']:,}** coins.")

# --- Daily and Claim ---
@bot.command()
async def daily(ctx):
    u = get_user(str(ctx.author.id))
    left = on_cooldown(u, "daily")
    if left > 0:
        return await ctx.send(f"⏳ Daily in {left//3600}h {(left%3600)//60}m.")
    u["coins"] += 25_000
    u["total_coins"] += 25_000
    u["last_daily"] = now()
    save_data()
    await ctx.send(f"🗓️ Daily claimed: +25,000 coins! Balance: {u['coins']:,}")

@bot.command()
async def claim(ctx):
    u = get_user(str(ctx.author.id))
    left = on_cooldown(u, "claim")
    if left > 0:
        return await ctx.send(f"⏳ Claim in {left//3600}h {(left%3600)//60}m.")
    u["coins"] += 5_000
    u["total_coins"] += 5_000
    u["last_claim"] = now()
    save_data()
    await ctx.send(f"🪙 Claimed 5,000 coins! Balance: {u['coins']:,}")

# --- Gambling ---
BASE_LOSE_CHANCE = 0.30  # 30%

@bot.command(aliases=["cf", "coinflip", "coin"])
async def gamble(ctx, amount):
    u = get_user(str(ctx.author.id))
    left = on_cooldown(u, "gamble")
    if left > 0:
        return await ctx.send(f"⏳ Wait {left}s to gamble again.")
    try:
        amount = int(amount)
    except:
        return await ctx.send("❌ Invalid amount.")
    if amount <= 0 or amount > u["coins"]:
        return await ctx.send("❌ Not enough coins.")
    win_prob = min(0.99, 1 - BASE_LOSE_CHANCE + get_win_bonus(u))
    win = random.random() < win_prob
    u["casino_stats"]["coinflips_played"] += 1
    if win:
        u["coins"] += amount
        u["casino_stats"]["coinflips_won"] += 1
        await ctx.send(f"🎉 Won {amount:,}! Win chance: {win_prob*100:.1f}%")
    else:
        u["coins"] -= amount
        await ctx.send(f"❌ Lost {amount:,}. Win chance was {win_prob*100:.1f}%")
    u["last_gamble"] = now()
    save_data()

# --- Slots ---
SLOT_SYMBOLS = ["🍒", "🍋", "🔔", "⭐", "💎"]
SLOT_PAYOUTS = {"🍒": 2, "🍋": 3, "🔔": 5, "⭐": 8, "💎": 15}

@bot.command()
async def slots(ctx, amount: int):
    u = get_user(str(ctx.author.id))
    left = on_cooldown(u, "slots")
    if left > 0:
        return await ctx.send(f"⏳ Wait {left}s to use slots again.")
    if amount <= 0 or amount > u["coins"]:
        return await ctx.send("❌ Invalid bet or not enough coins.")
    r = [random.choice(SLOT_SYMBOLS) for _ in range(3)]
    msg = f"🎰 | {' '.join(r)}"
    if r[0] == r[1] == r[2]:
        win_amount = amount * SLOT_PAYOUTS[r[0]]
        u["coins"] += win_amount
        await ctx.send(f"{msg}\n🎉 Triple {r[0]}! Won {win_amount:,}")
    elif len(set(r)) == 2:
        win_amount = int(amount * 1.5)
        u["coins"] += win_amount
        await ctx.send(f"{msg}\n✅ Two of a kind! Won {win_amount:,}")
    else:
        u["coins"] -= amount
        await ctx.send(f"{msg}\n❌ No match. Lost {amount:,}")
    u["last_slots"] = now()
    save_data()

# --- Rings ---
@bot.command()
async def shop(ctx):
    embed = discord.Embed(title="💎 Ring Shop", color=0x3498db)
    for name, info in RING_SHOP.items():
        embed.add_field(
            name=name.capitalize(),
            value=f"Cost: {info['cost']:,} coins\nXP Multiplier: {info['xp_mult']}x\nWin Bonus: +{int(info['win_bonus']*100)}%",
            inline=False
        )
    await ctx.send(embed=embed)

@bot.command()
async def buyring(ctx, name: str):
    u = get_user(str(ctx.author.id))
    name = name.lower()
    ring = RING_SHOP.get(name)
    if not ring:
        return await ctx.send("❌ Ring not found.")
    if name in u["rings_owned"]:
        return await ctx.send("❌ Already own this ring.")
    if u["coins"] < ring["cost"]:
        return await ctx.send("❌ Not enough coins.")
    u["coins"] -= ring["cost"]
    u["rings_owned"].append(name)
    save_data()
    await ctx.send(f"✅ Bought **{name.capitalize()} Ring**.")

@bot.command()
async def equipring(ctx, name: str):
    u = get_user(str(ctx.author.id))
    name = name.lower()
    if name not in u["rings_owned"]:
        return await ctx.send("❌ You don't own that ring.")
    if name in u["rings_equipped"]:
        return await ctx.send("❌ Already equipped.")
    if len(u["rings_equipped"]) >= MAX_EQUIPPED_RINGS:
        return await ctx.send(f"❌ Max {MAX_EQUIPPED_RINGS} rings equipped.")
    u["rings_equipped"].append(name)
    save_data()
    await ctx.send(f"✅ Equipped **{name.capitalize()} Ring**.")

@bot.command()
async def rings(ctx):
    u = get_user(str(ctx.author.id))
    owned = ", ".join(r.capitalize() for r in u["rings_owned"]) or "None"
    equipped = ", ".join(r.capitalize() for r in u["rings_equipped"]) or "None"
    await ctx.send(f"💍 Owned: {owned}\n🧬 Equipped: {equipped}")

# --- Stats ---
@bot.command()
async def inventory(ctx):
    u = get_user(str(ctx.author.id))
    embed = discord.Embed(title=f"{ctx.author.display_name}'s Inventory", color=ctx.author.color)
    embed.add_field(name="Coins", value=f"{u['coins']:,}")
    embed.add_field(name="Packs", value=f"{u['packs']:,}")
    embed.add_field(name="Level", value=str(u["level"]))
    embed.add_field(name="XP", value=str(u["xp"]))
    embed.add_field(name="Rings", value=", ".join(r.capitalize() for r in u["rings_equipped"]) or "None", inline=False)
    await ctx.send(embed=embed)

@bot.command()
async def mystats(ctx):
    u = get_user(str(ctx.author.id))
    cs = u["casino_stats"]
    embed = discord.Embed(title=f"{ctx.author.display_name}'s Stats", color=0x2ecc71)
    embed.add_field(name="Level", value=str(u["level"]))
    embed.add_field(name="Messages", value=str(u["messages"]))
    embed.add_field(name="Total Coins", value=f"{u['total_coins']:,}")
    embed.add_field(name="Casino Wins/Losses", value=f"{cs['total_winnings']:,} / {cs['total_losses']:,}")
    await ctx.send(embed=embed)

@bot.command(aliases=["lb"])
async def leaderboard(ctx):
    users_sorted = sorted(data["users"].items(), key=lambda x: x[1]["coins"], reverse=True)[:10]
    desc = []
    for i, (uid, u) in enumerate(users_sorted, start=1):
        member = ctx.guild.get_member(int(uid))
        name = member.display_name if member else uid
        desc.append(f"**{i}.** {name} — {u['coins']:,} coins (Lv {u['level']})")
    embed = discord.Embed(title="🏆 Leaderboard (Coins)", description="\n".join(desc), color=0xf1c40f)
    await ctx.send(embed=embed)

# ------------------- RUN -------------------
if __name__ == "__main__":
    bot.run(TOKEN)