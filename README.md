import discord
import random
import asyncio
import youtube_dl

from discord.ext import commands

# Intents for message content, reactions, and member updates
intents = discord.Intents.default()
intents.message_content = True
intents.reactions = True
intents.members = True

# Create the Discord client
client = discord.Client(intents=intents)

# Emoticon dictionary for emote commands
emoticon_dict = {
    '$feliz': '',
    '$triste': '☹️',
    '$enojado': '',
    '$sorprendido': '',
    '$pensativo': '',
}

# List of predefined trivia questions and answers
trivia_questions = [
    ("What is the capital of France?", "Paris"),
    ("What is the tallest mountain in the world?", "Mount Everest"),
    ("What is the largest planet in our solar system?", "Jupiter"),
    ("In what year was the first iPhone released?", "2007"),
    ("Who painted the Mona Lisa?", "Leonardo da Vinci"),
]

# Function to generate a random trivia question
def get_trivia_question():
    question, answer = random.choice(trivia_questions)
    return f" Trivia Time! {question}"

# Function to generate a random joke
def get_joke():
    # Replace with your preferred joke API or implementation
    joke = "Here's a placeholder joke for now!"
    return joke

# Function to check if a user has the appropriate role for a command
def has_role(member, role_name):
    for role in member.roles:
        if role.name == role_name:
            return True
    return False

@client.event
async def on_ready():
    print(f'Logged in as {client.user} (ID: {client.user.id})')

@client.event
async def on_message(message):
    if message.author == client.user:
        return

    # Command handling
    if message.content.startswith('$'):
        command = message.content.split()[0].lower()

        if command == '$hello':
            await message.channel.send("Hi there!  What can I do for you today?")

        elif command == '$bye':
            await message.channel.send("See you later! ")

        elif command in emoticon_dict:
            emoticon = emoticon_dict[command]
            await message.channel.send(emoticon)

        elif command == '$moneda':
            result = random.randint(1, 2)
            if result == 1:
                await message.channel.send("Cara")
            else:
                await message.channel.send("Cruz")

        elif command == '$trivia':
            question = get_trivia_question()
            await message.channel.send(question)

        elif command == '$joke':
            # Check if user has the "Joke Teller" role (optional)
            if has_role(message.author, "Joke Teller"):
                joke = get_joke()
                await message.channel.send(joke)
            else:
                await message.channel.send("Sorry, you don't have permission to tell jokes. ")

        else:
            await message.channel.send(f"Sorry, I don't understand the command '{command}'.")

# Function to add reactions to specific messages (optional)
@client.event
async def on_reaction_add(reaction, user):
    if user == client.user:
        return

    message = reaction.message
    if message.author == client.user:
        # Example: React with thumbs up to upvote trivia answers
        if reaction.emoji == '' and message.content.startswith(" Trivia Time!"):
            await message.add_reaction(reaction.emoji)


from discord.ext import commands

bot = commands.Bot(command_prefix='$')  # Definición del bot

@bot.command()
async def joined(ctx, member: discord.Member):
    """Says when a member joined."""
@bot.command()
async def repeat(ctx, times: int, content='repeating...'):
    """Repeats a message multiple times."""
    for i in range(times):
        await ctx.send(content)
@bot.command()
async def add(ctx, left: int, right: int):
    """Adds two numbers together."""
    await ctx.send(left + right)


@bot.command()
async def roll(ctx, dice: str):
    """Rolls a dice in NdN format."""
    try:
        rolls, limit = map(int, dice.split('d'))
    except Exception:
        await ctx.send('Format has to be in NdN!')
        return

    result = ', '.join(str(random.randint(1, limit)) for r in range(rolls))
    await ctx.send(result)


@bot.command(description='For when you wanna settle the score some other way')
async def choose(ctx, *choices: str):
    """Chooses between multiple choices."""
    await ctx.send(random.choice(choices))

@bot.group()
async def cool(ctx):
    """Says if a user is cool.

    In reality this just checks if a subcommand is being invoked.
    """
    if ctx.invoked_subcommand is None:
        await ctx.send(f'No, {ctx.subcommand_passed} is not cool')


