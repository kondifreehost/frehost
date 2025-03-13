import os
import discord
from discord.ext import commands
from discord.ui import View, Button
from dotenv import load_dotenv
import keep_alive

keep_alive.keep_alive()

# Załaduj zmienne środowiskowe z pliku .env
load_dotenv()
TOKEN = os.getenv("DISCORD_TOKEN")

if not TOKEN:
    print("❌ BŁĄD: Token nie został załadowany! Sprawdź plik .env.")
    exit()

# Ustawienia bota
intents = discord.Intents.default()
intents.messages = True
intents.guilds = True
intents.members = True
intents.message_content = True  # Potrzebne do obsługi wiadomości

bot = commands.Bot(command_prefix="!", intents=intents)

# Kategorie ticketów i ich emoji
TICKET_CATEGORIES = {"Zakup": "🛒", "Partnerstwo": "🤝", "Pomoc": "🎓"}


class TicketView(View):
    """Widok z przyciskami do tworzenia ticketów."""

    def __init__(self):
        super().__init__(timeout=None)
        for category_name, emoji in TICKET_CATEGORIES.items():
            self.add_item(
                TicketButton(label=category_name,
                             emoji=emoji,
                             custom_id=category_name))


class TicketButton(Button):
    """Przycisk do tworzenia ticketu."""

    def __init__(self, label, emoji, custom_id):
        super().__init__(label=label,
                         emoji=emoji,
                         style=discord.ButtonStyle.primary,
                         custom_id=custom_id)

    async def callback(self, interaction: discord.Interaction):
        guild = interaction.guild
        category_name = self.custom_id
        category = discord.utils.get(guild.categories, name=category_name)

        if category is None:
            category = await guild.create_category(category_name)

        # Sprawdzenie, czy użytkownik ma już ticket
        existing_channel = discord.utils.get(
            guild.text_channels,
            name=f"ticket-{interaction.user.name.lower()}")
        if existing_channel:
            await interaction.response.send_message(
                f"⚠️ Masz już otwarty ticket: {existing_channel.mention}",
                ephemeral=True)
            return

        # Ustawienia uprawnień do ticketu
        overwrites = {
            guild.default_role:
            discord.PermissionOverwrite(view_channel=False),
            interaction.user:
            discord.PermissionOverwrite(view_channel=True, send_messages=True),
            guild.me:
            discord.PermissionOverwrite(view_channel=True)
        }

        # Tworzenie nowego kanału ticketu
        ticket_channel = await guild.create_text_channel(
            name=f"ticket-{interaction.user.name}",
            category=category,
            overwrites=overwrites)

        await ticket_channel.send(
            f"🎟️ {interaction.user.mention}, Twój ticket został utworzony w kategorii **{category_name}**.\n"
            "Proszę czekać na odpowiedź od administratora.",
            view=CloseTicketView())
        await interaction.response.send_message(
            f"✅ Ticket został utworzony: {ticket_channel.mention}",
            ephemeral=True)


class CloseTicketView(View):
    """Widok z przyciskiem do zamykania ticketów."""

    def __init__(self):
        super().__init__(timeout=None)
        self.add_item(CloseTicketButton())


class CloseTicketButton(Button):
    """Przycisk do zamykania ticketu."""

    def __init__(self):
        super().__init__(label="❌ Zamknij Ticket",
                         style=discord.ButtonStyle.danger)

    async def callback(self, interaction: discord.Interaction):
        await interaction.channel.delete()


@bot.event
async def on_ready():
    print(f"✅ Bot zalogowany jako {bot.user}")


@bot.command()
async def ticket(ctx):
    """Wyświetla panel do tworzenia ticketów."""
    embed = discord.Embed(
        title="🎟️ System Ticketów",
        description=
        "Wybierz kategorię ticketa, klikając odpowiedni przycisk poniżej.\n\n"
        "🛒 **Zakup** - Pytania dotyczące zakupów\n"
        "🤝 **Partnerstwo** - Współpraca i partnerstwa\n"
        "🎓 **Pomoc** - Wsparcie techniczne\n\n"
        "🛠️ **Nie otwieraj ticketa bez potrzeby!**",
        color=discord.Color.blue())
    await ctx.send(embed=embed, view=TicketView())


@bot.command()
async def commands(ctx):
    """Wyświetla listę komend bota."""
    commands_list = [command.name for command in bot.commands]
    commands_text = "\n".join(f"**!{cmd}**" for cmd in commands_list)
    embed = discord.Embed(title="📜 Lista Komend",
                          description=commands_text
                          or "Brak dostępnych komend!",
                          color=discord.Color.green())
    await ctx.send(embed=embed)


# Utrzymanie bota online
keep_alive.keep_alive()

bot.run(TOKEN)

keep_alive.keep_alive()
