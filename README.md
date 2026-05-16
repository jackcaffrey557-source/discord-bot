import discord
from discord.ext import commands, tasks
from discord import app_commands
import json
import os
import asyncio
import random
from datetime import datetime, timedelta

# -----------------------------
# CONFIG
# -----------------------------

TOKEN = os.environ.get("DISCORD_TOKEN", "REDACTED")

LEVEL_CHANNEL_ID = 1432330332916547640

intents = discord.Intents.default()
intents.members = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)

# -----------------------------
# FILES
# -----------------------------
SPONSOR_FILE = "sponsors.json"
LEVEL_FILE = "levels.json"
WARN_FILE = "warnings.json"
STAFF_WARN_FILE = "staff_warnings.json"
CONFIG_FILE = "config.json"

# -----------------------------
# MOD ROLE NAMES (edit to match your server exactly)
# -----------------------------
ROLE_SR_MOD       = "Sr Mod"
ROLE_MOD          = "Mod"
ROLE_STAFF_TEAM   = "Staff Team"
ROLE_SENIOR_STAFF = "Senior Staff"
ROLE_MUTED        = "Muted"
ROLE_BOOSTER      = "Booster"

# -----------------------------
# GIVEAWAYS
# -----------------------------
ACTIVE_GIVEAWAYS = {}

# -----------------------------
# SPONSOR ROLES
# -----------------------------
SPONSOR_ROLES = {
    1_000_000: "1M Sponsor",
    10_000_000: "10M Sponsor",
    30_000_000: "30M Sponsor",
    50_000_000: "50M Sponsor",
    100_000_000: "100M Sponsor",
    500_000_000: "500M Sponsor",
    1_000_000_000: "1B Sponsor",
    2_000_000_000: "2B Sponsor",
    5_000_000_000: "5B Sponsor",
}

# -----------------------------
# LEVEL ROLES
# -----------------------------
LEVEL_ROLES = {
    1: "Level 1",
    5: "Level 5",
    10: "Level 10",
    20: "Level 20",
    30: "Level 30",
    40: "Level 40",
    50: "Level 50",
    60: "Level 60",
    70: "Level 70",
    80: "Level 80",
    90: "Level 90",
    100: "Level 100",
}

# -----------------------------
# FILE HELPERS
# -----------------------------
def load(file):
    if not os.path.exists(file):
        return {}
    with open(file) as f:
        return json.load(f)

def save(file, data):
    with open(file, "w") as f:
        json.dump(data, f, indent=4)

# -----------------------------
# MODERATION HELPERS
# -----------------------------
def member_has_role(member: discord.Member, role_name: str) -> bool:
    return any(r.name == role_name for r in member.roles)

def mod_check(role_name: str):
    async def predicate(interaction: discord.Interaction) -> bool:
        if not member_has_role(interaction.user, role_name):
            await interaction.response.send_message(
                f"❌ You need the **{role_name}** role to use this command.",
                ephemeral=True
            )
            return False
        return True
    return app_commands.check(predicate)

def min_role_id_check(min_role_id: int):
    """Allow members who have the given role ID or any role ranked higher."""
    async def predicate(interaction: discord.Interaction) -> bool:
        threshold = interaction.guild.get_role(min_role_id)
        if threshold is None:
            await interaction.response.send_message(
                "❌ Permission role not found in this server. Contact an admin.",
                ephemeral=True
            )
            return False
        # Member passes if any of their roles is at or above the threshold position
        if any(r.position >= threshold.position for r in interaction.user.roles):
            return True
        await interaction.response.send_message(
            f"❌ You need the **{threshold.name}** role or higher to use this command.",
            ephemeral=True
        )
        return False
    return app_commands.check(predicate)

def mod_or_sr_check():
    async def predicate(interaction: discord.Interaction) -> bool:
        if not (member_has_role(interaction.user, ROLE_MOD) or
                member_has_role(interaction.user, ROLE_SR_MOD)):
            await interaction.response.send_message(
                f"❌ You need the **{ROLE_MOD}** or **{ROLE_SR_MOD}** role to use this command.",
                ephemeral=True
            )
            return False
        return True
    return app_commands.check(predicate)

async def apply_mute(member: discord.Member, duration_seconds: int, reason: str):
    until = discord.utils.utcnow() + timedelta(seconds=duration_seconds)
    await member.timeout(until, reason=reason)

async def post_modlog(guild: discord.Guild, embed: discord.Embed):
    cfg = load(CONFIG_FILE)
    channel_id = cfg.get("modlog_channel_id")
    if not channel_id:
        return
    channel = guild.get_channel(int(channel_id))
    if channel:
        await channel.send(embed=embed)

