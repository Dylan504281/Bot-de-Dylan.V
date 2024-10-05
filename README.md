import discord
import random

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

client.run('Token')  # Replace with your actual bot token