@cool.command(name='bot')
async def _bot(ctx):
    """Is the bot cool?"""
    await ctx.send('Yes, the bot is cool.')

# Suppress noise about console usage from errors
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
    'source_address': '0.0.0.0',  # bind to ipv4 since ipv6 addresses cause issues sometimes
}

ffmpeg_options = {
    'options': '-vn',
}

ytdl = youtube_dl.YoutubeDL(ytdl_format_options)


class YTDLSource(discord.PCMVolumeTransformer):
    def _init_(self, source, *, data, volume=0.5):
        super()._init_(source, volume)

        self.data = data

        self.title = data.get('title')
        self.url = data.get('url')

    @classmethod
    async def from_url(cls, url, *, loop=None, stream=False):
        loop = loop or asyncio.get_event_loop()
        data = await loop.run_in_executor(None, lambda: ytdl.extract_info(url, download=not stream))

        if 'entries' in data:
            # take first item from a playlist
            data = data['entries'][0]

        filename = data['url'] if stream else ytdl.prepare_filename(data)
        return cls(discord.FFmpegPCMAudio(filename, **ffmpeg_options), data=data)


class Music(commands.Cog):
    def _init_(self, bot):
        self.bot = bot

    @commands.command()
    async def join(self, ctx, *, channel: discord.VoiceChannel):
        """Joins a voice channel"""

        if ctx.voice_client is not None:
            return await ctx.voice_client.move_to(channel)

        await channel.connect()

    @commands.command()
    async def play(self, ctx, *, query):
        """Plays a file from the local filesystem"""

        source = discord.PCMVolumeTransformer(discord.FFmpegPCMAudio(query))
        ctx.voice_client.play(source, after=lambda e: print(f'Player error: {e}') if e else None)

        await ctx.send(f'Now playing: {query}')

    @commands.command()
    async def yt(self, ctx, *, url):
        """Plays from a url (almost anything youtube_dl supports)"""

        async with ctx.typing():
            player = await YTDLSource.from_url(url, loop=self.bot.loop)
            ctx.voice_client.play(player, after=lambda e: print(f'Player error: {e}') if e else None)

        await ctx.send(f'Now playing: {player.title}')

    @commands.command()
    async def stream(self, ctx, *, url):
        """Streams from a url (same as yt, but doesn't predownload)"""

        async with ctx.typing():
            player = await YTDLSource.from_url(url, loop=self.bot.loop, stream=True)
            ctx.voice_client.play(player, after=lambda e: print(f'Player error: {e}') if e else None)

        await ctx.send(f'Now playing: {player.title}')

    @commands.command()
    async def volume(self, ctx, volume: int):
        """Changes the player's volume"""

        if ctx.voice_client is None:
            return await ctx.send("Not connected to a voice channel.")

        ctx.voice_client.source.volume = volume / 100
        await ctx.send(f"Changed volume to {volume}%")

    @commands.command()
    async def stop(self, ctx):
        """Stops and disconnects the bot from voice"""

        await ctx.voice_client.disconnect()

    @play.before_invoke
    @yt.before_invoke
    @stream.before_invoke
    async def ensure_voice(self, ctx):
        if ctx.voice_client is None:
            if ctx.author.voice:
                await ctx.author.voice.channel.connect()
            else:
                await ctx.send("You are not connected to a voice channel.")
                raise commands.CommandError("Author not connected to a voice channel.")
        elif ctx.voice_client.is_playing():
            ctx.voice_client.stop()


intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(
    command_prefix=commands.when_mentioned_or("!"),
    description='Relatively simple music bot example',
    intents=intents,
)


@bot.event
async def on_ready():
    print(f'Logged in as {bot.user} (ID: {bot.user.id})')
    print('------')


async def main():
    async with bot:
       await bot.add_cog(Music(bot))
    await bot.start('Token')


asyncio.run(main())   

client.run("Token")