# -----------------------------
# READY
# -----------------------------
import hashlib

def _commands_hash():
    names = sorted(c.name for c in bot.tree.get_commands())
    return hashlib.md5("|".join(names).encode()).hexdigest()

@bot.event
async def on_ready():
    bot.add_view(RolePanelView())  # re-register persistent view on every restart
    print(f"✅ Logged in as {bot.user}", flush=True)
    invite_url = (
        f"https://discord.com/api/oauth2/authorize"
        f"?client_id={bot.application_id}"
        f"&permissions=8"
        f"&scope=bot%20applications.commands"
    )
    print(f"Re-invite URL: {invite_url}", flush=True)

    # Only sync when commands have actually changed — avoids Discord rate limits
    current_hash = _commands_hash()
    cfg = load(CONFIG_FILE)
    last_hash = cfg.get("cmd_hash", "")

    if current_hash == last_hash:
        print(f"Commands unchanged ({len(bot.tree.get_commands())} total) — skipping sync", flush=True)
        return

    print(f"Command change detected — syncing {len(bot.tree.get_commands())} commands...", flush=True)
    try:
        for guild in bot.guilds:
            bot.tree.copy_global_to(guild=guild)
            cmds = await bot.tree.sync(guild=guild)
            print(f"  ✅ Guild '{guild.name}' — {len(cmds)} commands synced", flush=True)
        cfg["cmd_hash"] = current_hash
        save(CONFIG_FILE, cfg)
    except Exception as e:
        import traceback
        print(f"❌ Sync error: {e}", flush=True)
        traceback.print_exc()

