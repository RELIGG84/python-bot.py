[python bot.py](https://github.com/user-attachments/files/22060883/python.bot.py)
import discord
import requests
import json
import random
import os
import asyncio
from discord.ui import Button, View
from datetime import datetime, timedelta

# Importimi i librarive të nevojshme
try:
    import youtube_dl
    import nacl
except ImportError:
    print("Nuk u gjetën libraritë e nevojshme. Po instaloj...")
    os.system('pip install youtube_dl PyNaCl')
    print("Instalimi përfundoi. Ju lutem, ridizni botin.")
    exit()

#----------------------------------------------------
# TË DHËNAT TUAJA PERSONALE (PLOTËSOJINI TË GJITHA!)
#----------------------------------------------------
# Këtu duhet të vendosni token-in tuaj personal të botit
TOKEN = 'MTQxMDc0MzU0OTAxNTg4ODA2NA.GAc08J.WinXMh5HcfwWoTy2jL0Pk4zxDPo1TLv8uYOx00'

# ID-të e kanaleve në serverin tënd
WELCOME_CHANNEL_ID = 1073356064164155515
LOGGING_CHANNEL_ID = 1411378530918469682
ROLE_SELECTION_CHANNEL_ID = 1411378306519007323
ANNOUNCEMENT_CHANNEL_ID = 1411378627400171581
MODERATION_CHANNEL_ID = 1411378717179252866
VERIFICATION_CHANNEL_ID = 1411378187249647616
LFG_CHANNEL_ID = 1411378895554347230
TICKET_CHANNEL_ID = 1411378431752536185
MAP_CHANNEL_ID = 1411385917444456652
GIVEAWAY_CHANNEL_ID = 1073356074108850268
BOOSTS_CHANNEL_ID = 1073367317188186234
STW_CHANNEL_ID = 1073366235460407416
TRADE_CHAT_CHANNEL_ID = 1073365998759071866
BUILD_CHAT_CHANNEL_ID = 1073366134620958771
DAILY_MISSIONS_CHANNEL_ID = 1073366475093573662
STAFF_CHAT_CHANNEL_ID = 1073356089900400672
BUMP_CHANNEL_ID = 1242464414603673620
BOT_COMMANDS_CHANNEL_ID = 1073356086247166052
PROMOTIONS_CHANNEL_ID = 1085828818314473492

# ID-ja e botit të bump-it (p.sh. Disboard). Gjejeni ID-në e botit tuaj dhe vendoseni këtu.
BUMP_BOT_ID = VENDOS_ID_E_BOTIT_BUMP_KETU

# Të dhënat e krijuesit të serverit
CREATOR_CODE = "RELI"
CREATOR_NAME = "RELI"

# KODI I MAPIT TUAJ TË FORTNITE
MAP_CODE = "7188-6091-4920 https://fortnite.gg/island?code=7188-6091-4920"

# LINQET TUAJA TË RRJETEVE SOCIALE
TIKTOK_LINK = "https://www.tiktok.com/@religg"
INSTAGRAM_LINK = "https://www.instagram.com/reli.88888/"
TWITCH_LINK = "https://www.twitch.tv/religg50"
YOUTUBE_LINK = "https://www.youtube.com/@RELI-GG"

# LISTA E FJALËVE OFENDUESE NË SHQIP (Shto/Fshij sipas nevojës)
OFFENSIVE_WORDS = [
    "robt", "mut", "kac", "kurv", "peder", "pidh", "bysh", "byth", "qif", "qiell",
    "bitch", "fuck", "asshole", "pederast", "drogaxhi", "rober"
]

# API-ja publike e Fortnite
FORTNITE_SHOP_API = "https://fortnite-api.com/v2/shop/br"
FORTNITE_STATS_API = "https://fortnite-api.com/v2/stats/br/v2"
FORTNITE_STW_API = "https://fortnite-api.com/v1/stw/missions"

# Ruajtja e të dhënave të promocionit
last_promotion_time = {}

#----------------------------------------------------
# KLASA DHE FUNKSIONET KRYESORE TË BOTIT
#----------------------------------------------------

intents = discord.Intents.default()
intents.members = True
intents.message_content = True
intents.reactions = True

client = discord.Client(intents=intents)
tree = discord.app_commands.CommandTree(client)

# Emri i skedarit ku do të ruhen emrat e përdoruesve
USERS_FILE = 'users.json'
if not os.path.exists(USERS_FILE):
    with open(USERS_FILE, 'w') as f:
        json.dump({}, f)

# Funksioni për të llogaritur nivelin
def get_level(xp):
    level = 0
    while get_required_xp(level + 1) <= xp:
        level += 1
    return level

# Funksioni për të llogaritur pikët e nevojshme për nivelin tjetër
def get_required_xp(level):
    return 5 * (level ** 2) + 50 * level + 100

# OPSIONET PËR MUZIKËN
youtube_dl.utils.bug_reports_message = lambda: ''
ytdl_format_options = {
    'format': 'bestaudio/best',
    'outtmpl': '%(extractor)s-%(id)s-%(title)s.%(ext)s',
    'restrictfilenames': True,
    'noplaylist': True,
    'nocheckcertificate': True,
    'ignoreerrors': False,
    'logtostderr': False,
    'quiet': True,
    'no_warnings': True,
    'default_search': 'auto',
    'source_address': '0.0.0.0'
}
ffmpeg_options = {'options': '-vn'}
ytdl = youtube_dl.YoutubeDL(ytdl_format_options)

class YTDLSource(discord.PCMVolumeTransformer):
    def __init__(self, source, *, data, volume=0.5):
        super().__init__(source, volume)
        self.data = data
        self.title = data.get('title')
        self.url = data.get('url')
    @classmethod
    async def from_url(cls, url, *, loop=None, stream=False):
        loop = loop or asyncio.get_event_loop()
        data = await loop.run_in_executor(None, lambda: ytdl.extract_info(url, download=not stream))
        if 'entries' in data:
            data = data['entries'][0]
        filename = data['url'] if stream else ytdl.prepare_filename(data)
        return cls(discord.FFmpegPCMAudio(filename, **ffmpeg_options), data=data)

# KLASA E BUTONAVE PËR VERIFIKIM
class VerifyView(View):
    def __init__(self, bot_client):
        super().__init__(timeout=None)
        self.bot_client = bot_client
        
    @discord.ui.button(label="Verifiko Tani!", style=discord.ButtonStyle.green, custom_id="verify_button")
    async def verify_button_callback(self, interaction: discord.Interaction, button: Button):
        member = interaction.user
        guild = interaction.guild
        
        verified_role = discord.utils.get(guild.roles, name=DEFAULT_ROLE_NAME)
        unverified_role = discord.utils.get(guild.roles, name=UNVERIFIED_ROLE_NAME)
        
        if not verified_role or not unverified_role:
            await interaction.response.send_message("Ndodhi një gabim me rolet. Ju lutem kontaktoni administratorin.", ephemeral=True)
            return

        if verified_role in member.roles:
            await interaction.response.send_message("Ju jeni tashmë i verifikuar!", ephemeral=True)
            return
            
        try:
            await member.add_roles(verified_role)
            await member.remove_roles(unverified_role)
            
            dm_embed = discord.Embed(
                title=f"🎉 Mirë se erdhe në server, {member.name}! 🎉",
                description=f"Mirë se erdhe në familjen tonë! Tani ke akses të plotë në të gjitha kanalet. ❤️\n\n"
                              f"Për të na mbështetur, përdor kodin e krijuesit **{CREATOR_CODE}** në dyqanin e Fortnite! 🌟\n\n"
                              f"Nëse ke bërë subscribe/follow, përdor komandën `/prove-follow` për të marrë një rol special në server! ✨",
                color=discord.Color.red()
            )
            dm_embed.add_field(name="Më ndiq në rrjete sociale:", value=f"**YouTube:** {YOUTUBE_LINK}\n**TikTok:** {TIKTOK_LINK}\n**Twitch:** {TWITCH_LINK}\n**Instagram:** {INSTAGRAM_LINK}", inline=False)
            dm_embed.set_footer(text="Shpresojmë të kënaqesh në server!")

            try:
                await member.send(embed=dm_embed)
            except discord.Forbidden:
                pass # Nuk mund të dërgoj mesazh privat

            await interaction.response.send_message("Jeni verifikuar me sukses! Udhëzimet e tjera i ke në mesazhin privat.", ephemeral=True)
            
        except discord.Forbidden:
            await interaction.response.send_message("Nuk kam leje për të ndryshuar rolet. Ju lutem, kontrolloni lejet e mia.", ephemeral=True)
        except Exception as e:
            await interaction.response.send_message(f"Ndodhi një gabim: {e}", ephemeral=True)

# KLASA E BUTONAVE PËR ZGJEDHJEN E ROLEVE
class RoleSelectionView(View):
    def __init__(self, roles_data):
        super().__init__(timeout=None)
        for role_data in roles_data:
            button = Button(label=role_data['name'], style=discord.ButtonStyle.secondary, emoji=role_data['emoji'], custom_id=role_data['name'])
            button.callback = self.button_callback
            self.add_item(button)
    async def button_callback(self, interaction: discord.Interaction):
        role_name = interaction.item.custom_id
        guild = interaction.guild
        member = interaction.user
        role = discord.utils.get(guild.roles, name=role_name)
        if role is None:
            await interaction.response.send_message(f"Roli '{role_name}' nuk u gjet. Lajmëroni administratorin.", ephemeral=True)
            return
        try:
            await member.add_roles(role)
            await interaction.response.send_message(f"Roli **{role.name}** ju dha me sukses!", ephemeral=True)
        except discord.Forbidden:
            await interaction.response.send_message("Nuk kam leje për të dhënë këtë rol. Ju lutemi kontrolloni lejet e mia.", ephemeral=True)
        except Exception as e:
            await interaction.response.send_message(f"Ndodhi një gabim: {e}", ephemeral=True)

# KLASA PËR KËRKIM LOJTARËSH
class LFGView(View):
    def __init__(self, creator_id, game, players_needed):
        super().__init__(timeout=3600)  # Skadon pas 1 ore
        self.creator_id = creator_id
        self.game = game
        self.players_needed = players_needed
        self.players_joined = 0

    @discord.ui.button(label="Bashkohu!", style=discord.ButtonStyle.green)
    async def join_button_callback(self, interaction: discord.Interaction, button: Button):
        if self.players_joined >= self.players_needed:
            await interaction.response.send_message("Grupi tashmë është plot!", ephemeral=True)
            return
        
        creator = interaction.guild.get_member(self.creator_id)
        if not creator:
            await interaction.response.send_message("Lojtari që nisi kërkesën u largua.", ephemeral=True)
            return
            
        self.players_joined += 1
        
        await interaction.response.send_message(f"{interaction.user.mention} u bashkua me sukses!", ephemeral=True)
        
        if self.players_joined >= self.players_needed:
            await interaction.message.edit(content=f"**GRUPI ËSHTË PLOT!** 🥳", view=None)
            await creator.send(f"Hey {creator.mention}, grupi juaj për **{self.game}** është plot! Filloni të luani!")
        else:
            await interaction.message.edit(content=f"Kërkohen {self.players_needed - self.players_joined} lojtarë të tjerë për **{self.game}**!", view=self)

# EVENTS DHE FUNKSIONET KRYESORE
#----------------------------------------------------
@client.event
async def on_ready():
    await tree.sync()
    print(f'Boti {client.user} u lidh me sukses dhe është gati!')
    client.loop.create_task(check_inactivity())
    client.loop.create_task(send_daily_shop())
    client.loop.create_task(set_channel_permissions())

async def set_channel_permissions():
    """Vendos lejet e duhura për kanalet e specializuara."""
    await client.wait_until_ready()
    try:
        guild = client.get_guild(1411378306519007323) # Këtu vendos ID-në e serverit tuaj
        if not guild:
            print("Serveri nuk u gjet. Ju lutem sigurohuni që ID-ja e serverit të jetë e saktë.")
            return

        # Zgjidh rolet e nevojshme
        everyone_role = guild.default_role
        
        # Vendosni këtu emrin e rolit që ka leje të plotë
        full_access_role = discord.utils.get(guild.roles, name="Verified") 
        if not full_access_role:
            print("Roli 'Verified' nuk u gjet. Kontrolloni emrin e rolit.")
            return

        # Lista e ID-ve të kanaleve që do të kenë leje të kufizuara
        channels_to_restrict = [
            TRADE_CHAT_CHANNEL_ID,
            BUILD_CHAT_CHANNEL_ID,
            GIVEAWAY_CHANNEL_ID,
            PROMOTIONS_CHANNEL_ID
        ]

        for channel_id in channels_to_restrict:
            channel = guild.get_channel(channel_id)
            if channel:
                # Vendos leje për @everyone - DENY
                await channel.set_permissions(everyone_role, send_messages=False)
                # Vendos leje për rolin e caktuar - ALLOW
                await channel.set_permissions(full_access_role, send_messages=True)
                print(f"Lejet për kanalin {channel.name} u përditësuan me sukses.")
            else:
                print(f"Kanali me ID {channel_id} nuk u gjet.")
    except Exception as e:
        print(f"Ndodhi një gabim gjatë vendosjes së lejeve: {e}")


@client.event
async def on_member_join(member):
    unverified_role = discord.utils.get(member.guild.roles, name=UNVERIFIED_ROLE_NAME)
    if unverified_role:
        await member.add_roles(unverified_role)
    else:
        print(f'Roli "{UNVERIFIED_ROLE_NAME}" nuk u gjet.')
    
    welcome_channel = client.get_channel(VERIFICATION_CHANNEL_ID)
    if welcome_channel:
        await welcome_channel.send(f"Mirë se erdhe në server, **{member.mention}**! Për të vazhduar, shko në mesazhin tënd privat dhe ndiq udhëzimet. ❤️", delete_after=10)

@client.event
async def on_member_remove(member):
    logging_channel = client.get_channel(LOGGING_CHANNEL_ID)
    if logging_channel:
        await logging_channel.send(f"**{member.name}** u largua nga serveri. Na mbetet një anëtar më pak. 😢")

@client.event
async def on_member_update(before, after):
    # Kontrollon nëse një anëtar ka bërë boost serverin
    if not before.premium_since and after.premium_since:
        boosts_channel = client.get_channel(BOOSTS_CHANNEL_ID)
        if boosts_channel:
            embed = discord.Embed(
                title="🚀 Server Boost!",
                description=f"Faleminderit shumë **{after.mention}** që bëre boost serverin! Vlerësojmë mbështetjen tuaj! ✨",
                color=discord.Color.magenta()
            )
            embed.set_thumbnail(url=after.avatar.url)
            await boosts_channel.send(embed=embed)


@client.event
async def on_message(message):
    if message.author.bot:
        return

    # Funksionaliteti i bump-it
    if message.author.id == BUMP_BOT_ID and message.channel.id == BUMP_CHANNEL_ID:
        # Përshtateni tekstin e mesazhit sipas botit që përdorni (p.sh., Disboard)
        if "Bump done" in message.embeds[0].description:
            await asyncio.sleep(7200) # Prit 2 orë (7200 sekonda)
            bump_channel = client.get_channel(BUMP_CHANNEL_ID)
            if bump_channel:
                embed = discord.Embed(
                    title="⏰ Koha për të bumbuar!",
                    description=f"Është koha për të bumbuar serverin përsëri! Kliko <#{BUMP_CHANNEL_ID}> dhe shkruaj `/bump`.\n"
                                  "Çdo bump ndihmon komunitetin të rritet!",
                    color=discord.Color.red()
                )
                await bump_channel.send(f"@here", embed=embed)
    
    # Fshirja e komandave jashtë kanalit "bot-commands"
    if message.content.startswith("/") and message.channel.id != BOT_COMMANDS_CHANNEL_ID:
        try:
            await message.delete()
            await message.channel.send(f"Përshëndetje {message.author.mention}, të gjitha komandat e botit mund të përdoren në kanalin <#{BOT_COMMANDS_CHANNEL_ID}>.", delete_after=10)
        except discord.Forbidden:
            print("Nuk kam leje për të fshirë mesazhe.")
            
    user_id = str(message.author.id)
    with open(USERS_FILE, 'r') as f:
        users = json.load(f)
    
    if user_id not in users:
        users[user_id] = {"ban_warnings": 0, "last_message": None, "xp": 0}

    # Shto pikë për çdo mesazh (10 pikë per mesazh)
    users[user_id]["xp"] = users[user_id].get("xp", 0) + 10
    
    message_content_lower = message.content.lower()
    is_offensive = any(word in message_content_lower for word in OFFENSIVE_WORDS)
    
    if is_offensive:
        try:
            # Kontrollon nëse përdoruesi është administrator
            if message.author.guild_permissions.administrator:
                await message.author.send("Përdorimi i fjalëve ofenduese nuk lejohet, por si administrator nuk do të ndëshkoheni nga boti.")
                return

            await message.delete()
            member = message.author
            
            users[user_id]["ban_warnings"] = users[user_id].get("ban_warnings", 0) + 1
            
            warnings_count = users[user_id]["ban_warnings"]
            logging_channel = client.get_channel(LOGGING_CHANNEL_ID)
            
            if warnings_count < 3:
                dm_message = f"**Paralajmërim:** Mesazhi juaj u fshi pasi përdorët gjuhë të papërshtatshme. Ky është paralajmërimi juaj i **{warnings_count}**-të. Pas 3 shkeljeve, do të ndaloheni përgjithmonë nga serveri."
                await member.send(dm_message)
                if logging_channel:
                    await logging_channel.send(f"Anëtari **{member.name}** mori paralajmërimin e **{warnings_count}**-të për fjalë ofenduese.")
            else:
                try:
                    await member.ban(reason="Përdorim i përsëritur i fjalëve ofenduese.")
                    await member.send("Ju jeni ndaluar përgjithmonë nga serveri për përdorim të përsëritur të fjalëve ofenduese.")
                    if logging_channel:
                        await logging_channel.send(f"Anëtari **{member.name}** u ndalua (ban) përgjithmonë për shkak të 3 paralajmërimave.")
                    del users[user_id]
                except discord.Forbidden:
                    await member.send("Gabim: Nuk kam leje për t'ju ndaluar. Ju lutem, kontaktoni një administrator.")
            
            with open(USERS_FILE, 'w') as f:
                json.dump(users, f, indent=4)
                
    with open(USERS_FILE, 'w') as f:
        json.dump(users, f, indent=4)

# KOMANDAT PËR SERVERIN
#----------------------------------------------------
@tree.command(name="ping", description="Shfaq vonesën e botit (latency).")
async def ping_command(interaction: discord.Interaction):
    await interaction.response.send_message(f"Ping! Latenca e botit është **{round(client.latency * 1000)}ms**.")

@tree.command(name="gaming-inspo", description="Merr një mesazh motivues ose një këshillë për gaming.")
async def gaming_inspo_command(interaction: discord.Interaction):
    inspo_messages = [
        "Jepuni çdo loje një shans. Mjeshtëria vjen me praktikën. ✨",
        "Mos u dekurajoni nga humbjet, ato janë thjesht hapa drejt fitores së ardhshme! 🏆",
        "Mos harro të bësh pushim! Lojërat janë më të mira kur je i freskët. 🎮",
        "Çdo lojtar ka një stil unik. Gjeje tëndin dhe përqafoje! 🔥",
        "Loja nuk është vetëm për fitore. Është për të shijuar udhëtimin dhe për të kaluar mirë me shokët."
    ]
    await interaction.response.send_message(random.choice(inspo_messages), ephemeral=True)

@tree.command(name="map-code", description="Shfaq kodin e mapit të Fortnite të serverit.")
async def map_code_command(interaction: discord.Interaction):
    embed = discord.Embed(
        title="Kodi i Mapit të Fortnite",
        description=f"Kodi i mapit të Fortnite të {CREATOR_NAME} është **{MAP_CODE}**\n\n**Më mbështet duke përdorur kodin e krijuesit: `{CREATOR_CODE}`**",
        color=discord.Color.blue()
    )
    embed.set_thumbnail(url="https://cdn.discordapp.com/attachments/1073356064164155515/1073357597148567582/Screenshot_20230209-170131_Fortnite.jpg")
    embed.set_footer(text=f"Për {CREATOR_NAME} Community", icon_url=interaction.guild.icon.url)
    await interaction.response.send_message(embed=embed)

@tree.command(name="prove-follow", description="Merr rol special nëse ke bërë subscribe/follow.")
async def prove_follow_command(interaction: discord.Interaction):
    await interaction.response.send_message("Për të marrë rolin special, dërgo një screenshot ose video ku tregon që je `Subscriber`/`Follower` në kanalin **<#1411378306519007323>** (Role-selection) dhe do të kontaktoheni nga stafi.", ephemeral=True)

@tree.command(name="clear", description="Fshin një numër të caktuar mesazhesh.")
@discord.app_commands.describe(amount="Sa mesazhe dëshironi të fshini?")
async def clear_command(interaction: discord.Interaction, amount: int):
    moderator_role = discord.utils.get(interaction.guild.roles, name=MODERATOR_ROLE_NAME)
    if not interaction.user.guild_permissions.manage_messages and moderator_role not in interaction.user.roles:
        await interaction.response.send_message("Nuk keni leje për të përdorur këtë komandë.", ephemeral=True)
        return

    if amount > 100:
        await interaction.response.send_message("Mund të fshini maksimumi 100 mesazhe në një kohë.", ephemeral=True)
        return

    await interaction.response.defer(ephemeral=True)
    deleted = await interaction.channel.purge(limit=amount + 1)
    await interaction.followup.send(f"U fshinë **{len(deleted)-1}** mesazhe. ✨", ephemeral=True)

@tree.command(name="kick", description="Përjashton (kick) një anëtar nga serveri.")
@discord.app_commands.describe(member="Anëtari që dëshironi të përjashtoni.", reason="Arsyeja e përjashtimit.")
async def kick_command(interaction: discord.Interaction, member: discord.Member, reason: str = "Asnjë arsye nuk u dha.")
    if not interaction.user.guild_permissions.kick_members:
        await interaction.response.send_message("Nuk keni leje për të përdorur këtë komandë.", ephemeral=True)
        return

    if member.guild_permissions.administrator:
        await interaction.response.send_message("Nuk mund të përjashtoni një administrator!", ephemeral=True)
        return

    if member == interaction.guild.owner:
        await interaction.response.send_message("Nuk mund ta përjashtoni pronarin e serverit!", ephemeral=True)
        return

    await interaction.response.defer(ephemeral=True)
    try:
        await member.kick(reason=reason)
        await interaction.followup.send(f"Anëtari **{member.name}** u përjashtua nga serveri. Arsyeja: **{reason}**", ephemeral=False)
        await member.send(f"Jeni përjashtuar nga serveri **{interaction.guild.name}**. Arsyeja: **{reason}**")
        logging_channel = client.get_channel(LOGGING_CHANNEL_ID)
        if logging_channel:
            await logging_channel.send(f"Anëtari **{member.name}** u përjashtua nga {interaction.user.name}. Arsyeja: **{reason}**")
    except discord.Forbidden:
        await interaction.followup.send("Nuk kam leje për të bërë këtë veprim.", ephemeral=True)
    except Exception as e:
        await interaction.followup.send(f"Ndodhi një gabim: {e}", ephemeral=True)

@tree.command(name="ban", description="Ndëshkon (ban) një anëtar nga serveri.")
@discord.app_commands.describe(member="Anëtari që dëshironi të ndëshkoni.", reason="Arsyeja e ndëshkimit.")
async def ban_command(interaction: discord.Interaction, member: discord.Member, reason: str = "Asnjë arsye nuk u dha.")
    if not interaction.user.guild_permissions.ban_members:
        await interaction.response.send_message("Nuk keni leje për të përdorur këtë komandë.", ephemeral=True)
        return

    if member.guild_permissions.administrator:
        await interaction.response.send_message("Nuk mund të ndëshkoni një administrator!", ephemeral=True)
        return

    if member == interaction.guild.owner:
        await interaction.response.send_message("Nuk mund ta ndëshkoni pronarin e serverit!", ephemeral=True)
        return

    await interaction.response.defer(ephemeral=True)
    try:
        await member.ban(reason=reason)
        await interaction.followup.send(f"Anëtari **{member.name}** u ndëshkua nga serveri. Arsyeja: **{reason}**", ephemeral=False)
        await member.send(f"Jeni ndëshkuar nga serveri **{interaction.guild.name}**. Arsyeja: **{reason}**")
        logging_channel = client.get_channel(LOGGING_CHANNEL_ID)
        if logging_channel:
            await logging_channel.send(f"Anëtari **{member.name}** u ndëshkua nga {interaction.user.name}. Arsyeja: **{reason}**")
    except discord.Forbidden:
        await interaction.followup.send("Nuk kam leje për të bërë këtë veprim.", ephemeral=True)
    except Exception as e:
        await interaction.followup.send(f"Ndodhi një gabim: {e}", ephemeral=True)

@tree.command(name="fortnite-shop", description="Shfaq sendet e ditës në dyqanin e Fortnite.")
async def fortnite_shop_command(interaction: discord.Interaction):
    await interaction.response.defer()
    
    url = FORTNITE_SHOP_API
    try:
        response = requests.get(url, headers={"Authorization": API_KEY_I_FORTNITE_KETU})
        data = response.json()
        
        embed = discord.Embed(
            title="Dyqani i Fortnite Sot",
            description="Këto janë sendet në dyqanin e Fortnite për ditën e sotme.",
            color=discord.Color.red()
        )
        
        for section in data['data']['sections']:
            items_text = ""
            for item in section['items']:
                name = item['displayName']
                price = item['price']['finalPrice']
                items_text += f"**{name}** - {price} V-Bucks\n"
            
            if items_text:
                embed.add_field(name=section['name'], value=items_text, inline=False)
                
        if data['data']['featured']:
            featured_item = data['data']['featured'][0]
            if 'images' in featured_item and 'icon' in featured_item['images']:
                embed.set_image(url=featured_item['images']['icon'])
        
        embed.set_footer(text=f"Data e fundit e përditësimit: {data['data']['date']}")
        await interaction.followup.send(embed=embed)
        
    except Exception as e:
        await interaction.followup.send(f"Ndodhi një gabim gjatë marrjes së të dhënave të dyqanit: {e}")

@tree.command(name="stw-missions", description="Shfaq misionet ditore të Save the World.")
async def stw_missions_command(interaction: discord.Interaction):
    await interaction.response.defer()
    
    url = FORTNITE_STW_API
    try:
        response = requests.get(url, headers={"Authorization": API_KEY_I_FORTNITE_KETU})
        data = response.json()
        
        embed = discord.Embed(
            title="Misionet e Save the World Sot",
            description="Misionet ditore të Save the World.",
            color=discord.Color.red()
        )
        
        missions_data = data['data']
        if not missions_data:
            embed.description = "Nuk ka misione të listuara për sot."
        else:
            for mission in missions_data:
                name = mission['name']
                region = mission['region']
                rewards = ", ".join([reward['displayName'] for reward in mission['rewards']])
                
                embed.add_field(name=f"{name} ({region})", value=f"Shpërblimet: {rewards}", inline=False)
        
        await interaction.followup.send(embed=embed)
        
    except Exception as e:
        await interaction.followup.send(f"Ndodhi një gabim gjatë marrjes së të dhënave të misioneve: {e}")

@tree.command(name="fortnite-stats", description="Shfaq statistikat e lojtarit të Fortnite.")
@discord.app_commands.describe(username="Emri i përdoruesit të Fortnite.")
async def fortnite_stats_command(interaction: discord.Interaction, username: str):
    await interaction.response.defer()
    
    url = f"{FORTNITE_STATS_API}?name={username}"
    try:
        response = requests.get(url, headers={"Authorization": API_KEY_I_FORTNITE_KETU})
        data = response.json()
        
        if data['status'] == 404:
            await interaction.followup.send(f"Lojtari me emrin '{username}' nuk u gjet. Kontrolloni saktësinë e emrit.")
            return

        stats = data['data']['stats']['all']
        
        embed = discord.Embed(
            title=f"Statistikat e Fortnite për {data['data']['account']['name']}",
            color=discord.Color.green()
        )
        
        stats_text = (
            f"**Fitore:** {stats['overall']['wins']}\n"
            f"**Vrasje:** {stats['overall']['kills']}\n"
            f"**KD:** {stats['overall']['kd']}\n"
            f"**Ndeshje të luajtura:** {stats['overall']['matches']}\n"
            f"**Ora e luajtur:** {stats['overall']['minutesPlayed']} minuta"
        )
        embed.add_field(name="Statistika Totale", value=stats_text, inline=False)
        
        await interaction.followup.send(embed=embed)
        
    except Exception as e:
        await interaction.followup.send(f"Ndodhi një gabim gjatë marrjes së statistikave: {e}")

@tree.command(name="trade", description="Krijon një postim tregtimi në kanalin e tregtisë.")
@discord.app_commands.describe(item_offered="Senda që ofroni.", item_wanted="Senda që kërkoni.", description="Detaje shtesë (opsionale).")
async def trade_command(interaction: discord.Interaction, item_offered: str, item_wanted: str, description: str = "Nuk u dha asnjë përshkrim."):
    trade_channel = client.get_channel(TRADE_CHAT_CHANNEL_ID)
    if not trade_channel:
        await interaction.response.send_message("Kanali i tregtimit nuk u gjet. Ju lutem kontaktoni administratorin.", ephemeral=True)
        return

    embed = discord.Embed(
        title="〔🔁〕POSTIM TREGTIMI",
        description=f"**Postuar nga:** {interaction.user.mention}",
        color=discord.Color.blue()
    )
    embed.add_field(name="OFEROJ:", value=item_offered, inline=False)
    embed.add_field(name="KËRKOJ:", value=item_wanted, inline=False)
    embed.add_field(name="Përshkrimi:", value=description, inline=False)
    embed.set_footer(text="Për të tregtuar, kontaktoni përdoruesin direkt!")

    await trade_channel.send(embed=embed)
    await interaction.response.send_message("Postimi juaj i tregtimit u dërgua me sukses!", ephemeral=True)

@tree.command(name="build", description="Posto një krijim në kanalin e ndërtimeve.")
@discord.app_commands.describe(title="Titulli i ndërtimit.", description="Përshkrimi i ndërtimit.")
async def build_command(interaction: discord.Interaction, title: str, description: str):
    build_channel = client.get_channel(BUILD_CHAT_CHANNEL_ID)
    if not build_channel:
        await interaction.response.send_message("Kanali i ndërtimeve nuk u gjet. Ju lutem kontaktoni administratorin.", ephemeral=True)
        return

    if not interaction.message.attachments:
        await interaction.response.send_message("Ju lutem ngarkoni një foto ose video të ndërtimit tuaj me këtë komandë.", ephemeral=True)
        return
    
    attachment = interaction.message.attachments[0]
    
    embed = discord.Embed(
        title=f"〔🕌〕NDËRTIM NGA: {title}",
        description=description,
        color=discord.Color.purple()
    )
    embed.set_author(name=interaction.user.display_name, icon_url=interaction.user.avatar.url)
    embed.set_image(url=attachment.url)
    embed.set_footer(text="Për të shprehur opinionin tuaj, jepni një reagim ose komentoni direkt në mesazh.")

    await build_channel.send(embed=embed)
    await interaction.response.send_message("Krijimi juaj u postua me sukses!", ephemeral=True)

@tree.command(name="staff-say", description="Dërgo një mesazh në kanalin e stafit.")
@discord.app_commands.checks.has_any_role('VENDOS KETU ROLIN E STAFF-IT TEND')
@discord.app_commands.describe(message="Mesazhi që dëshironi të dërgoni.")
async def staff_say_command(interaction: discord.Interaction, message: str):
    staff_channel = client.get_channel(STAFF_CHAT_CHANNEL_ID)
    if not staff_channel:
        await interaction.response.send_message("Kanali i stafit nuk u gjet.", ephemeral=True)
        return
    
    embed = discord.Embed(
        title="Njoftim nga stafi",
        description=message,
        color=discord.Color.gold()
    )
    embed.set_footer(text=f"Mesazh nga: {interaction.user.display_name}", icon_url=interaction.user.avatar.url)
    
    await staff_channel.send(embed=embed)
    await interaction.response.send_message("Mesazhi u dërgua në kanalin e stafit.", ephemeral=True)

@tree.command(name="promote", description="Promovo një video, live-stream, ose rrjet social.")
@discord.app_commands.describe(link="Linku i materialit që po promovoni.", description="Përshkrim i shkurtër i materialit.")
async def promote_command(interaction: discord.Interaction, link: str, description: str):
    user_id = str(interaction.user.id)
    current_time = datetime.now()
    
    cooldown = timedelta(hours=24)
    if user_id in last_promotion_time:
        time_since_last_promo = current_time - last_promotion_time[user_id]
        if time_since_last_promo < cooldown:
            remaining_time = cooldown - time_since_last_promo
            hours = int(remaining_time.total_seconds() // 3600)
            minutes = int((remaining_time.total_seconds() % 3600) // 60)
            await interaction.response.send_message(f"Ju lutemi prisni! Mund të bëni një tjetër promovim pas **{hours} orësh** dhe **{minutes} minutash**.", ephemeral=True)
            return

    promotions_channel = client.get_channel(PROMOTIONS_CHANNEL_ID)
    if not promotions_channel:
        await interaction.response.send_message("Kanali i promocioneve nuk u gjet. Ju lutem kontaktoni administratorin.", ephemeral=True)
        return
        
    embed = discord.Embed(
        title="Promovim i Ri!",
        description=f"**Dërguar nga:** {interaction.user.mention}\n\n"
                      f"**Përshkrimi:**\n{description}\n\n"
                      f"**Linku:** {link}",
        color=discord.Color.red()
    )
    embed.set_thumbnail(url=interaction.user.avatar.url)
    embed.set_footer(text="Promovimi i radhës është i mundur pas 24 orësh.")
    
    await promotions_channel.send(embed=embed)
    last_promotion_time[user_id] = current_time
    await interaction.response.send_message("Promovimi juaj u dërgua me sukses!", ephemeral=True)

#----------------------------------------------------
# KOMANDAT DHE VEÇORITË KRYESORE
#----------------------------------------------------
# Kjo është pika e hyrjes së botit
client.run(MTQxMDc0MzU0OTAxNTg4ODA2NA.GAc08J.WinXMh5HcfwWoTy2jL0Pk4zxDPo1TLv8uYOx00)