# -----------------------------
# LEVEL SYSTEM
# -----------------------------
def get_level(xp):
    return int(xp ** 0.5 // 10)

async def update_level_roles(member, level):
    for lvl, role_name in LEVEL_ROLES.items():
        role = discord.utils.get(member.guild.roles, name=role_name)
        if role:
            if level >= lvl and role not in member.roles:
                await member.add_roles(role)

# -----------------------------
# BOOSTER ROLE — auto-assign on boost/unboost
# -----------------------------
@bot.event
async def on_member_update(before: discord.Member, after: discord.Member):
    # Detect boost start
    if before.premium_since is None and after.premium_since is not None:
        role = discord.utils.get(after.guild.roles, name=ROLE_BOOSTER)
        if role and role not in after.roles:
            await after.add_roles(role, reason="Server boost detected")
            print(f"🚀 Booster role given to {after}", flush=True)
    # Detect boost end
    elif before.premium_since is not None and after.premium_since is None:
        role = discord.utils.get(after.guild.roles, name=ROLE_BOOSTER)
        if role and role in after.roles:
            await after.remove_roles(role, reason="Server boost ended")
            print(f"Booster role removed from {after}", flush=True)

@bot.tree.command(name="syncboosters", description="Give the Booster role to all current server boosters (Sr Mod only)")
@min_role_id_check(1503100201257406705)
async def syncboosters(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    role = discord.utils.get(interaction.guild.roles, name=ROLE_BOOSTER)
    if not role:
        await interaction.followup.send(
            f"❌ No role named **{ROLE_BOOSTER}** found. Create it in your server roles first.",
            ephemeral=True
        )
        return

    boosters = [m for m in interaction.guild.members if m.premium_since]
    added = 0
    for member in boosters:
        if role not in member.roles:
            await member.add_roles(role, reason="Booster sync")
            added += 1

    await interaction.followup.send(
        f"✅ Sync complete — **{ROLE_BOOSTER}** role given to {added} new booster(s). "
        f"{len(boosters)} total current booster(s) in the server.",
        ephemeral=True
    )

# XP system
@bot.event
async def on_message(message):
    if message.author.bot:
        return

    data = load(LEVEL_FILE)
    uid = str(message.author.id)

    if uid not in data:
        data[uid] = {"xp": 0, "level": 0}

    data[uid]["xp"] += 5
    new_level = get_level(data[uid]["xp"])

    if new_level > data[uid]["level"]:
        data[uid]["level"] = new_level
        channel = bot.get_channel(LEVEL_CHANNEL_ID)
        if channel:
            await channel.send(
                f"📈 {message.author.mention} reached level **{new_level}**!"
            )
        await update_level_roles(message.author, new_level)

    save(LEVEL_FILE, data)
    await bot.process_commands(message)

# -----------------------------
# SPONSOR SYSTEM
# -----------------------------
async def update_sponsor_roles(member: discord.Member, total: float):
    for threshold in sorted(SPONSOR_ROLES.keys(), reverse=True):
        role_name = SPONSOR_ROLES[threshold]
        role = discord.utils.get(member.guild.roles, name=role_name)
        if role:
            if total >= threshold and role not in member.roles:
                await member.add_roles(role)
            elif total < threshold and role in member.roles:
                await member.remove_roles(role)

@bot.tree.command(name="sponsor_total", description="Check a user's sponsor total")
async def sponsor_total(interaction: discord.Interaction, user: discord.Member):
    data = load(SPONSOR_FILE)
    await interaction.response.send_message(
        f"{user} has ${data.get(str(user.id), 0):,}"
    )

@bot.tree.command(name="set_sponsor", description="Set a user's sponsor total and assign roles (admin only)")
@app_commands.checks.has_permissions(administrator=True)
async def set_sponsor(interaction: discord.Interaction, user: discord.Member, amount: float):
    data = load(SPONSOR_FILE)
    data[str(user.id)] = amount
    save(SPONSOR_FILE, data)
    await update_sponsor_roles(user, amount)
    await interaction.response.send_message(
        f"Set {user.mention}'s sponsor total to **${amount:,.0f}** and updated their roles.",
        ephemeral=True
    )

@bot.tree.command(name="add_sponsor", description="Add to a user's sponsor total and update roles (admin only)")
@app_commands.checks.has_permissions(administrator=True)
async def add_sponsor(interaction: discord.Interaction, user: discord.Member, amount: float):
    data = load(SPONSOR_FILE)
    current = data.get(str(user.id), 0)
    new_total = current + amount
    data[str(user.id)] = new_total
    save(SPONSOR_FILE, data)
    await update_sponsor_roles(user, new_total)
    await interaction.response.send_message(
        f"Added **${amount:,.0f}** to {user.mention}. New total: **${new_total:,.0f}**. Roles updated.",
        ephemeral=True
    )

@bot.tree.command(name="sponsor_leaderboard", description="Show the top sponsors in the server")
async def sponsor_leaderboard(interaction: discord.Interaction, top: int = 10):
    data = load(SPONSOR_FILE)
    if not data:
        await interaction.response.send_message("No sponsor data yet.", ephemeral=True)
        return

    sorted_sponsors = sorted(data.items(), key=lambda x: x[1], reverse=True)[:min(top, 25)]

    lines = []
    medals = {1: "🥇", 2: "🥈", 3: "🥉"}
    for rank, (user_id, total) in enumerate(sorted_sponsors, start=1):
        member = interaction.guild.get_member(int(user_id))
        name = member.display_name if member else f"User {user_id}"
        prefix = medals.get(rank, f"**#{rank}**")
        lines.append(f"{prefix} {name} — **${total:,.0f}**")

    embed = discord.Embed(
        title="💰 Sponsor Leaderboard",
        description="\n".join(lines),
        color=discord.Color.gold()
    )
    embed.set_footer(text=f"Top {len(sorted_sponsors)} sponsors")
    await interaction.response.send_message(embed=embed)

@bot.tree.command(name="sync_sponsor_roles", description="Scan all sponsor data and assign correct roles to everyone (admin only)")
@app_commands.checks.has_permissions(administrator=True)
async def sync_sponsor_roles(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    data = load(SPONSOR_FILE)
    if not data:
        await interaction.followup.send("No sponsor data found.", ephemeral=True)
        return

    updated = 0
    failed = 0
    for user_id, total in data.items():
        try:
            member = interaction.guild.get_member(int(user_id))
            if member is None:
                member = await interaction.guild.fetch_member(int(user_id))
            await update_sponsor_roles(member, total)
            updated += 1
        except Exception:
            failed += 1

    msg = f"Sync complete. Updated roles for **{updated}** member(s)."
    if failed:
        msg += f" Could not find **{failed}** member(s) (they may have left the server)."
    await interaction.followup.send(msg, ephemeral=True)

# -----------------------------
# GIVEAWAY SYSTEM
# -----------------------------
class GiveawayView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.entries = []

    @discord.ui.button(label="Enter Giveaway", style=discord.ButtonStyle.green, emoji="🎉")
    async def enter(self, interaction: discord.Interaction, button: discord.ui.Button):
        if interaction.user in self.entries:
            return await interaction.response.send_message("Already entered", ephemeral=True)
        self.entries.append(interaction.user)
        await interaction.response.send_message("Entered giveaway!", ephemeral=True)

@bot.tree.command(name="giveaway", description="Start a giveaway with live tracking")
async def giveaway(interaction: discord.Interaction, prize: str, minutes: int, winners: int):
    end_time = datetime.utcnow() + timedelta(minutes=minutes)
    view = GiveawayView()
    embed = discord.Embed(
        title="🎉 GIVEAWAY",
        description=f"**Prize:** {prize}\n**Winners:** {winners}",
        color=discord.Color.gold()
    )
    msg = await interaction.channel.send(embed=embed, view=view)
    ACTIVE_GIVEAWAYS[msg.id] = {
        "view": view,
        "end": end_time,
        "prize": prize,
        "winners": winners
    }
    await interaction.response.send_message("Giveaway started!", ephemeral=True)

    while True:
        if msg.id not in ACTIVE_GIVEAWAYS:
            break
        data = ACTIVE_GIVEAWAYS[msg.id]
        if datetime.utcnow() >= data["end"]:
            break
        remaining = data["end"] - datetime.utcnow()
        embed.description = (
            f"**Prize:** {data['prize']}\n"
            f"**Winners:** {data['winners']}\n"
            f"**Time Left:** {str(remaining).split('.')[0]}\n"
            f"**Entries:** {len(view.entries)}"
        )
        await msg.edit(embed=embed)
        await asyncio.sleep(20)

    if len(view.entries) == 0:
        return await interaction.channel.send("No entries.")
    winners_list = random.sample(view.entries, min(winners, len(view.entries)))
    await interaction.channel.send(
        "🎉 Winners: " + ", ".join(w.mention for w in winners_list)
    )

@bot.tree.command(name="giveaway_end", description="Force end a giveaway")
async def end(interaction: discord.Interaction):
    await interaction.response.send_message("Ended giveaway", ephemeral=True)

@bot.tree.command(name="giveaway_pause", description="Pause a giveaway")
async def pause(interaction: discord.Interaction, message_id: str):
    ACTIVE_GIVEAWAYS.pop(int(message_id), None)
    await interaction.response.send_message("Paused", ephemeral=True)

@bot.tree.command(name="giveaway_resume", description="Resume giveaway (restarts timer)")
async def resume(interaction: discord.Interaction):
    await interaction.response.send_message("Resume not needed (auto system handles timing)", ephemeral=True)

@bot.tree.command(name="reroll", description="Reroll giveaway winner")
async def reroll(interaction: discord.Interaction, message_id: str, winners: int = 1):
    data = ACTIVE_GIVEAWAYS.get(int(message_id))
    if not data:
        return await interaction.response.send_message("Not found", ephemeral=True)
    entries = data["view"].entries
    winner_list = random.sample(entries, min(winners, len(entries)))
    await interaction.channel.send(
        "🔁 Reroll: " + ", ".join(w.mention for w in winner_list)
    )
    await interaction.response.send_message("Done", ephemeral=True)

# -----------------------------
# /setmodlog — Sr Mod only
# -----------------------------
@bot.tree.command(name="setmodlog", description="Set the channel where all mod actions are logged")
@min_role_id_check(1503100201257406705)
async def setmodlog(interaction: discord.Interaction, channel: discord.TextChannel):
    cfg = load(CONFIG_FILE)
    cfg["modlog_channel_id"] = str(channel.id)
    save(CONFIG_FILE, cfg)
    await interaction.response.send_message(
        f"✅ Mod log channel set to {channel.mention}. All bans, mutes, warns, and staff warns will be posted there.",
        ephemeral=True
    )

# -----------------------------
# /ban — Sr Mod only
# -----------------------------
@bot.tree.command(name="ban", description="Ban a member from the server")
@min_role_id_check(1503100201257406705)
async def ban(interaction: discord.Interaction, user: discord.Member, reason: str = "No reason provided"):
    if user.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't ban someone with an equal or higher role.", ephemeral=True)
        return
    await user.ban(reason=f"Banned by {interaction.user} — {reason}")
    embed = discord.Embed(
        title="🔨 Member Banned",
        description=f"**User:** {user.mention}\n**Reason:** {reason}\n**Banned by:** {interaction.user.mention}",
        color=discord.Color.red()
    )
    embed.timestamp = discord.utils.utcnow()
    await interaction.response.send_message(embed=embed)
    await post_modlog(interaction.guild, embed)

# -----------------------------
# /timeout — Sr Mod only
# -----------------------------
@bot.tree.command(name="timeout", description="Timeout a member (duration in minutes)")
@min_role_id_check(1503100201257406705)
async def timeout_cmd(interaction: discord.Interaction, user: discord.Member, minutes: int, reason: str = "No reason provided"):
    if user.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't timeout someone with an equal or higher role.", ephemeral=True)
        return
    await apply_mute(user, minutes * 60, reason)
    embed = discord.Embed(
        title="⏱️ Member Timed Out",
        description=f"**User:** {user.mention}\n**Duration:** {minutes} minute(s)\n**Reason:** {reason}\n**By:** {interaction.user.mention}",
        color=discord.Color.orange()
    )
    embed.timestamp = discord.utils.utcnow()
    await interaction.response.send_message(embed=embed)
    await post_modlog(interaction.guild, embed)

# -----------------------------
# /mute — Sr Mod only (24h timeout)
# -----------------------------
@bot.tree.command(name="mute", description="Mute a member for 24 hours")
@min_role_id_check(1503100201257406705)
async def mute(interaction: discord.Interaction, user: discord.Member, reason: str = "No reason provided"):
    if user.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't mute someone with an equal or higher role.", ephemeral=True)
        return
    await apply_mute(user, 86400, reason)
    embed = discord.Embed(
        title="🔇 Member Muted",
        description=f"**User:** {user.mention}\n**Duration:** 24 hours\n**Reason:** {reason}\n**By:** {interaction.user.mention}",
        color=discord.Color.orange()
    )
    embed.timestamp = discord.utils.utcnow()
    await interaction.response.send_message(embed=embed)
    await post_modlog(interaction.guild, embed)

# -----------------------------
# /warn — Mod or Sr Mod
# -----------------------------
@bot.tree.command(name="warn", description="Warn a member (3 warns = 24h mute + Mod ping)")
@mod_or_sr_check()
async def warn(interaction: discord.Interaction, user: discord.Member, reason: str):
    if member_has_role(user, ROLE_STAFF_TEAM) or member_has_role(user, ROLE_MOD) or member_has_role(user, ROLE_SR_MOD):
        await interaction.response.send_message(
            "❌ Use `/staffwarn` for staff members.", ephemeral=True
        )
        return

    data = load(WARN_FILE)
    uid = str(user.id)
    if uid not in data:
        data[uid] = {"warns": [], "count": 0}

    data[uid]["warns"].append({
        "reason": reason,
        "by": str(interaction.user.id),
        "at": datetime.utcnow().isoformat()
    })
    data[uid]["count"] += 1
    count = data[uid]["count"]
    save(WARN_FILE, data)

    embed = discord.Embed(
        title="⚠️ Member Warned",
        description=f"**User:** {user.mention}\n**Reason:** {reason}\n**Total warns:** {count}/3\n**By:** {interaction.user.mention}",
        color=discord.Color.yellow()
    )
    embed.timestamp = discord.utils.utcnow()
    await interaction.response.send_message(embed=embed)
    await post_modlog(interaction.guild, embed)

    if count >= 3:
        await apply_mute(user, 86400, "Reached 3 warnings")
        mod_role = discord.utils.get(interaction.guild.roles, name=ROLE_MOD)
        ping = mod_role.mention if mod_role else f"@{ROLE_MOD}"
        await interaction.channel.send(
            f"🚨 {ping} — {user.mention} has reached **3 warnings** and has been **muted for 24 hours**. Please review their case."
        )
        auto_embed = discord.Embed(
            title="🔇 Auto-Mute (3 Warnings)",
            description=f"**User:** {user.mention}\n**Reason:** Reached 3 warnings\n**Duration:** 24 hours",
            color=discord.Color.dark_red()
        )
        auto_embed.timestamp = discord.utils.utcnow()
        await post_modlog(interaction.guild, auto_embed)

# -----------------------------
# /warnings — check a user's warns
# -----------------------------
@bot.tree.command(name="warnings", description="Check how many warnings a member has")
@mod_or_sr_check()
async def warnings(interaction: discord.Interaction, user: discord.Member):
    data = load(WARN_FILE)
    uid = str(user.id)
    entry = data.get(uid, {"warns": [], "count": 0})
    count = entry["count"]
    lines = []
    for i, w in enumerate(entry["warns"], 1):
        by_member = interaction.guild.get_member(int(w["by"]))
        by_name = by_member.display_name if by_member else "Unknown"
        lines.append(f"**{i}.** {w['reason']} — by {by_name}")

    embed = discord.Embed(
        title=f"⚠️ Warnings for {user.display_name}",
        description="\n".join(lines) if lines else "No warnings on record.",
        color=discord.Color.yellow()
    )
    embed.set_footer(text=f"Total: {count}/3")
    await interaction.response.send_message(embed=embed, ephemeral=True)

# -----------------------------
# /clearwarns — Sr Mod only
# -----------------------------
@bot.tree.command(name="clearwarns", description="Clear all warnings for a member")
@min_role_id_check(1503100201257406705)
async def clearwarns(interaction: discord.Interaction, user: discord.Member):
    data = load(WARN_FILE)
    uid = str(user.id)
    data[uid] = {"warns": [], "count": 0}
    save(WARN_FILE, data)
    await interaction.response.send_message(
        f"✅ Cleared all warnings for {user.mention}.", ephemeral=True
    )
    log_embed = discord.Embed(
        title="🗑️ Warnings Cleared",
        description=f"**User:** {user.mention}\n**Cleared by:** {interaction.user.mention}",
        color=discord.Color.green()
    )
    log_embed.timestamp = discord.utils.utcnow()
    await post_modlog(interaction.guild, log_embed)

# -----------------------------
# /staffwarn — Sr Mod only, warns Staff Team members
# -----------------------------
@bot.tree.command(name="staffwarn", description="Warn a Staff Team member (3 = Senior Staff ping)")
@min_role_id_check(1503100201257406705)
async def staffwarn(interaction: discord.Interaction, user: discord.Member, reason: str):
    if not member_has_role(user, ROLE_STAFF_TEAM):
        await interaction.response.send_message(
            f"❌ {user.mention} does not have the **{ROLE_STAFF_TEAM}** role.", ephemeral=True
        )
        return

    data = load(STAFF_WARN_FILE)
    uid = str(user.id)
    if uid not in data:
        data[uid] = {"warns": [], "count": 0}

    data[uid]["warns"].append({
        "reason": reason,
        "by": str(interaction.user.id),
        "at": datetime.utcnow().isoformat()
    })
    data[uid]["count"] += 1
    count = data[uid]["count"]
    save(STAFF_WARN_FILE, data)

    embed = discord.Embed(
        title="🛑 Staff Member Warned",
        description=f"**Staff:** {user.mention}\n**Reason:** {reason}\n**Total staff warns:** {count}/3\n**By:** {interaction.user.mention}",
        color=discord.Color.red()
    )
    embed.timestamp = discord.utils.utcnow()
    await interaction.response.send_message(embed=embed)
    await post_modlog(interaction.guild, embed)

    if count >= 3:
        senior_role = discord.utils.get(interaction.guild.roles, name=ROLE_SENIOR_STAFF)
        ping = senior_role.mention if senior_role else f"@{ROLE_SENIOR_STAFF}"
        await interaction.channel.send(
            f"🚨 {ping} — {user.mention} (Staff Team) has received **3 staff warnings**. Please review their conduct."
        )

# -----------------------------
# /staffwarnings — check a staff member's warns
# -----------------------------
@bot.tree.command(name="staffwarnings", description="Check staff warnings for a Staff Team member")
@min_role_id_check(1503100201257406705)
async def staffwarnings(interaction: discord.Interaction, user: discord.Member):
    data = load(STAFF_WARN_FILE)
    uid = str(user.id)
    entry = data.get(uid, {"warns": [], "count": 0})
    count = entry["count"]
    lines = []
    for i, w in enumerate(entry["warns"], 1):
        by_member = interaction.guild.get_member(int(w["by"]))
        by_name = by_member.display_name if by_member else "Unknown"
        lines.append(f"**{i}.** {w['reason']} — by {by_name}")

    embed = discord.Embed(
        title=f"🛑 Staff Warnings for {user.display_name}",
        description="\n".join(lines) if lines else "No staff warnings on record.",
        color=discord.Color.red()
    )
    embed.set_footer(text=f"Total: {count}/3")
    await interaction.response.send_message(embed=embed, ephemeral=True)

# -----------------------------
# /clearstaffwarns — Sr Mod only
# -----------------------------
@bot.tree.command(name="clearstaffwarns", description="Clear all staff warnings for a Staff Team member")
@min_role_id_check(1503100201257406705)
async def clearstaffwarns(interaction: discord.Interaction, user: discord.Member):
    data = load(STAFF_WARN_FILE)
    uid = str(user.id)
    data[uid] = {"warns": [], "count": 0}
    save(STAFF_WARN_FILE, data)
    await interaction.response.send_message(
        f"✅ Cleared all staff warnings for {user.mention}.", ephemeral=True
    )
    log_embed = discord.Embed(
        title="🗑️ Staff Warnings Cleared",
        description=f"**Staff:** {user.mention}\n**Cleared by:** {interaction.user.mention}",
        color=discord.Color.green()
    )
    log_embed.timestamp = discord.utils.utcnow()
    await post_modlog(interaction.guild, log_embed)

# -----------------------------
# /unban — Sr Mod only
# -----------------------------
@bot.tree.command(name="unban", description="Unban a user by their Discord user ID")
@min_role_id_check(1503100201257406705)
async def unban(interaction: discord.Interaction, user_id: str, reason: str = "No reason provided"):
    try:
        uid = int(user_id)
    except ValueError:
        await interaction.response.send_message("❌ Invalid user ID — must be a number.", ephemeral=True)
        return

    try:
        user = await bot.fetch_user(uid)
        await interaction.guild.unban(user, reason=f"Unbanned by {interaction.user} — {reason}")
        embed = discord.Embed(
            title="✅ Member Unbanned",
            description=f"**User:** {user.mention} (`{user}` — ID: `{uid}`)\n**Reason:** {reason}\n**Unbanned by:** {interaction.user.mention}",
            color=discord.Color.green()
        )
        embed.timestamp = discord.utils.utcnow()
        await interaction.response.send_message(embed=embed)
        await post_modlog(interaction.guild, embed)
    except discord.NotFound:
        await interaction.response.send_message(f"❌ No user found with ID `{user_id}`, or they are not banned.", ephemeral=True)
    except discord.Forbidden:
        await interaction.response.send_message("❌ I don't have permission to unban members.", ephemeral=True)

# -----------------------------
# /kick — Sr Mod only
# -----------------------------
@bot.tree.command(name="kick", description="Kick a member from the server")
@min_role_id_check(1503100201257406705)
async def kick(interaction: discord.Interaction, user: discord.Member, reason: str = "No reason provided"):
    if user.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't kick someone with an equal or higher role.", ephemeral=True)
        return
    await user.kick(reason=f"Kicked by {interaction.user} — {reason}")
    embed = discord.Embed(
        title="👢 Member Kicked",
        description=f"**User:** {user.mention}\n**Reason:** {reason}\n**Kicked by:** {interaction.user.mention}",
        color=discord.Color.orange()
    )
    embed.timestamp = discord.utils.utcnow()
    await interaction.response.send_message(embed=embed)
    await post_modlog(interaction.guild, embed)

# -----------------------------
# /banlist — Sr Mod only
# -----------------------------
@bot.tree.command(name="banlist", description="Show all currently banned users")
@min_role_id_check(1503100201257406705)
async def banlist(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    bans = [entry async for entry in interaction.guild.bans()]
    if not bans:
        await interaction.followup.send("✅ No users are currently banned.", ephemeral=True)
        return

    lines = []
    for entry in bans[:25]:
        reason = entry.reason or "No reason provided"
        lines.append(f"• **{entry.user}** (`{entry.user.id}`) — {reason}")

    embed = discord.Embed(
        title=f"🔨 Ban List ({len(bans)} total)",
        description="\n".join(lines),
        color=discord.Color.red()
    )
    if len(bans) > 25:
        embed.set_footer(text=f"Showing 25 of {len(bans)} banned users")
    else:
        embed.set_footer(text="Use /unban <user_id> to reverse a ban")
    embed.timestamp = discord.utils.utcnow()
    await interaction.followup.send(embed=embed, ephemeral=True)

# -----------------------------
# NOTIFICATION ROLE NAMES
# -----------------------------
ROLE_LIVES           = "lives"
ROLE_SPAWNER_UPDATES = "Spawner Updates"

# -----------------------------
# ROLE PANEL VIEW (persistent)
# -----------------------------
class RolePanelView(discord.ui.View):
    """Persistent view — survives bot restarts via custom_id."""

    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(
        label="🔴 LIVES",
        style=discord.ButtonStyle.danger,
        custom_id="role_panel:lives"
    )
    async def lives_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        await _toggle_role(interaction, ROLE_LIVES)

    @discord.ui.button(
        label="🟢 SPAWNER UPDATES",
        style=discord.ButtonStyle.success,
        custom_id="role_panel:spawner_updates"
    )
    async def spawner_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        await _toggle_role(interaction, ROLE_SPAWNER_UPDATES)


async def _toggle_role(interaction: discord.Interaction, role_name: str):
    """Add the role if the member doesn't have it, remove if they do."""
    role = discord.utils.get(interaction.guild.roles, name=role_name)
    if role is None:
        await interaction.response.send_message(
            f"❌ Role **{role_name}** not found. Ask an admin to check the role name.",
            ephemeral=True
        )
        return

    member = interaction.user
    if role in member.roles:
        await member.remove_roles(role, reason="Role panel toggle")
        await interaction.response.send_message(
            f"✅ Removed **{role.name}** — you'll no longer receive these notifications.",
            ephemeral=True
        )
    else:
        await member.add_roles(role, reason="Role panel toggle")
        await interaction.response.send_message(
            f"✅ Gave you **{role.name}** — you'll now receive these notifications!",
            ephemeral=True
        )


# -----------------------------
# /rolemessage — Sr Mod only
# -----------------------------
@bot.tree.command(name="rolemessage", description="Post the notification role picker panel in this channel")
@min_role_id_check(1503100201257406705)
async def rolemessage(interaction: discord.Interaction):
    embed = discord.Embed(
        title="🔔 Notification Roles",
        description=(
            "Pick the notifications you want to receive.\n"
            "Click a button to **get or remove** the role anytime.\n\u200b"
        ),
        color=discord.Color.from_str("#f5a623")
    )
    embed.add_field(
        name="🔴  LIVES",
        value="Get notified when we go live",
        inline=False
    )
    embed.add_field(
        name="🟢  SPAWNER UPDATES",
        value="Get notified when our spawner prices change",
        inline=False
    )
    embed.set_footer(text="Donut Loot Lounge • Notification Centre")

    await interaction.response.send_message(embed=embed, view=RolePanelView())


# -----------------------------
# LEVEL CHECK COMMAND
# -----------------------------
@bot.tree.command(name="level", description="Check your level")
async def level(interaction: discord.Interaction, user: discord.Member = None):
    if not user:
        user = interaction.user
    data = load(LEVEL_FILE)
    user_data = data.get(str(user.id), {"xp": 0, "level": 0})
    await interaction.response.send_message(
        f"{user} | Level {user_data['level']} | XP {user_data['xp']}"
    )

# -----------------------------
# /add_xp — min role ID required
# -----------------------------
@bot.tree.command(name="add_xp", description="Add XP to a member")
@min_role_id_check(1503100201257406705)
async def add_xp(interaction: discord.Interaction, user: discord.Member, amount: int):
    if amount <= 0:
        await interaction.response.send_message("❌ Amount must be a positive number.", ephemeral=True)
        return
    data = load(LEVEL_FILE)
    uid = str(user.id)
    entry = data.get(uid, {"xp": 0, "level": 0})
    entry["xp"] += amount
    entry["level"] = get_level(entry["xp"])
    data[uid] = entry
    save(LEVEL_FILE, data)
    await update_level_roles(user, entry["level"])
    embed = discord.Embed(
        title="✅ XP Added",
        description=(
            f"**{user.mention}** received **+{amount:,} XP**\n"
            f"Now at **{entry['xp']:,} XP** — Level **{entry['level']}**"
        ),
        color=discord.Color.green()
    )
    embed.set_footer(text=f"Added by {interaction.user}")
    await interaction.response.send_message(embed=embed)


# -----------------------------
# /remove_xp — min role ID required
# -----------------------------
@bot.tree.command(name="remove_xp", description="Remove XP from a member")
@min_role_id_check(1503100201257406705)
async def remove_xp(interaction: discord.Interaction, user: discord.Member, amount: int):
    if amount <= 0:
        await interaction.response.send_message("❌ Amount must be a positive number.", ephemeral=True)
        return
    data = load(LEVEL_FILE)
    uid = str(user.id)
    entry = data.get(uid, {"xp": 0, "level": 0})
    entry["xp"] = max(0, entry["xp"] - amount)
    entry["level"] = get_level(entry["xp"])
    data[uid] = entry
    save(LEVEL_FILE, data)
    embed = discord.Embed(
        title="✅ XP Removed",
        description=(
            f"**{user.mention}** lost **-{amount:,} XP**\n"
            f"Now at **{entry['xp']:,} XP** — Level **{entry['level']}**"
        ),
        color=discord.Color.orange()
    )
    embed.set_footer(text=f"Removed by {interaction.user}")
    await interaction.response.send_message(embed=embed)


# -----------------------------
# /disclaimer — min role ID required
# -----------------------------
@bot.tree.command(name="disclaimer", description="Post the server disclaimer in this channel")
@min_role_id_check(1503100201257406705)
async def disclaimer(interaction: discord.Interaction):
    msg = (
        "# DISCLAIMER\n"
        "We DO NOT own any of these schematics and take no credit for them at all. "
        "We give full credit to designers.\n\n"
        "If you want to join some <#1432330332706705579>!!\n"
        "If you boost the server you gain access to <@&1503794848724029591>!"
    )
    await interaction.response.send_message(msg)


# -----------------------------
# RUN BOT
# -----------------------------
bot.run(TOKEN)
